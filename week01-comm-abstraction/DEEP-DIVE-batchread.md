# 深挖：Modbus 批量读的地址合并算法

📄 原文：[ModbusTcpClient.cs#L1015-L1121](https://github.com/zhaopeiym/IoTClient/blob/master/IoTClient/Clients/Modbus/ModbusTcpClient.cs#L1015-L1121)

这是 IoTClient 里最值得抄进自己项目的一段逻辑。分两层：

## 第一层：按 (功能码, 站号) 分组

```csharp
private Result<List<ModbusOutput>> BatchRead(List<ModbusInput> addresses)
{
    var functionCodes = addresses.Select(t => t.FunctionCode).Distinct();       // L1019
    foreach (var functionCode in functionCodes)
    {
        var stationNumbers = addresses.Where(t => t.FunctionCode == functionCode)
                                      .Select(t => t.StationNumber).Distinct();
        foreach (var stationNumber in stationNumbers)
        {
            var addressList = addresses
                .Where(t => t.FunctionCode == functionCode && t.StationNumber == stationNumber)
                .DistinctBy(t => t.Address)                                      // L1026 去重
                .ToDictionary(t => t.Address, t => t.DataType);
            var tempResult = BatchRead(addressList, stationNumber, functionCode); // 第二层
            ...
            // L1044-1049：任一子批次失败 → 立即整体返回，不再往下读
            else { result.SetErrInfo(tempResult); return result.EndTime(); }
        }
    }
}
```

**为什么必须先分组**：功能码不同（0x01 读线圈 / 0x03 读保持寄存器 / 0x04 读输入寄存器）对应的是 PLC 里**不同的地址空间**，字节序和长度规则都不同，绝不能混在一帧里。

**为什么失败要短路**（L1047 注释写得很好）：

> 如果存在异常，按正确逻辑应该是直接返回失败，不继续往下读取了。因为可能存在通信异常，如果继续往下读，可能会导致更多的异常日志产生，甚至造成系统卡顿等问题。

这是现场经验：网络断了的时候，如果继续把剩下 50 个批次逐个超时，界面会卡死、日志会刷屏，还会把你真正的故障掩盖掉。**快速失败比"尽量多读几个"更重要。**

## 第二层：区间合并

```csharp
var minAddress = addresses.Select(t => t.Key).Min();
var maxAddress = addresses.Select(t => t.Key).Max();

while (maxAddress >= minAddress)                       // L1086
{
    int readLength = 121;                              // 125 - 4，协议上限留余量
    var tempAddress = addresses.Where(t => t.Key >= minAddress
                                        && t.Key <= minAddress + readLength).ToList();

    var tempMax = tempAddress.OrderByDescending(t => t.Key).FirstOrDefault();
    switch (tempMax.Value)                             // L1099 按类型定右边界
    {
        case DataTypeEnum.Bool:
        case DataTypeEnum.Byte:
        case DataTypeEnum.Int16:
        case DataTypeEnum.UInt16:
            readLength = tempMax.Key + 1 - minAddress; break;    // 16 位占 1 个寄存器
        case DataTypeEnum.Int32:
        case DataTypeEnum.UInt32:
        case DataTypeEnum.Float:
            readLength = tempMax.Key + 2 - minAddress; break;    // 32 位占 2 个
        case DataTypeEnum.Int64:
        case DataTypeEnum.UInt64:
        case DataTypeEnum.Double:
            readLength = tempMax.Key + 4 - minAddress; break;    // 64 位占 4 个
    }

    var tempResult = Read(minAddress.ToString(), stationNumber, functionCode,
                          Convert.ToUInt16(readLength), byteFormatting: false);   // L1121 一次读整段
    if (!tempResult.IsSucceed) return ...;             // 失败短路

    var rValue = tempResult.Value.Reverse().ToArray();  // L1134 字节序反转（小端 → 大端）
    foreach (var item in tempAddress) { /* 按类型从 rValue 中切出对应偏移的值 */ }
    minAddress += readLength;                          // 推进到下一个未覆盖区间
}
```

### 三个必须注意的细节

1. **`121 = 125 - 4`**：Modbus 单次读寄存器上限 125；减 4 是为了让 64 位类型（占 4 个寄存器）能完整落在区间内不被截断。如果按 125 切，恰好跨界的 `double` 会被切成两半。
2. **`byteFormatting: false`**：先拿原始字节，最后再统一做大小端转换。如果让 `Read` 每段自己做转换，跨段拼装的 32/64 位值会错。这是"把格式化推迟到最后一刻"的经典手法。
3. **`minAddress += readLength`**：不是 `+1`。中间那些没人关心的地址会被**一并读回来然后丢弃**——用带宽换往返次数，局域网里永远划算。

### 一个真实的权衡

如果两个测点相隔 500 个寄存器，合并意味着要多读 500 个寄存器（1000 字节），但省掉一次往返（30ms）。
**判据：多读的字节数 × 0.1ms/KB 与 一次往返 30ms 比较。** 也就是说，只要"空隙 < 约 200 个寄存器"就值得合并。这个阈值叫 *gap threshold*，很多商业 SCADA 驱动里是可配参数。

---

## 你的练手题

写一个不依赖 IoTClient 的纯算法实现：

```csharp
public record PointRef(int Address, DataType Type);

public static List<(int Start, int Count)> BuildReadRanges(
    IEnumerable<PointRef> points,
    int maxRegistersPerRequest = 121,
    int maxGapRegisters = 32)      // 空隙超过这个值就不合并，宁可多发一帧
{
    // 1. 按 Address 升序
    // 2. 按类型算每个点占几个寄存器：16 位→1，32 位→2，64 位→4
    // 3. 贪心合并：当前区间尾部 + gap <= 下个点起点 + maxGap 且总长 <= maxRegistersPerRequest 就并入
    // 4. 输出区间列表
}
```

**验收标准**：
- 输入 200 个连续地址 → 输出 2 个区间（121 + 79）
- 输入 2 个相隔 500 的地址 → 输出 2 个区间（不合并）
- 输入含一个 `double` 落在 120 号地址 → 它必须完整落在同一区间，不能被切开
- 空输入 → 空列表，不抛异常

写完对比一下 IoTClient 的做法：它是"按 minAddress 滑动窗口"，你是"贪心合并"。两者结果接近，但你的实现能表达 `maxGapRegisters` 这个业务参数——**这就是"读懂之后能改得更好"的证据。**
