# W4 · 驱动插件化与收口

**时间预算**：2 天 × 2 小时
**参考项目**：[RapidScada/scada-v6](https://github.com/RapidScada/scada-v6)（成熟商用 SCADA 的开源版）
**本周要回答的问题**：接第 7 个品牌 PLC 时，你是复制粘贴第 6 个类，还是只新增一个文件？

---

## 1. 一个判据，量出你的架构水平

> **新增一个设备品牌，需要改动的文件数 = 你的架构水平。**

| 改 1 个文件 | 改 2~3 个文件 | 改 5 个以上 | 要改主程序并重新编译 |
|---|---|---|---|
| 插件化（理想） | 注册表式 | 简单工厂（IoTClient 现状） | 硬编码 `switch` / `if` |

IoTClient 落在第三档：加品牌要动 **枚举 + 工厂 switch + 新增适配器**。这不是错，因为它的定位是"库"而不是"平台"——但你要做的是长期维护的上位机平台，必须走到第一档。

## 2. Rapid SCADA 的解法：驱动是"数据"，不是"代码"

它把驱动做成了**独立的 DLL 库 + 一份 XML 配置**：

```
ScadaComm/                  通讯服务（不关心任何具体协议）
  └─ 按配置加载 Drv*.dll     ← 驱动按需加载，支持热插拔
DrvModbus/  DrvSnmp/ ...    每个驱动独立项目，独立版本号
  ├─ *Logic.cs              协议实现（读/写/解析）
  ├─ *View.cs               参数编辑界面（可无）
  └─ *.xml                  参数结构定义
```

三个关键决策值得抄：

1. **通讯服务不认识任何品牌**。它只认识"设备 + 驱动的逻辑接口"。删掉 `DrvModbus.dll`，服务照常启动，只是那几台设备报"驱动未找到"。
2. **驱动的参数界面由驱动自己提供**（`View`），主程序不需要为每个品牌画配置窗体。这是解决"接新品牌要改界面"的根。
3. **配置以 XML 描述**，参数 schema 跟着驱动走，主程序不硬编码字段。

> 具体类名/接口以官方文档和 v6 源码为准；本周要学的是**这三点结构决策**，不是背 API。

## 3. 你自己的插件契约（本周动手写）

目标：**.NET Framework 4.8 可用，不引入 DI 容器，一个 `drivers/` 目录丢 DLL 就能扩展。**

```csharp
// ① 统一配置：所有驱动只读自己关心的字段（解决 IoTClient 工厂参数过宽）
public sealed class DeviceConfig
{
    public string Name { get; set; }
    public string DriverKey { get; set; }          // 例如 "modbus-tcp" / "siemens-s7-1200"
    public string Host { get; set; }
    public int Port { get; set; } = 502;
    public string SerialPort { get; set; }
    public int BaudRate { get; set; } = 9600;
    public int TimeoutMs { get; set; } = 1500;
    public Dictionary<string, string> Extra { get; set; } = new();  // 品牌私有参数
}

// ② 驱动契约：只留 6 个成员（对照 IIoTClient 砍掉 25 个重载）
public interface IDeviceDriver : IDisposable
{
    string Key { get; }                            // 唯一标识，与 DriverKey 对应
    string DisplayName { get; }                    // 界面显示名
    DeviceState State { get; }                     // Online / Offline / Warning
    Result Open(DeviceConfig cfg);
    Result Close();
    Result<IDictionary<string, object>> ReadBatch(IList<PointRef> points);
    Result Write(string address, object value);
}

// ③ 发现机制：特性 + 反射，不需要改主程序
[AttributeUsage(AttributeTargets.Class)]
public sealed class DeviceDriverAttribute : Attribute
{
    public string Key { get; }
    public DeviceDriverAttribute(string key) => Key = key;
}
```

**加载器（40 行，写一次终身受益）**：

```csharp
public static class DriverRegistry
{
    static readonly Dictionary<string, Func<IDeviceDriver>> _map =
        new(StringComparer.OrdinalIgnoreCase);

    public static void LoadFrom(string driversDir)
    {
        foreach (var dll in Directory.GetFiles(driversDir, "*.Driver.dll"))
        {
            var asm = Assembly.LoadFrom(dll);
            foreach (var t in asm.GetTypes())
            {
                if (t.IsAbstract || !typeof(IDeviceDriver).IsAssignableFrom(t)) continue;
                var attr = t.GetCustomAttribute<DeviceDriverAttribute>();
                if (attr == null) continue;
                _map[attr.Key] = () => (IDeviceDriver)Activator.CreateInstance(t);
            }
        }
    }

    public static IDeviceDriver Create(string key) =>
        _map.TryGetValue(key, out var f)
            ? f()
            : throw new InvalidOperationException($"未找到驱动 [{key}]，请检查 drivers 目录");

    public static IEnumerable<string> Keys => _map.Keys;
}
```

**验收：把这段代码的"改动文件数"从 3 降到 0。**

- 新品牌 → 新建一个 `Xxx.Driver` 类库，加 `[DeviceDriver("xxx")]`，编译后丢进 `drivers/`；
- 主程序：**一行不改，不重新编译**；
- 参数界面：先用 `Extra` 字典 + 通用键值对编辑器顶住，后续再让驱动提供自己的编辑面板。

⚠️ **两个必踩的坑，提前埋点**：
1. `Assembly.LoadFrom` 会锁文件，覆盖 DLL 前必须先 `AppDomain.CurrentDomain.SetShadowCopyPath` 或重启程序。生产环境用**独立 `AppDomain` 或 `AssemblyLoadContext`（.NET Core+）**。
2. 驱动 DLL 和主程序的公共契约必须**单独抽一个 `Xxx.Contracts` 程序集**，只放接口/DTO，版本号锁定。否则主程序一升级，所有驱动全废。

## 4. 收口：把自研工具库变成能给人看的仓库

你手上已经有的东西（多语言资源提取、EF6 数据层、Excel 读写、NModbus 轮询封装、控件封装）现在是"散装能力"，收口成仓库才有价值。

**不要一次全放。** 按"能被别人 5 分钟跑起来"倒推，一次只开源一个组件。

推荐首发：**`PollingKit`（NModbus 轮询封装）**——因为它是你最有含金量、也最容易演示的一块。

仓库最小合格线：

```
PollingKit/
├── README.md              ① 解决什么问题 ② 30 秒上手 ③ 设计取舍 ④ 性能数据
├── LICENSE                MIT（想被商用就 MIT，别用 GPL）
├── src/PollingKit/        只放这一个组件，别塞别的东西
├── samples/WinFormsDemo/  ★ 能双击运行的演示，比文档重要 10 倍
├── tests/                 哪怕只有 5 个测试，也证明你知道怎么测
└── docs/design.md         为什么这么设计（面试官只看这个）
```

README 里最该写的**不是 API 列表，是这三条**：

1. **"为什么需要它"**：NModbus 只给你 `ReadHoldingRegisters`，但现场需要"分组批量读 + 断线重连 + 值变化事件 + 三态设备状态"。
2. **"它比直接用 NModbus 好在哪"**：一次轮询的往返次数从 N 降到 ~3（附你自己测的数据）。
3. **"它不做什么"**：不做界面、不做数据库、不做报警。**明确边界比功能多更显专业。**

**加分项（面试时非常有效）**：把 W1 的地址合并算法做成 `PollingKit.Core` 里一个**纯函数 + 单元测试**，README 里贴测试输出。这说明你会把算法从 IO 里剥出来测——80% 的上位机开发者做不到。

## 5. 四周之后的自己，应该能回答这四个问题

1. 你的通讯层一次轮询发几帧？为什么是这个数？
2. 加一个西门子 S7 驱动，要改几个文件？
3. 曲线为什么卡，你怎么定位的？
4. 断线时你的程序是什么行为？日志里能看到什么？

**这四个问题就是面试的技术面主线。** 答得出来，8 年经验才立得住。

---

## 自测题

1. 为什么驱动的公共契约要单独一个程序集？
2. `Assembly.LoadFrom` 锁文件会导致什么运维问题？怎么绕？
3. 插件化之后，"驱动参数界面"由谁负责？为什么不该由主程序负责？
4. 为什么开源首发选轮询库，而不是 EF6 数据层？
5. 为什么要写"它不做什么"？

<details>
<summary>参考答案</summary>

1. 主程序与驱动的版本契约只有一个来源。若契约混在主程序里，主程序升级会改契约版本，所有已部署驱动必须重编，插件化的意义就没了。
2. DLL 被进程锁定，无法热更新/覆盖，只能停机重启；现场升级驱动要停线。用独立 AppDomain（.NET Framework）或 AssemblyLoadContext（.NET Core+）加载，卸载时释放。
3. 驱动自己提供（可选实现一个 `IConfigEditor`）。主程序不可能预知每个品牌的参数，硬编码就意味着"接新品牌要改界面"，而这正是插件化要消灭的东西。
4. 演示成本最低：一个 WinForms demo + 一段模拟从站，5 分钟能跑；指标可量化（往返次数、延迟）。EF6 数据层依赖数据库环境，别人跑不起来，说服力反而弱。
5. 边界清晰 = 设计成熟。什么都能做的库没人敢用；明确"不做界面/不做报警"说明你想过职责划分，也降低了使用者的评估成本。
</details>

---

## 本周产出物

1. `IDeviceDriver` + `DeviceDriverAttribute` + `DriverRegistry` 三件套（<150 行），跑通"丢 DLL 就多一个驱动"。
2. `PollingKit` 仓库骨架 + README 草稿（按上面 6 项）。
3. 把"改动文件数判据"写进 [../docs/interview-cards.md](../docs/interview-cards.md)。

## 一句话带走

> 平台和库的区别只有一条：**库要求使用者改代码，平台只要求使用者放文件。**
