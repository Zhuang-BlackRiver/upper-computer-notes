# W2 · 完整项目分层（Wu.CommTool 精读）

**时间预算**：2 天 × 2 小时
**仓库**：`Monika1313/Wu.CommTool`（WPF + Prism，15 个协议模块，目标框架 `net6.0-windows;net7.0-windows;net48`）
**本周要回答的问题**：一个上位机程序里塞进 Modbus/MQTT/TCP/串口 5 类协议、15 个功能页，怎么做到"加一个协议不改主界面一行代码"？

> 为什么选它而不选 SharpSCADA：SharpSCADA 是 2017 年的单人项目，架构上只有一个大 `MainWindow` 和一堆全局静态类，能看协议实现，学不到分层。Wu.CommTool 是这几年组织得最干净的中文上位机开源项目，而且**它的技术栈和你的日常完全重合**（见下）。

---

## 0. 先看它的技术栈，判断值不值得学

| 依赖 | 出现次数 | 说明 |
|---|---|---|
| `Prism.Wpf` 8.1.97 | 15（每个模块一个） | 模块化 + 区域导航 + MVVM 基础设施 |
| `Prism.DryIoc` | 2 | DI 容器选型 |
| `MQTTnet` | 2 | MQTT 客户端/服务端模块 |
| `log4net` | 1 | 日志 |
| `Newtonsoft.Json` | 1 | 配置序列化 |
| `WindowsFirewallHelper` | 1 | **细节**：自动加防火墙规则，避免"服务端监听了但连不上" |
| `System.Management` | 1 | WMI，查网卡/进程 |

`net48` 在目标框架里 —— **意味着这套分层可以直接搬进你现有的 .NET Framework 4.8 项目**，不用先升级到 .NET 8。这一点比学 Avalonia/.NET 8 的项目对你更实用。

`WindowsFirewallHelper` 这个依赖值得单独说：从站/服务端功能一开就监听端口，客户现场 Windows 防火墙默认拦截，程序员调试时是关防火墙的，交付到现场就"连不上"。**一个 20 行的防火墙规则，能省掉一次出差。** 这种细节只有真交付过的人才会写进工具。

---

## 1. 三层目录：宿主 / 内核 / 模块

📄 [PROJECT_STRUCTURE.md](https://github.com/Monika1313/Wu.CommTool/blob/master/PROJECT_STRUCTURE.md)

```
Wu.CommTool/                    ← 宿主：壳（主窗口、菜单、主题、自动更新）
Wu.CommTool.Core/               ← 内核：枚举、扩展、通用工具、模型（不含界面）
Wu.CommTool.DynamicTheme.*/     ← 主题资源（可换肤）
Modules/                        ← 15 个功能模块，每个一个独立 csproj
  Wu.CommTool.Modules.ModbusTcp/
  Wu.CommTool.Modules.ModbusRtu/
  Wu.CommTool.Modules.ModbusAscii/
  Wu.CommTool.Modules.MqttClient/
  Wu.CommTool.Modules.MqttServer/
  Wu.CommTool.Modules.TcpClient/  TcpServer/  Udp/
  Wu.CommTool.Modules.MrtuSlave/  NetworkTool/
  Wu.CommTool.Modules.JsonTool/  CryptoTool/  ConvertTools/  About/  Message/
```

**这个划分的判据很清晰**：

- 只有**枚举、数据模型、纯函数工具**才进 Core。Core 不引用 Prism，不引用任何 `System.Windows`。
- **每个协议模块独立成 csproj**，只依赖 Core + Prism。模块之间**零引用**（ModbusTcp 不知道 MqttClient 存在）。
- 宿主只负责"把模块拼起来 + 画壳"。

> **对照自己**：你的上位机如果把 `ModbusHelper`、`MqttHelper`、`MainForm` 放同一个 csproj，那编译一次要全量重编，改一个协议可能碰坏另一个——这就是"能跑但不能长"的信号。

## 2. 模块内部的固定五件套

每个模块都长一样（以 ModbusTcp 为例）：

```
Modules/Wu.CommTool.Modules.ModbusTcp/
├── ModbusTcpModule.cs           ← Prism IModule：注册页面与导航
├── Models/                      ← 协议模型 + 通讯逻辑（★ 没有界面代码）
│   ├── MtcpFrame.cs             帧结构（MBAP + PDU）
│   ├── MtcpDevice.cs            设备实体 + 轮询循环 + 状态机（683 行）
│   ├── MtcpMaster.cs            主站逻辑（365 行）
│   ├── MtcpSlave.cs             从站模拟器（1530 行，支持 0x01/0x03/0x04/0x05/0x06/0x0F/0x10）
│   └── MtcpDeviceManager.cs     多设备管理
├── ViewModels/                  ← 页面状态 + 命令
│   ├── ModbusTcpViewModel.cs          菜单/导航中转（148 行）
│   ├── MtcpDeviceMonitorViewModel.cs  设备监控（414 行）
│   └── DialogViewModels/MtcpLogViewModel.cs
├── Views/                       ← 只有 XAML + 极薄 code-behind
├── Enums/                       ← 功能码、帧类型、消息类型
└── Events/                      ← 弱引用事件
```

📄 [ModbusTcpModule.cs#L9-L24](https://github.com/Monika1313/Wu.CommTool/blob/master/Modules/Wu.CommTool.Modules.ModbusTcp/ModbusTcpModule.cs#L9-L24)

```csharp
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<ModbusTcpView, ModbusTcpViewModel>();          // 模块主界面
    containerRegistry.RegisterForNavigation<ModbusTcpCustomFrameView, ModbusTcpCustomFrameViewModel>(); // 自定义帧
    containerRegistry.RegisterForNavigation<ModbusTcpMasterView, ModbusTcpMasterViewModel>();          // 主站
    containerRegistry.RegisterForNavigation<ModbusTcpSlaveView, ModbusTcpSlaveViewModel>();            // 从站
    containerRegistry.RegisterForNavigation<AnalyzeMtcpFrameView, AnalyzeMtcpFrameViewModel>();        // 解析帧弹窗
    containerRegistry.RegisterForNavigation<MtcpDeviceMonitorView>();                                  // 设备监控
    containerRegistry.RegisterForNavigation<MtcpDeviceEditView, MtcpDeviceEditViewModel>();            // 设备编辑
    containerRegistry.RegisterForNavigation<MtcpLogView, MtcpLogViewModel>();                           // 日志
}
```

**这 8 行是整个项目最值得抄的结构。** 每个功能页 = 一对 View/ViewModel + 一行注册。菜单里写 `NavigationParameters` 里的 region 名就能跳过去，**主窗口不需要 `using` 任何模块命名空间**。

如果你现在的做法是 `new Form2().Show()`、主窗体里 20 个 `if (menuItem == ...)`，那这就是升级点：把"页面"变成"导航项"，主壳只认名字不认类型。

## 3. 设备实体：状态机 + 轮询任务放在 Model 里

📄 [MtcpDevice.cs#L9-L37](https://github.com/Monika1313/Wu.CommTool/blob/master/Modules/Wu.CommTool.Modules.ModbusTcp/Models/MtcpDevice.cs#L9-L37)

```csharp
public partial class MtcpDevice : ObservableObject, IDisposable
{
    [ObservableProperty][property: JsonIgnore] DeviceState deviceState;
    // ...
}
```

📄 [MtcpDevice.cs#L537-L552](https://github.com/Monika1313/Wu.CommTool/blob/master/Modules/Wu.CommTool.Modules.ModbusTcp/Models/MtcpDevice.cs#L537-L552)

```csharp
public void UpdateState()
{
    var onlineCount = MtcpDatas.Where(x => x.State == true).Count();
    if (在线数 == 总数) DeviceState = DeviceState.Online;      // 全部在线
    else if (...== 0)   DeviceState = DeviceState.Offline;     // 全部离线
    else                DeviceState = DeviceState.Warning;     // ★ 存在离线的测点
}
```

**`Warning` 这个状态是本项目最有现场味的设计。** 二元"在线/离线"不够用：一个设备 50 个测点，48 个读到、2 个读不到，是"离线"吗？不是，是**部分测点异常**——这才是需要人去看的状态。你现在的设备状态是不是只有一个 `bool Connected`？那就是把三种故障压成了一种。

📄 [MtcpDevice.cs#L255-L268](https://github.com/Monika1313/Wu.CommTool/blob/master/Modules/Wu.CommTool.Modules.ModbusTcp/Models/MtcpDevice.cs#L255-L268)

```csharp
public void RunMonitorTask()
{
    while (Owner.State)          // 循环条件挂在 Owner（管理器）上
    {
        // 遍历测点 → 组帧 → 发送 → 解析 → 回填 State → UpdateState()
    }
}
```

**注意两件事**：

1. 轮询是**每设备一个长循环**，不是全局一个大定时器。好处：一个设备慢不拖累其他设备；坏处：100 个设备 = 100 个线程。商业项目里这里应该是一条轮询队列 + 固定线程池，**这是你可以提出的改进点**。
2. 循环条件 `Owner.State` 而不是 `while(true)` + `Thread.Abort`——用外部标志位优雅退出，配合 `CancellationToken` 是标准做法。`IDisposable` 也实现了（L679），说明作者知道资源要还。

📄 [MtcpDevice.cs#L368](https://github.com/Monika1313/Wu.CommTool/blob/master/Modules/Wu.CommTool.Modules.ModbusTcp/Models/MtcpDevice.cs#L368)
`AnalyzeDataAddress()` —— 把用户填的 `40001`、`4x00001`、`400001` 这些**各家地址写法**统一解析成内部偏移。这是上位机必须做的一件事：甲方给的地址表格式永远不统一，解析器要能容忍。

## 4. 主站和从站放在一起：一个被低估的设计

`MtcpMaster.cs`（365 行）和 `MtcpSlave.cs`（1530 行）在同一个模块里。从站支持 7 个功能码 + 4 类异常码 + 4 种寄存器区维护。

**为什么这很重要**：没有从站模拟器，你调通讯只能等现场、等 PLC、等甲方。有了它，**在工位上就能把"读超时、非法地址、异常码 0x02"这些分支全测一遍**。

> 这是你简历上能写的差异化能力：**"能自己造对端来测自己的客户端"**。绝大多数上位机开发者只会写客户端，出问题只能到现场抓包。

---

## 5. 拿它当"反面教材"读的地方

读完要能说出它哪里不好，不然只是抄：

| 现象 | 问题 | 你的改法 |
|---|---|---|
| `MtcpSlave.cs` 1530 行单文件 | 功能码处理、寄存器维护、报文解析、日志全在一个类 | 按功能码拆 `IFunctionHandler`，一个 handler 一个类 |
| 每设备一个轮询线程 | 设备多了线程爆炸、上下文切换开销 | 单轮询调度器 + `Channel<T>` 队列 |
| `ObservableObject` 直接在 Model 上 | 通讯模型绑了 CommunityToolkit.Mvvm，Core 想复用就得带 WPF 依赖 | 通讯层用纯 POCO + 事件，VM 层再包一层 |
| 日志走 `ShowMessage` 攒到界面 | 界面日志和文件日志两套 | `ILogger` 抽象 + 双 Sink |
| 大量 `x:Name` + code-behind | 部分页面还是事件驱动 | 全 Command 化，code-behind 只留窗口行为 |

**读开源项目的正确姿势：一半学它做对的，一半记它做错的。** 后者才是你面试时"你和别人不一样"的地方。

---

## 自测题

1. 为什么会把 `DeviceState` 设计成 `Online/Offline/Warning` 三态，而不是 `bool Connected`？
2. `RegisterForNavigation` 相比 `new Window().Show()` 解决了什么问题？举一个具体场景。
3. 每设备一个轮询线程，在 200 台设备的项目里会出什么事？怎么改？
4. 为什么 `MtcpSlave` 的存在能显著缩短调试周期？
5. Core 层为什么不引用 Prism？

<details>
<summary>参考答案</summary>

1. 故障粒度不同：整台离线（通讯断）和部分测点异常（地址错/类型错/被别的程序占用）处理方式完全不同，后者换线、重启都没用，要改配置。一个 bool 会把两类问题混成"设备坏了"，现场排查方向直接跑偏。
2. 解耦壳与页面：宿主不需要 `using` 模块命名空间、不需要知道 View 类型，菜单只传 region 名。场景：把 ModbusTcp 模块整个删掉，主程序仍能编译运行，只是少一个菜单项——这就是"模块"的意义。
3. 200 个线程同时抢 socket/串口，CPU 大量时间在上下文切换；串口还被 OS 串行化，线程越多排队越乱；某个设备超时会拖慢自己的循环而不是别人的（这是好事），但线程栈内存 200×1MB = 200MB 白吃。改法：单调度器 + 优先级队列 + 每设备独立超时。
4. 不必等现场 PLC：可以在本机模拟异常码、超时、非法地址、断线重连，把客户端的错误分支全跑到。很多 bug 只在"PLC 回了异常帧"时出现，没有从站就永远测不到。
5. Core 要保持可被 WinForms/控制台/服务端复用。引用 Prism 就等于绑死 WPF，自研工具库将来想做成 .NET Standard 库给别的项目用就废了。
</details>

---

## 本周产出物

1. **画出你自己项目的"模块边界图"**：标出哪些类跨了协议、哪些界面类里藏着通讯代码。
2. **写出你的模块模板**（目录 + 5 个文件的空壳），要求：加一个"西门子 S7"模块只新增文件、不改宿主。
3. 把"三态设备状态"和"从站模拟器"两条写进 [../docs/interview-cards.md](../docs/interview-cards.md)。

## 一句话带走

> 分层的验收标准不是"目录好看"，而是**删掉一个模块，主程序还能编译运行**。
