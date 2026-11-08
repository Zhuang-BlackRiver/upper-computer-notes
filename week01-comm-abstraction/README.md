# W1 · 通讯层抽象（IoTClient 精读）

**时间预算**：2 天 × 2 小时
**仓库**：`zhaopeiym/IoTClient`（MIT，~11.6k 行 C#，.NET Standard 2.0）
**本周要回答的问题**：你现有的设备通讯层，是不是"每接一个新设备就复制一个类，改 8 个地方"？

---

## 0. 先看地图（10 分钟）

```
IoTClient/
├── Result.cs                    ← 统一返回值：成败 + 报文 + 耗时（191 行）
├── SocketBase.cs                ← TCP 连接生命周期 + 收包循环 + 重试（195 行）
├── SerialPortBase.cs            ← 串口同一套抽象
├── Interfaces/
│   ├── IIoTClient.cs            ← 面向业务的统一接口（251 行，读/写/批量）
│   └── IEthernetClient.cs
├── Clients/
│   ├── Modbus/
│   │   ├── ModbusTcpClient.cs   ← 1557 行，本站最大单文件
│   │   ├── ModbusRtuClient.cs / ModbusAsciiClient.cs / ModbusRtuOverTcpClient.cs
│   │   ├── Base/ModbusSerialBase.cs
│   │   └── Interfaces/IModbusClient.cs   ← 355 行，功能码级接口
│   └── PLC/
│       ├── SiemensClient.cs     ← 1777 行（最大）
│       ├── MitsubishiClient.cs  ← 1368 行
│       ├── OmronFinsClient.cs   ← 1173 行
│       └── AllenBradleyClient.cs
└── Extensions.Adapter/
    ├── IoTClientFactory.cs      ← 简单工厂：设备类型 → 统一接口
    ├── IIoTClientCommon.cs      ← 适配器接口（业务层只认它）
    └── Communication/*.cs       ← 7 个适配器，把各家差异磨平
```

**读法**：不要按目录顺序读。按 `Result → SocketBase → IIoTClient → ModbusTcpClient.BatchRead → Factory` 这条链读，这是"一条数据从业务层到字节流"的真实路径。

---

## 1. Result：上位机不该把异常抛给界面

📄 [IoTClient/Result.cs#L9-L119](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/Result.cs#L9-L119)

```csharp
public class Result
{
    public bool IsSucceed { get; set; } = true;
    public string Err { get; set; }              // setter 里自动 AddErr2List()
    public int ErrCode { get; set; }             // 408 = 连接失败
    public Exception Exception { get; set; }
    public List<string> ErrList { get; private set; }   // 累积错误链
    public string Requst { get; set; }            // 请求报文（原文 hex）
    public string Response { get; set; }          // 响应报文
    public double? TimeConsuming { get; private set; }  // 耗时 ms
    internal Result EndTime() { ... }             // 每个返回点都调
}
```

**这一版设计里最值钱的是三个字段**，不是 `IsSucceed`：

| 字段 | 现场价值 |
|---|---|
| `Requst` / `Response` | 客户说"读不到数"，你把这两行 hex 发给 PLC 厂家，对面 5 分钟就能判责。没有这个字段，只能到现场抓包 |
| `TimeConsuming` | 判断"是网络慢还是 PLC 慢"的唯一客观依据。200 次采样取 P95，比 ping 有用 |
| `ErrList` | 断线重连时错误会连环发生，`ErrList` 保留完整链条；只留最后一条会丢失根因 |

`Result<T> : Result` 再加 `Value`，且**复制父类状态时不重算时间**（`Assignment()` 里 `base.EndTime()`），保证耗时统计的是整条调用链而不是最后一段。

> **对照自己**：你的 `ReadAsync` 现在是不是 `try/catch` + 返回 `null`？界面层是不是在判 `null`？那就是异常语义泄漏到了 UI。

---

## 2. SocketBase：把"收包"这件小事做对

📄 [IoTClient/SocketBase.cs#L18-L139](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/SocketBase.cs#L18-L139)

三个细节，都是踩过坑才写得出来的：

**(a) 固定 4096 缓冲、循环 `Receive` 直到收满**

```csharp
protected const int BufferSize = 4096;                       // L29
int receiveLength = (receiveCount - receiveFinish) >= BufferSize
    ? BufferSize : (receiveCount - receiveFinish);           // L110
var readLeng = socket.Receive(receiveBytes, receiveFinish, receiveLength, SocketFlags.None);
if (readLeng == 0) { socket?.SafeClose(); /* 连接被断开 */ }  // L114-121
```

TCP 是流，`Receive` 返回的字节数**永远不等于你要的字节数**。这里用"剩余长度取 min(4096, 剩余)"循环收满；收到 0 字节不是"没数据"，是**对端关闭**，必须立刻关 socket 并标记失败。这一条能消灭 80% 的"偶发读错位"。

**(b) 断线自动重连一次，而不是无限重试**

📄 [SocketBase.cs#L154-L193](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/SocketBase.cs#L154-L193)

```csharp
var result = SendPackageSingle(command);
if (!result.IsSucceed)
{
    WarningLog?.Invoke(result.Err, result.Exception);   // 日志走委托，不依赖具体日志框架
    var conentResult = Connect();                        // 重连
    if (!conentResult.IsSucceed) return new Result<byte[]>(conentResult);
    return SendPackageSingle(command);                   // 只重试一次
}
```

注意 `WarningLog` 是个 `delegate`（L13），库不绑定 NLog/Serilog——**你决定日志往哪写**。这是库设计该有的克制。代码里还留着 `// TODO 重试机制应改成用户主动设置`，作者自己知道这是个妥协。

**(c) 只有最小单元加锁**

📄 [ModbusTcpClient.cs#L101-L131](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/Clients/Modbus/ModbusTcpClient.cs#L101-L131)

```csharp
public override Result<byte[]> SendPackageSingle(byte[] command)
{
    //从发送命令到读取响应为最小单元，避免多线程执行串数据（可线程安全执行）
    lock (this)
    {
        socket.Send(command);
        var socketReadResul = SocketRead(socket, 8);                  // 先读 MBAP 头
        int length = headPackage[4] * 256 + headPackage[5] - 2;       // L114 长度字段
        socketReadResul = SocketRead(socket, length);                  // 再读 PDU
        ...
    }
}
```

**锁的范围是"发一帧 + 收一帧"**，不是整个轮询方法。所以多个线程可以并发调用 `ReadFloat`，串行化只发生在真正碰 socket 的那几毫秒。MBAP 头里 `[4][5]` 是后续长度——先读 8 字节头再按长度读体，这是所有"长度前缀"协议的通用套路（S7、FINS、MC 也都这么干）。

> 注意 `lock(this)` 是个已知的味道（外部能锁同一个对象），生产代码建议 `private readonly object _sync = new object();`——这是你可以对面试官说的改进点。

---

## 3. IIoTClient：接口是给业务层看的，不是给协议看的

📄 [IoTClient/Interfaces/IIoTClient.cs#L9-L251](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/Interfaces/IIoTClient.cs#L9-L251)

```csharp
public interface IIoTClient
{
    bool Connected { get; }
    LoggerDelegate WarningLog { get; set; }
    Result Open();
    Result Close();
    Result<byte[]> SendPackageSingle(byte[] command);

    // ★ 关键：批量读，业务层只需要给"地址 → 类型"字典
    Result<Dictionary<string, object>> BatchRead(Dictionary<string, DataTypeEnum> addresses, int batchNumber);
    Result BatchWrite(Dictionary<string, object> addresses, int batchNumber);

    Result<bool>    ReadBoolean(string address);
    Result<ushort>  ReadUInt16(string address);
    Result<float>   ReadFloat(string address);
    // ... 还有 10 个重载
    Result Write(string address, float value);
}
```

**设计取舍的真实代价**：`ReadBoolean/ReadFloat/...` 这一串重载，本质是用"类型爆炸"换"调用处不用转型"。业务代码写 `var r = client.ReadFloat("40001")` 比 `client.Read("40001", DataType.Float)` 好读，但接口有 30 个方法，每加一种 PLC 都得实现一遍。

更干净的做法是**只留 `BatchRead` + 单个泛型 `Read<T>`**，把类型分发收到内部。这里没这么做，是因为它的主要用户是"工具/调试软件"（见 README 里一堆商用项目），重载更好用。**这是产品定位决定接口形状的典型例子**——面试可以拿来说。

---

## 4. BatchRead：本周最重要的一段代码

📄 [ModbusTcpClient.cs#L1015-L1121](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/Clients/Modbus/ModbusTcpClient.cs#L1015-L1121)
📄 逐行拆解见 → [DEEP-DIVE-batchread.md](./DEEP-DIVE-batchread.md)

一句话总结：**先把地址按"功能码 + 站号"分组，再把每个组内相邻地址合并成一个连续区间，用一次 `Read` 拿回来，最后按类型从字节里切值。**

```csharp
int readLength = 121;   // 125 - 4 = 121   ← Modbus 单次最多 125 个寄存器
```

那 4 个是留给 32 位/64 位类型跨界的余量。`121` 这个魔数背后是协议上限，不是随手写的。

**收益量化**（你自己算一遍，很有说服力）：
- 200 个测点、往返 30ms、逐点读 → 6 秒/轮
- 地址合并成 3 个区间 → 90ms/轮，**快 60 倍**

> 这就是"上位机性能瓶颈在往返不在算"的具体证据。你现在的轮询是不是 `foreach (var p in points) Read(p)`？如果是，这就是你下一个项目的最大优化点。

---

## 5. Factory + Adapter：屏蔽差异的那一层

📄 [IoTClientFactory.cs#L12-L68](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient.Extensions.Adapter/IoTClientFactory.cs#L12-L68)

```csharp
public static IIoTClientCommon CreateClient2EthernetDevice(
    EthernetDeviceVersion deviceVersion, string ip, int port, int timeout = 1500, ...)
{
    switch (deviceVersion)
    {
        case EthernetDeviceVersion.ModbusTcp:  iotClientCommon = new ModbusTcpCommunication(...); break;
        case EthernetDeviceVersion.Siemens_S7_1200: iotClientCommon = new SiemensClientCommunication(SiemensVersion.S7_1200, ...); break;
        // ... 12 个 case
        default: throw new Exception($"类型[{deviceVersion}]暂未实现");
    }
}
```

**它解决的问题**：`SiemensClient` 要 `SiemensVersion`，`MitsubishiClient` 要 `MitsubishiVersion`，构造签名全不一样。业务层不该知道这些。`IIoTClientCommon` 把差异吃掉，返回统一接口。

**它的技术债**（你的改进点，写在这里就是面试素材）：

| 问题 | 表现 | 更好的做法 |
|---|---|---|
| switch 膨胀 | 加一个品牌要改工厂 + 加枚举 + 加适配器，3 处 | 注册表 `Dictionary<EthernetDeviceVersion, Func<Config, IIoTClientCommon>>`，每个驱动自己注册 |
| 编译期绑定 | 想做成插件必须重新编译主程序 | 反射扫描 `[DeviceDriver]` 特性的程序集，或 MEF/AssemblyLoadContext |
| 参数过宽 | `timeout`、`format` 对串口无意义 | 用 `DeviceConfig` 对象，各驱动只读自己关心的字段 |

**这正是 W4 要补的洞**：从"简单工厂"走到"驱动注册表 + 插件发现"。

---

## 自测题（答不上就回到对应行号）

1. `SocketBase.SocketRead` 里 `readLeng == 0` 为什么必须当成错误处理，而不是"继续等"？
2. `lock (this)` 锁住的范围为什么是"发一帧 + 收一帧"，而不是整个 `BatchRead`？
3. 如果一次要读 200 个地址，`BatchRead` 会发几次请求？`121` 这个数字从哪来？
4. `Result<T>` 的构造函数为什么要复制父类状态而不是重新 new？
5. 用一句话说出 `IIoTClientCommon` 存在的理由。如果去掉它，业务层要改几处？

<details>
<summary>参考答案（先自己想）</summary>

1. TCP 收到 0 字节 = 对端 FIN，连接已断；继续等只会卡到超时，且后续所有读写都会错位。必须关 socket 并让上层重连。
2. 只有"发+收"是原子操作，中间插进别的线程的报文就会串数据（报文错位）。BatchRead 的地址合并是纯计算，不需要锁，锁大了会让并发轮询退化成串行。
3. 取决于地址分布：先按功能码/站号分组，再按 121 个寄存器一档切区间，理想情况（地址连续）1~2 次；最坏每个地址一个区间。121 = 125（协议上限）− 4（32/64 位类型跨界余量）。
4. 因为 `Result<T>` 常常由 `Result` 转换而来（如失败时 `new Result<byte[]>(conntResult)`），要保留原始请求报文、耗时、错误链，否则排障信息丢失。
5. 把各品牌 PLC 构造参数（SiemensVersion / MitsubishiVersion / 串口参数）的差异收敛在一个接口后面，业务层面向 `IIoTClientCommon` 编程；去掉后每处 new 设备的地方都要知道品牌细节，接一个新品牌要改 N 个调用点。
</details>

---

## 本周产出物

1. **`IDeviceDriver` 抽象草案**（20 行以内）：只保留 `Connect/Disconnect/ReadBatch/Write/State`，对照 `IIoTClient` 砍掉 25 个重载，看自己的业务到底需要几个方法。
2. **一个地址合并算法**：输入 `List<(int addr, DataType type)>`，输出 `List<(int start, int len)>`。要求：单区间 ≤ 121 寄存器，32/64 位类型不跨区间截断。
3. **把这两条写进 [../docs/interview-cards.md](../docs/interview-cards.md)**，格式见该文件。

## 一句话带走

> 通讯层的抽象水平，不看接口有多少方法，而看**加一个新设备品牌要改几个文件**、**一次轮询要发几帧**。
