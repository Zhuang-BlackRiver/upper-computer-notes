# upper-computer-notes

> C# 上位机（工业自动化 / SCADA / HMI）开源项目源码精读笔记。
> 不是 API 抄写，而是**从别人的实现里提炼可复用的设计决策**，并且每条结论都能指到具体文件的具体行。

## 为什么有这个仓库

做了 8 年非标自动化上位机，写的通讯层、轮询、曲线、界面大多是"就地长出来的"。
这套笔记换一种方式：**先读被上千人 fork 过的代码，再倒推自己的设计哪里欠债。**

每一周产出的不是"我读完了"，而是三条能讲给面试官听的技术决策 + 一份能跑的最小代码。

## 四周路线

| 周 | 主题 | 精读项目 | 核心问题 | 产出 | 状态 |
|---|---|---|---|---|---|
| W1 | 通讯层抽象 | [IoTClient](https://github.com/zhaopeiym/IoTClient) | 上位机为什么不该"一个设备一个类"？批量读怎么合并地址？ | `IDeviceDriver` 抽象草案 + 批量读合并算法 | ⬜ |
| W2 | 完整项目分层 | [Wu.CommTool](https://github.com/Monika1313/Wu.CommTool) | 15 个协议模块怎么塞进一个程序还不乱？ | 模块化目录模板 + 分层判据 | ⬜ |
| W3 | 实时数据到界面 | [ScottPlot](https://github.com/ScottPlot/ScottPlot) | 万点波形怎么不卡？轮询线程怎么不冻界面？ | WinForms 万点实时曲线 demo | ⬜ |
| W4 | 驱动插件化与收口 | [RapidScada/scada-v6](https://github.com/RapidScada/scada-v6) | 新增一个设备品牌要改几个文件？ | 自研工具库开源化收口 | ⬜ |

## 目录

```
week01-comm-abstraction/
  README.md                 精读主线：Result / IIoTClient / SocketBase / 工厂
  DEEP-DIVE-batchread.md    批量读地址合并算法（逐行拆）
week02-project-structure/
  README.md                 Prism 模块化、Models/VMs/Views 边界、设备状态机
week03-realtime-chart/
  README.md                 渲染性能、数据抽稀、线程模型
week04-driver-plugin/
  README.md                 插件约定、驱动注册表、自研库收口清单
docs/
  interview-cards.md        每周产出的"面试可讲决策"卡片
  reading-log.md            阅读记录模板（按天记，防止读了个寂寞）
```

## 怎么用这套笔记

1. **先 clone 参考仓库到同级目录**，笔记里的路径都相对于各自仓库根：

```bash
git clone --depth 1 https://github.com/zhaopeiym/IoTClient.git
git clone --depth 1 https://github.com/Monika1313/Wu.CommTool.git
```

2. 打开笔记里的"对照文件"表格，**先只读被点名的那几十行**，不要从头看到尾。
3. 读完回答该周的"自测题"，答不上就回到对应行号。
4. 只有自测题过了，才允许自己动手写产出物。

## 三条贯穿全程的判断

- **上位机的性能瓶颈从来不在算，而在往返。** 一次 Modbus 往返 20~50ms，逐点轮询 200 个点就是 4~10 秒一轮；合并地址后可以压到 2~3 次往返。
- **通讯层不该抛异常给界面。** 现场断线是常态，不是异常。参考 `Result<T>` 的做法：返回值带成功标志 + 报文 + 耗时。
- **能加一个驱动而不改一行界面代码，才算分层成功。** 判据见 W4。

## 进度

- [ ] W1 通讯层抽象
- [ ] W2 完整项目分层
- [ ] W3 实时数据到界面
- [ ] W4 驱动插件化与收口

---

笔记内所有代码引用均为 MIT / 各自许可证下的开源项目，仅作阅读对照，未复制进本仓库。
