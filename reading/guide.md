# SREdgeCompute_TCP 源码学习详解 · 云端阅读版

[学习首页](../README.md) · [逐页讲稿与答疑](speaker.md) · [下载原始HTML](../SREdgeCompute_TCP_源码学习详解.html?raw=1)

这是原HTML内容的GitHub阅读版，可在登录有权限的GitHub账号后跨设备查看。源码位置保留为文本索引；本仓库没有上传项目源码，不依赖编写时的本机G盘。原HTML保留在仓库根目录。

> 材料仅用于源码学习与汇报；未连接现场PLC或Broker，未修改业务源码。隔离验证不等于原工程全量运行或现场验收。完整文件清单、135个命令索引、配置快照和原始探针记录保留在下方。

<a name="contents"></a>

## 章节目录

1. [01 边缘侧代码与通信链路 · SREdgeCompute_TCP](#s1)
2. [02 边缘侧承担协议适配与指令编排](#s2)
3. [03 一条业务链路跨越五个参与方](#s3)
4. [04 目标解决方案包含两类项目](#s4)
5. [05 启动过程先建立通信，再运行消费循环](#s5)
6. [06 六个服务构成边缘侧主执行链](#s6)
7. [07 Protocol 和 Model 处于不同数据层](#s7)
8. [08 MQTT 通过主题完成发布与订阅](#s8)
9. [09 QoS 确认的是消息投递层](#s9)
10. [10 当前 MQTT 使用库默认项与少量配置](#s10)
11. [11 C3 的五个主题各有明确方向](#s11)
12. [12 下行从 HTTP 校验进入 MQTT 命令队列](#s12)
13. [13 状态上行从 TCP 拆帧到界面状态仓库](#s13)
14. [14 RETURN 把业务 ID 带回后端和前端](#s14)
15. [15 “完成”在三类命令中含义不同](#s15)
16. [16 三种心跳分别观察不同对象](#s16)
17. [17 TCP 使用五种固定布局结构](#s17)
18. [18 一条走行指令如何变成 18 字节](#s18)
19. [19 周期控制与脉冲共享同一份结构](#s19)
20. [20 注册表把命令名映射到处理委托](#s20)
21. [21 堆料策略固定发送七段点位](#s21)
22. [22 取料策略以五层数据完成一次确认](#s22)
23. [23 毫米波数据绕过普通命令队列](#s23)
24. [24 故障数据并入下一次状态上报](#s24)
25. [25 保留文件与未实现接口不能当作能力](#s25)
26. [26 离线探针揭示数据与回执风险](#s26)
27. [27 运行期可靠性仍需台架验证](#s27)
28. [28 安全、配置与部署必须单独验收](#s28)
29. [29 下一步用隔离台架补齐完整证据](#s29)
30. [30 源码学习可以按七个检查点复述](#s30)
31. [31 业务报文使用 JSON，PLC 使用字节帧](#s31)
32. [32 命令覆盖分布集中在堆取料与机构控制](#s32)
33. [33 结论均可回到源码与验证记录](#s33)
34. [34 已建立可追踪的边缘侧源码模型](#s34)

<a name="s1"></a>

· PPT 01

## 边缘侧代码与通信链路 SREdgeCompute_TCP

文件结构 · MQTT 上下行 · PLC 功能实现

### 源码详解与边界

汇报以当前工作区快照为准。目标是理解边缘程序如何把业务命令转换成 PLC 帧、如何把 PLC 状态转换成后端和前端可用的数据，以及源码中的可靠性边界。没有访问现场设备，也没有修改项目业务源码。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S38 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/SREdgeCompute_TCP.csproj:1

<a name="s2"></a>

01 / 总体定位 · PPT 02

## 边缘侧承担协议适配与指令编排

看清软件边界，才能准确理解“下发成功”。

### 边缘侧负责什么

- 接收 MQTT 命令，分发到处理方法。
- 编码 PLC 控制帧与策略帧。
- 解析 PLC 状态，发布状态与回执。

### 还需其他系统负责什么

- 后端负责业务校验与策略组织。
- PLC 负责现场动作与联锁执行。
- 前端通过 HTTP、SignalR 交互。

### 源码详解与边界

不能只根据 EdgeCompute 名称认定所有策略算法都运行在边缘。AutoStackActionService 将策略模型组织并序列化为 Value；边缘的 AutoStackPointCommand 做参数映射、固定分段、编码和确认轮询。PLC 程序没有包含在本目标源码中，因此现场联锁的正确性不能通过这些 C# 文件证明。

**源码依据**

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

S22 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/CommandManager.cs:53

S28 · Web/src/api/main/manualStackAction.ts:1

S29 · Web/src/stores/signalR/allData.ts:72

S40 · SRIntelligentSystem/SRIntelligentSystem.Application/Command/Control/Auto/AutoStackActionService.cs:60

<a name="s3"></a>

01 / 总体定位 · PPT 03

## 一条业务链路跨越五个参与方

下行：操作请求 → MQTT 命令 → PLC 控制；上行：PLC 状态 → MQTT → 界面。

Web 界面 → 业务后端 → MQTT Broker → 边缘程序 → PLC

HTTP / SignalR          MQTT / TCP          自定义 TCP

C3 示例：后端客户端 C3；边缘客户端 EdgeC3。
三个 PLC 端口：2001 状态与周期控制、2002 命令、2004 故障。

### 源码详解与边界

图中箭头表示下行方向，状态与回执沿反向路径上行。浏览器并非直接连接 MQTT：下发由 Web API 进入后端，实时展示使用 SignalR。Broker 负责消息路由，不负责解析业务 JSON 或执行机械动作。PLC 与边缘的连接使用自定义二进制 TCP 协议。

**源码依据**

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S22 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/CommandManager.cs:53

S23 · SRIntelligentSystem/Plugins/Dhhi.Net.EventBus/MqttEventSourceStorer.cs:23

S24 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:51

S28 · Web/src/api/main/manualStackAction.ts:1

S29 · Web/src/stores/signalR/allData.ts:72

<a name="s4"></a>

02 / 代码结构 · PPT 04

## 目标解决方案包含两类项目

53 个文件，45 个 C# 文件，14,697 行 C#；统计包含注释与空行。

| 目录 / 项目 | 职责 | 阅读重点 |
| --- | --- | --- |
| SREdgeCompute_TCP | net8.0 控制台主程序 | 入口、通信、命令、协议、模型 |
| RequireDocumentation | Roslyn 源码分析器 | XML 注释与字段分组检查 |
| Client / Service | 连接、事件与业务编排 | MQTT ↔ 队列 ↔ PLC |
| Protocol / Model | 二进制契约与业务数据 | 类型、字节序、位域、缩放 |
| Infrastructure / Resource | 命令注册与资源 | 特性注册才是当前命令入口 |

### 源码详解与边界

统计由本次目录扫描生成，完整逐文件清单与 SHA-256 位于配套学习文档。主程序 41 个 C# 文件，分析器 4 个。CommandService.cs 6053 行，占全部 C# 行数约 41.2%，阅读时应按命令类别拆分。两个 MainData 不要混淆：Protocol/MainData 是当前解析入口，Model/State/MainData 是另一个状态模型，未在当前主链路中使用。分析器 csproj 重复声明 TargetFramework，后一个 netstandard2.0 生效。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S38 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/SREdgeCompute_TCP.csproj:1

S39 · Edge/SREdgeCompute_TCP/RequireDocumentation/FieldGroupAnalyzer.cs:30

S07 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs:35

<a name="s5"></a>

02 / 代码结构 · PPT 05

## 启动过程先建立通信，再运行消费循环

入口在 Program.Main；依赖以单例为主，由 Generic Host 管理。

1. **01 / 构建 Host**注册 Config、TCP、MQTT、 命令服务与后台消费者。
2. **02 / 强制解析服务**启动 EdgeToPlcService； 解析 PlcToEdgeService， 绑定状态与故障事件。
3. **03 / 运行后台任务**host.Run() 启动消费者； 关闭时停止周期发送， 释放连接与定时器。

### 源码详解与边界

服务注册本身不会立即创建所有单例。Main 明确 GetRequiredService&lt;EdgeToPlcService&gt;() 和 PlcToEdgeService；自定义工厂在创建 EdgeToPlcService 时调用 Start，并注册 ApplicationStopping。MQTT/TCP 构造器也启动异步逻辑，其中 StartAsync/Init 没有完整 await 启动门控；必须区分 Build、实例化和实际连接建立三个时刻。Program 后半段生成结构文档的辅助方法在 Main 中调用被注释，不属于运行主路径。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S06 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs:15

S10 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/EdgeToPlcService.cs:30

<a name="s6"></a>

02 / 代码结构 · PPT 06

## 六个服务构成边缘侧主执行链

沿事件和调用方向读代码，比按文件字母顺序阅读更有效。

| Service 文件 | 输入 | 输出 / 作用 |
| --- | --- | --- |
| ConfigService | App.config | 只读配置字段 |
| CommandQueueService | MQTT 命令 / 毫米波 | 命令入队；毫米波直接转发 |
| CommandQueueBackgroundService | 内存命令队列 | 串行 await 消费 |
| CommandService | CommandModel | 修改周期帧 / 发送即时帧 |
| EdgeToPlcService | 共享控制结构与脉冲 | 每轮发送后延迟 500 ms |
| PlcToEdgeService | TCP 状态 / 故障事件 | AllData、心跳、RETURN |

### 源码详解与边界

队列使用 ConcurrentQueue&lt;CommandModel&gt;，只解决线程安全入队出队，并不提供持久化、去重、优先级或容量上限。消费者一次 await 一条命令，策略确认等待会阻塞后续普通命令。毫米波消息不经过该队列，直接调用 SetMMWData 修改周期控制结构。500 ms 是代码中固定延迟，不是精确周期，更不是硬实时承诺。

**源码依据**

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S06 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs:15

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

S10 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/EdgeToPlcService.cs:30

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

<a name="s7"></a>

02 / 代码结构 · PPT 07

## Protocol 和 Model 处于不同数据层

同一个数值，在线路上和 JSON 中可能具有不同的类型与单位。

### Protocol：PLC 字节契约

- Sequential + Pack=1 固定布局。
- 私有数值字段 + 属性字节序转换。
- 固定数组承载策略点；位域承载控制。

### Model：业务表达

- Command：ID、Cmd、dynamic Value。
- PLC：AllData 聚合各机构与作业状态。
- Auto：堆取料参数、点位与层信息。

### 源码详解与边界

AllDataModel 包含 MainData、ErrorData、ReclaimData、StackData、AuxiliaryInstitutionsData、MillimeterWave，以及 VerificationId/CmdId/StrategyId/CmdStatus/StrategyCmdStatus。CommandModel.Value 是 dynamic，并没有统一强类型契约；有的方法期待数字字符串，有的方法期待嵌套 JSON 字符串。ReturnCommandModel.CommandState 是业务枚举，默认 JSON 序列化为数值。Marshal 使用字段而不是 C# 属性说明来决定布局，应以真实类型和离线尺寸为准。

**源码依据**

S16 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/MainData.cs:1027

S17 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ActionProcessControl.cs:15

S18 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/RealTimeControl.cs:15

S19 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/StackStrategyProcessControl.cs:14

S20 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ReclaimStrategyProcessControl.cs:10

S35 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs:19

<a name="s8"></a>

03 / MQTT 基础 · PPT 08

## MQTT 通过主题完成发布与订阅

Topic 负责路由；Payload 是应用自定义的数据。

1. **CONNECT / 建立会话**客户端向 Broker 连接； CONNACK 返回连接结果。
2. **SUBSCRIBE / 登记主题**订阅者登记过滤器； SUBACK 确认订阅。
3. **PUBLISH / 投递消息**发布者给主题发送负载； Broker 转给匹配的订阅者。

### 源码详解与边界

MQTT 是发布/订阅消息协议，通常运行于 TCP。发布和订阅角色可以由同一客户端承担。Topic 大小写敏感，/ 表示层级，+ 匹配单层，# 匹配多层。主题是 Broker 路由键，不是磁盘路径。当前边缘订阅配置给出的两个精确主题；后端 EventSubscribe 中的正则表达式是在事件总线内分发，不能与 MQTT 通配符混为一谈。

**源码依据**

OASIS MQTT 3.1.1 标准 §3.1–3.3、§3.8、§4.3、§4.7：[https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) （访问 2026-08-28）

S03 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/MqttClient.cs:29

<a name="s9"></a>

03 / MQTT 基础 · PPT 09

## QoS 确认的是消息投递层

协议确认不能替代业务执行结果。

| 等级 | 确认过程 | 理解边界 |
| --- | --- | --- |
| QoS 0 | PUBLISH；无协议确认 | 至多一次，可能丢失 |
| QoS 1 | PUBLISH → PUBACK | 至少一次，可能重复 |
| QoS 2 | 四步消息握手 | 协议接收方只交付一次 |
| 业务 RETURN | 项目自定义 JSON | 仍需判断 PLC 和动作状态 |

### 源码详解与边界

QoS 2 过程为 PUBLISH、PUBREC、PUBREL、PUBCOMP。它不自动提供整个业务链路的事务性：处理程序重试、数据库操作和 PLC 执行都需要各自的幂等与确认设计。当前边缘和后端调用的 ManagedClient 扩展默认 QoS 为 0，不应将现有系统称为可靠必达链路。

**源码依据**

OASIS MQTT 3.1.1 标准 §3.1–3.3、§3.8、§4.3、§4.7：[https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) （访问 2026-08-28）

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S35 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs:19

<a name="s10"></a>

03 / 项目 MQTT · PPT 10

## 当前 MQTT 使用库默认项与少量配置

边缘依赖 MQTTnet 4.3.6.1152；下表为边缘客户端设置。

| 配置项 | 当前取值 | 依据 |
| --- | --- | --- |
| 协议 / 端口 | MQTT 3.1.1 / 1883 | 库默认 / App.config |
| 发布 / 订阅 QoS | 均为 0；retain=false | 扩展方法默认参数 |
| 会话 / Keep Alive | CleanSession=true / 15 s | MqttClientOptions 默认 |
| 自动重连 | 失败后延迟 5 s | 边缘显式设置 |
| 认证 / TLS / 遗嘱 | 本包装代码未配置 | 不代表部署已具备保护 |

### 源码详解与边界

客户端调用 WithTcpServer 和 WithClientId，没有显式设置协议版本、KeepAlive、CleanSession、账号、TLS 或遗嘱；版本固定的 MQTTnet 源码确认相关默认值。Retain 是保留主题最后一条消息，不是历史消息库；持久会话、ManagedClient 本地排队和业务队列是不同机制。项目未接入 ManagedClient 持久存储，不能把 EnqueueAsync 当作已经被 Broker 或 PLC 接受。 后端 Dhhi.Net.Mqtt 工程引用 ManagedClient 4.3.3.952（并非与边缘完全同版）；已检查该版本 EnqueueAsync/SubscribeAsync 默认 QoS 同为 0，retain=false。

**源码依据**

S02 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/App.config:1

S03 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/MqttClient.cs:29

S41 · SRIntelligentSystem/Plugins/Dhhi.Net.Mqtt/DhhiMqttClient.cs:11

MQTTnet v4.3.6.1152 ManagedMqttClientExtensions.cs：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs) （访问 2026-08-28）

MQTTnet v4.3.6.1152 MqttClientOptions.cs：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs) （访问 2026-08-28）

S43 · SRIntelligentSystem/Plugins/Dhhi.Net.Mqtt/Dhhi.Net.Mqtt.csproj:1

后端依赖 MQTTnet ManagedClient v4.3.3.952 的默认参数：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.3.952/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.3.952/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs) （访问 2026-08-28）

<a name="s11"></a>

03 / 项目 MQTT · PPT 11

## C3 的五个主题各有明确方向

上行、下行均以边缘程序为参照。

| Topic | 方向 | Payload / 接收端 |
| --- | --- | --- |
| /MACHINE/CMD/C3 | 后端 → 边缘 | CommandModel |
| /MACHINE/STATE/C3 | 边缘 → 后端 | AllDataModel |
| /MACHINE/RETURN/C3 | 边缘 → 后端 | ReturnCommandModel |
| /HeartBeat/Edge/C3 | 边缘 → 后端 | PLC VerifyID 数字文本 |
| /Data/MMWave/C3 | 外部采集 → 边缘 | 10 路距离；发布端待确认 |

### 源码详解与边界

C3 是当前 App.config 示例，不能直接视为所有现场设备配置。后端 JSON 还配置了 C4、C5、C6 以及 Console、Truck、Heartbeat 客户端。毫米波主题在目标源码中可确认订阅端，但没有找到该主题的实际发布实现；该边界明确标为外部待确认。故障没有独立上报 MQTT 主题，故障字典并入下一次 STATE。MQTT_CTRL_TOPIC 只有读取字段，配置中没有对应项，当前也没有控制主题订阅。

**源码依据**

S02 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/App.config:1

S03 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/MqttClient.cs:29

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S24 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:51

S25 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:251

S27 · SRIntelligentSystem/Plugins/Dhhi.Net.EventBus/Configuration/Mqtt.json:1

<a name="s12"></a>

04 / 完整下行链路 · PPT 12

## 下行从 HTTP 校验进入 MQTT 命令队列

以 WalkTargetAciton 为例；保留源码中的 Aciton 拼写。

| 步骤 | 源码入口 | 关键数据 |
| --- | --- | --- |
| 1 · 用户请求 | manualStackAction.ts | POST equipmentCode + value |
| 2 · 业务校验 | ManualStackActionService → CommandUtility | 设备、模式、指令占用 |
| 3 · 记录与发布 | CommandManager.ExcuteCommand | 生成 long ID；存库；序列化 |
| 4 · MQTT 发送 | EventSourceStorer → MultiMqttClient | CMD/C3；ClientId=C3 |
| 5 · 边缘分发 | MessageReceived → Queue → Registry | Cmd → 处理委托 |
| 6 · PLC 命令 | WalkTargetAciton → SendCmdDataAsync | 18 字节；TCP 2002 |

### 源码详解与边界

HTTP /api/manualStackAction/SWalkTargetAciton 通过动态 API 对应服务方法。服务把业务命令名转换为 WalkTargetAciton。CommandManager 保存数据库 Command 和内存状态，再用 MqttEventSource(equipmentCode, CmdTopic+equipmentCode, payload) 交给事件总线。WriteAsync 对 MqttEventSource 特判并调用 MQTT EnqueueAsync。边缘 UTF-8 解码、JSON 反序列化、队列消费、大小写不敏感注册表匹配，最后发出 PLC 二进制帧。普通 API 返回“指令下达成功”不等待机械动作完成。

**源码依据**

S28 · Web/src/api/main/manualStackAction.ts:1

S30 · SRIntelligentSystem/SRIntelligentSystem.Application/Command/Control/Manual/ManualStackActionService.cs:31

S31 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Utility/CommandUtility.cs:69

S22 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/CommandManager.cs:53

S23 · SRIntelligentSystem/Plugins/Dhhi.Net.EventBus/MqttEventSourceStorer.cs:23

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S07 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs:35

S09 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:2895

<a name="s13"></a>

04 / 完整上行链路 · PPT 13

## 状态上行从 TCP 拆帧到界面状态仓库

STATE 数据由 PLC 到帧事件驱动，并非 LoadTime 定时发布。

| 步骤 | 处理位置 | 结果 |
| --- | --- | --- |
| 1 · 拆帧 | TCPClient.ReceiveStream | 2 字节长度头 → 完整状态体 |
| 2 · 映射 | CatchData → UpdateAllData | 字节序、枚举、缩放、时间 |
| 3 · 发布 | MqttClient.PublishMessage | STATE/C3；JSON |
| 4 · 后端消费 | EventBus → MachineStateData | 设备状态 + 多个业务事件 |
| 5 · 前端展示 | SignalR AllData_C3 → useAllData | Pinia result 驱动界面 |

### 源码详解与边界

TCP 接收缓冲区可以累计半包并循环取出粘连的完整包。MainData.FromBytes 最小接受 198 字节，短于当前 218 字节的扩展部分补零。UpdateAllData 把当前原始状态映射为业务对象；随后同时推进心跳、回执和 STATE 发布。后端先获取旧设备快照，再更新设备状态，派发模式日志、PLC 告警、音频、作业量、维护和设备状态事件，最后 SignalR 广播 AllData_&lt;设备号&gt;。前端存在 mock 分支，联调时要确认未启用。

**源码依据**

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

S16 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/MainData.cs:1027

S23 · SRIntelligentSystem/Plugins/Dhhi.Net.EventBus/MqttEventSourceStorer.cs:23

S24 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:51

S29 · Web/src/stores/signalR/allData.ts:72

<a name="s14"></a>

04 / 回执链路 · PPT 14

## RETURN 把业务 ID 带回后端和前端

业务 long ID 与 PLC 短序号不能混用。

| 环节 | 实现 | 状态含义 |
| --- | --- | --- |
| 边缘处理方法 | PublishReturn(JSON) | ID / Cmd / Result / CommandState |
| 普通主机构命令 | UpdateHostCommandStatus | Normal → Completed |
| 后端订阅 | 订阅 → UpdateCommand | 用业务 ID 关联内存与数据库 |
| 前端通知 | SignalR CommandResponse | 按设备筛选并显示日志 |
| 可选等待接口 | WaitForCommandCompletionAsync | 轮询 200 ms；调用方可等 35 s |

### 源码详解与边界

回执中 CommandState：New=1、Issued=2、Executing=3、Completed=4、Stop=5、Suspend=6、FaultCompleted=7；并非所有方法都发布完整状态序列。普通主机构路径只有一个 _hostCommand/_hostID 槽位；读取下一帧的 CommandStatus，不校验该帧 ActionCommandID 是否对应当前命令，也不检查机械 Completed 位。后端未找到内存中业务 ID 时忽略回执。isCommandStop 只对 Stop/FaultCompleted 移除缓存，不对 Completed 移除，属于后续需检查的生命周期问题。

**源码依据**

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S22 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/CommandManager.cs:53

S25 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:251

S32 · Web/src/views/main/unmannedSystemOfBucketWheelMachine/main/index.vue:119

S35 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs:19

S31 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Utility/CommandUtility.cs:69

<a name="s15"></a>

04 / 回执语义 · PPT 15

## “完成”在三类命令中含义不同

汇报和联调应明确每一种确认到底证明了什么。

1. **本地写入 / 模式与脉冲命令**修改共享结构后就回 Completed； 此时未必已发送到 PLC。
2. **状态确认 / 主机构动作命令**看到 CommandStatus=Normal 即回 Completed； 不是机械到位证明。
3. **传输确认 / 堆取料策略命令**轮询 StrategyCommandID； 匹配后完成策略下发， 不代表堆取料作业结束。

### 源码详解与边界

SwitchMode 修改 IsReclaimMode、IsManualMode、IsJoystickControl，取消暂停后立即回成功。SetStop 调用 SetPulse，未使用其布尔返回值就回成功。WalkTargetAciton 注册 host 槽位，随后由 PLC 状态触发回执。策略命令至少有段号确认，但只是比较当前缓存 ID，没有校验新帧时间戳；旧 ACK 也可能满足条件。三个层次都应与机械动作的真实完成状态分开。

**源码依据**

S36 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:273

S37 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:357

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S13 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:3451

S14 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:4760

<a name="s16"></a>

04 / 心跳与在线状态 · PPT 16

## 三种心跳分别观察不同对象

收到心跳、TCP Connected、机械可运行，不能相互替代。

| 机制 | 信号与节奏 | 实际观察对象 |
| --- | --- | --- |
| MQTT Keep Alive | 默认 15 s；PINGREQ / PINGRESP | 客户端与 Broker 连接 |
| 边缘 → PLC | VerifyID 每轮加 1，模 256 | 周期控制帧通道 |
| PLC → 后端 | 状态 VerifyID → HeartBeat Topic | PLC 状态链路到达 |
| 后端检查 | 每 1 s 检查；超过 30 s 清理命令 | 按最后接收时间，非计数增量 |

### 源码详解与边界

边缘没有单独的定时 MQTT 心跳任务；CatchData 收到状态帧后把 PLC 的 VerifyID.ToString() 发到 HeartBeat_Topic。后端 UpdateHeartbeatByEquipmentCode 每次更新 BeatTime，不验证值是否递增。HeartBeatChecker 超过 30 秒时清理该设备命令记录和计数器，源码没有在这里主动下发现场急停。心跳失联处置不能被描述为 PLC 安全停机策略。

**源码依据**

MQTTnet v4.3.6.1152 MqttClientOptions.cs：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs) （访问 2026-08-28）

S10 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/EdgeToPlcService.cs:30

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

S26 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Job/HeartBeatChecker.cs:20

S42 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/EquipmentsManager.cs:94

<a name="s17"></a>

05 / PLC 协议实现 · PPT 17

## TCP 使用五种固定布局结构

离线直接链接真实结构体测量；长度包含关系需按收发方向区分。

| 结构 | 长度 / 通道 | 用途 |
| --- | --- | --- |
| MainData | 218 B 状态体 / 2001 | 接收先剥离 2 B 长度头 |
| RealTimeControl | 162 B 完整帧 / 2001 | 周期控制，GuideByte=2 |
| ActionProcessControl | 18 B 完整帧 / 2002 | 机构动作，GuideByte=3 |
| StackStrategyProcessControl | 250 B 完整帧 / 2002 | 30 点一段，GuideByte=4 |
| ReclaimStrategyProcessControl | 60 B 完整帧 / 2002 | 固定 5 层，GuideByte=4 |

### 源码详解与边界

MainData 不包含 TCPClient 剥离的长度头；当前完整上行状态传输为 2+218 字节，旧状态体可为 198。发送结构的 Length 字段赋值为 Marshal.SizeOf，包括 Length 自身，不能照搬接收分帧公式。错误通道单独为 1 字节长度头，解析出的体再跳过 1 字节引导符。ActionProcessControl 的字段实际都是 short，部分注释仍写 1 字节，不应按旧注释实现 PLC 对接。这里列出实际调用的策略 GuideByte=4，未被调用的 ExecuteStrategyControlCommand helper 中写 5 不是当前传输事实。

**源码依据**

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S16 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/MainData.cs:1027

S17 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ActionProcessControl.cs:15

S18 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/RealTimeControl.cs:15

S19 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/StackStrategyProcessControl.cs:14

S20 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ReclaimStrategyProcessControl.cs:10

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s18"></a>

05 / 手动命令示例 · PPT 18

## 一条走行指令如何变成 18 字节

教学示例值 12.3；离线编码验证，不向真实设备发送。

### 业务负载

```text
{
  "ID": 42,
  "Cmd": "WalkTargetAciton",
  "Value": "12.3"
}

Value1 = round(12.3, 1) × 10 = 123
```

### 字段顺序 / 小端序

长度 18 → 引导符 3 → 模式 1
动作 ID 0 → 所属机构 1 → 命令码 3

Value1=123，Value2=0，Value3=0

12 00 03 00 01 00 00 00 01 00
03 00 7B 00 00 00 00 00

### 源码详解与边界

输入 Value 应为可解析字符串。WalkTargetAciton 用 float.TryParse；解析失败当前只写日志，没有 return，会继续使用默认 value=0，这是重要边界。成功时 Math.Round(value,1)*10，再 Convert.ToInt16，ModeCheck=1，Affiliation=1，ActionCommandCode=3。ExecuteProcessControl 设置 GuideByte=3、使用并递增 byte 命令计数器。示例假定初始 ActionControlID=0。输出是小端序；数值类型、缩放和范围必须与 PLC 表一致。

**源码依据**

S09 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:2895

S17 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ActionProcessControl.cs:15

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s19"></a>

05 / 实时控制 · PPT 19

## 周期控制与脉冲共享同一份结构

锁保护结构更新；500 ms 延迟循环与 1000 ms 脉冲由代码固定。

1. **修改 / ModifyData(ref data)**在 _dataLock 内更新结构。 数值参数与控制位 留到下一轮发送。
2. **发送 / 控制帧序列化**GuideByte=2；VerifyID 递增。 写 TCP 2001 后延迟 500 ms。
3. **复位 / SetPulse + Timer**布尔成员置 true，1 s 后复位。 相同成员重复脉冲被拒绝； 不同成员可以并存。

### 源码详解与边界

SetPulse 解析表达式树中的成员名，对结构体装箱后反射写入，再拆箱赋回 _data。定时器同样锁定后复位。GetData 返回的是结构体副本。发送环在 Task.Run 中运行，异常捕获后退出，不保证每 500 ms 必定发送。发送失败期间脉冲可能在真正到达 PLC 前已经消失；软件急停共享命令队列与该周期帧，不能视作硬件安全回路。

**源码依据**

S10 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/EdgeToPlcService.cs:30

S18 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/RealTimeControl.cs:15

S37 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:357

<a name="s20"></a>

05 / 命令分发 · PPT 20

## 注册表把命令名映射到处理委托

当前静态扫描到 135 个 [Command] 标记，9 个业务分组。

### 注册过程

- 扫描实例方法上的 [Command]。
- 校验签名：Task 方法 + 单个 CommandModel。
- 按名称注册；忽略大小写，支持别名。

### 执行与边界

- Cmd 查找委托并 await 执行。
- 未知命令返回 FaultCompleted。
- Priority 解决重名注册冲突， 不改变队列执行顺序。

### 源码详解与边界

CommandAttribute 还支持 Enabled 与 Priority，CommandInfo 存储名称、描述、分组、方法信息和处理委托。同名命令只由较小 Priority 替换已有注册。手工 Register/Unregister 等 API 存在，但没有发现当前业务中使用它们重排任务。Resource/CommandXml.xml 没有被当前服务加载，因此不能把 XML 里的命令名当成 MQTT 支持清单。配套文档自动列出全部 135 个标记的名称、方法和行号。

**源码依据**

S07 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs:35

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

<a name="s21"></a>

05 / 自动堆料 · PPT 21

## 堆料策略固定发送七段点位

AutoStackPointCommand：参数映射 → 零帧 → 7 × 30 点。

1. **ACK 0 / 先发零帧**GuideByte=4；段号 0。 等待缓存 StrategyID=0。
2. **ACK 1–7 / 固定七段**每段 30 个槽位，共 210。 三组数组取共同最短长度； 不足补 0，超出截断。
3. **RETURN / 确认下发结束**各段最多尝试 3 次； 每次轮询窗口 1 s。 全部匹配才回 Completed。

### 源码详解与边界

每段 250 字节，由 5 个 short 头字段和 Total/WalkPosition/RotatePosition/PitchPosition 四个 30 元素 short 数组组成。序号为全局 idx+1，坐标四舍五入一位后乘 10；数组项手动做字节序转换。StrategyCommandCode 优先用输入正值，否则 10。先调用 ApplyStackParamsToPlcFull 修改周期参数，再发送策略，二者没有事务化同步。失败分支在当前段停止并回 FaultCompleted；PLC 侧已接收部分数据的回滚未在该 C# 方法实现。

**源码依据**

S13 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:3451

S19 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/StackStrategyProcessControl.cs:14

S40 · SRIntelligentSystem/SRIntelligentSystem.Application/Command/Control/Auto/AutoStackActionService.cs:60

<a name="s22"></a>

05 / 自动取料 · PPT 22

## 取料策略以五层数据完成一次确认

AutoPickPointCommand：独立锁保护，先确认 0，再确认 1。

1. **参数 / 更新周期参数**层号、进尺、安全距离、 料流、负载与边界等 写入周期结构。
2. **数据 / 60 B 固定五层帧**每层包含层号、走行、 回转、俯仰和终点。 数据帧段号固定为 1。
3. **确认 / StrategyID 轮询**零帧最多 3 次； 数据帧发送一次、等 1 s。 失败回 ACK_*_NOT_READY。

### 源码详解与边界

不能把堆料重试策略直接套用到取料。取料的 SendZeroAndWaitAsync 有三次尝试；实际数据帧只发一次，随后 WaitForExpectedWithEdgeIfNeededAsync 等待 1 秒。两个方法名虽含 EdgeIfNeeded，实际只判断缓存的 StrategyCommandID 是否等于预期，没有检测新帧边沿或 StrategyCommandStatus。_reclaimLock 只保护取料策略入口；整个命令消费者本来也是串行。开始取料、暂停、换层和参数修改是其他命令，策略下发完成不等于作业已启动。

**源码依据**

S14 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:4760

S20 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/ReclaimStrategyProcessControl.cs:10

<a name="s23"></a>

05 / 外部传感数据 · PPT 23

## 毫米波数据绕过普通命令队列

可确认边缘接收与 PLC 往返路径；外部发布实现不在目标源码中。

外部采集 → MQTT 主题 → SetMMWData → 周期控制帧 → PLC

输入距离 × 100 → 限幅到 short → 10 路寄存字段

回传应恢复工程值；现有 short / 100 会截断小数。
离线例：原值 123，映射结果 1，预期按比例恢复为 1.23。

### 源码详解与边界

MMWDataModel 定义 10 路 float，leftFront/leftMid/leftBack/rightFront/rightMid/rightBack/advanceLeft/advanceRight/retreatLeft/retreatRight。SetMMWData 依次映射至 MillimeterWaveRadar1..10，乘 100 并 Math.Clamp 到 short 范围。PLC 的距离反馈再由 MainData 映射到 AllData.MillimeterWave。此处的 /100 为 short 整数除法，即便目标是 float 也已丢失小数；日志 /100.0 与业务结果可能不一致。

**源码依据**

S02 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/App.config:1

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S15 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:186

S34 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:494

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s24"></a>

05 / 告警实现 · PPT 24

## 故障数据并入下一次状态上报

故障通道与状态通道分离；告警字典随 STATE 一起发出。

### 当前处理逻辑

- TCP 2004：1 B 长度头。
- 跳过报文体引导字节，遍历置 1 位。
- 键名 Error_0001…，值为字符串 "1"。

### 离线复现的边界

- 先输入故障位 1：字典出现告警。
- 再输入全 0：MonitorChanges 返回 null。
- CatchErrorData 不更新字典， 导致旧告警继续存在。

### 源码详解与边界

DetectChanges 名称容易误导：它没有比较 _previousData，而是扫描当前数据所有置位位。因此返回的是当前活动位集合，不是变化增量。_previousData 只被保存与重置。完全清零时 changes.Count=0 返回 null，调用者只在非 null 时更新 ErrorData，所以最后一个故障无法通过这条分支清除。若仍有其他故障位，返回非空字典会替换旧集合。后端 WarningManager 用新旧 ErrorData 做告警处理，因此该问题可能影响恢复展示，需台架回归。

**源码依据**

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S33 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:88

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s25"></a>

06 / 实现边界 · PPT 25

## 保留文件与未实现接口不能当作能力

以实际引用、注册和执行路径为准。

| 文件 / 代码 | 当前发现 | 学习结论 |
| --- | --- | --- |
| Resource/CommandXml.xml | 未发现运行链路加载 | 只作历史参考 |
| Config_File_Path / LoadTime | 仅声明与读取 | 不是当前 Excel 点表采集循环 |
| MQTT_CTRL_TOPIC | 无配置项、无订阅使用 | 当前只订阅 CMD 与毫米波 |
| Model/State/MainData | 未进入当前状态解析链 | 不要与 Protocol/MainData 混用 |
| SetPower / SetMCCPower 等 | 空实现且未标记注册 | 不能按方法名认定 MQTT 可调用 |

### 源码详解与边界

目标目录没有 Resource/Data.xlsx，但当前 Config_File_Path 没有进一步使用，因此不应直接断言该文件缺失会阻止当前主链路。GuideByteCompute 读取 PLC IP 最后一段，但当前发送路径使用固定 2/3/4，引导符方法没有实际调用。CalibrateEncoderWithBeiDou、StartReclaimGroup、StartStackGroup、SetPower、SetMCCPower 为未实现示例；具体可调用清单仍以注册表为准。分析器负责代码规范检查，不运行于现场通信链路。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S02 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/App.config:1

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S07 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs:35

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

<a name="s26"></a>

06 / 已验证的源码行为 · PPT 26

## 离线探针揭示数据与回执风险

直接链接真实结构和服务；网络、日志、JSON 与配置依赖使用替身。

| 验证项 | 实际输出 | 影响 |
| --- | --- | --- |
| 故障清除 | 1 → 0 后 ErrorData 仍有 1 项 | 可能显示过期告警 |
| 毫米波比例 | 123 映射为 1 | 小数距离丢失 |
| 普通主机回执 | 无动作 ID 匹配也回 Completed | 确认层级和关联不足 |
| 大端位域 | 数值字段换序；控制位仍 01 00 | 与数值字段字节序不一致 |
| 旧帧兼容 | 197 B 拒绝；198 B 补零接受 | 缺失扩展字段被视作零 |

### 源码详解与边界

探针使用 SDK 9.0.301 编译链接源码，非原工程 net8.0 全量构建。大小端各运行一次。数值字段 Action.Value1=123 正确切换 7B00/007B；RealTimeControl 的 _mainCommand 直接按宿主位运算，不经过 Utility，切到大端后仍为 0100。当前配置为小端，所以这是配置切换时的兼容风险，不是已确认现场故障。普通回执测试给 _hostID=42，收到 ActionCommandID=0、CommandStatus=Normal 即返回状态4，证明没有绑定新动作序号；不证明现场必然误动作。

**源码依据**

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S16 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/MainData.cs:1027

S18 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Protocol/RealTimeControl.cs:15

S33 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:88

S34 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:494

<a name="s27"></a>

06 / 静态分析风险 · PPT 27

## 运行期可靠性仍需台架验证

以下为静态分析结论，尚需运行环境验证。

| 风险 | 源码依据 | 建议验证 |
| --- | --- | --- |
| 队列阻塞 / 空转 | 串行等待；空队列无 Delay | 策略传输时停止命令延迟、CPU |
| 发送失败被吞掉 | SendCmdDataAsync catch 后重连 | 失败状态、重发、幂等 |
| 旧 ACK 误匹配 | 只比较缓存 StrategyID | 新帧时间与策略批次关联 |
| 非法值继续执行 | TryParse 失败只写日志 | 非法输入应拒绝下发 |
| 故障通道单独断线 | EnsureConnection 只检查 Ctrl/Cmd | 2004 断开后的恢复路径 |

### 源码详解与边界

TCP 命令发送失败不向上抛异常，也没有返回 bool，业务 catch 未必能感知发送失败。EnsureConnection 的快捷成功判断只检查 Ctrl/Cmd，而 IsConnected 检查三个通道；单独故障通道中断可能长期不恢复。TCP 长度头是有符号 Int16，缺少负数、最大值与缓冲限制防护。MQTT MessageReceived.Invoke 未空值保护；非法 JSON 和 null CommandModel 的检查也不足，属于稳定性检查点。安全相关停止命令不能依赖普通队列优先级，现有 Priority 仅决定注册冲突。

**源码依据**

S06 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs:15

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S09 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:2895

S13 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:3451

S14 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:4760

<a name="s28"></a>

06 / 联调前置条件 · PPT 28

## 安全、配置与部署必须单独验收

源码分析结论不等于可以直接连接现场生产设备。

### 部署与配置核对

- 核对 Broker 地址及每台唯一 ClientId。
- 确认 PLC 2001 / 2002 / 2004 和两端帧长、字节序、缩放。
- 确认 App.config 进入发布产物。

### 控制链路保护

- 认证、TLS、主题 ACL 需部署确认。
- 普通命令增加过期、去重与参数校验。
- 软件急停不能替代硬件联锁； 联调须在隔离台架进行。

### 源码详解与边界

两端配置中的 localhost/127.0.0.1 只有进程在相应本机可达 Broker 时才成立，跨机不能照抄。相同 Broker 上多个实例复用 C3 或 EdgeC3 会产生客户端身份冲突。发布配置是 Release、net8.0、win-x64、自包含；不能由存在 pubxml 就认定现场版本与当前源码完全一致。关于认证/TLS/ACL 的措辞是部署待确认，而不是断言现场完全没有网络隔离。

**源码依据**

S02 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/App.config:1

S03 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/MqttClient.cs:29

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S37 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:357

S38 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/SREdgeCompute_TCP.csproj:1

<a name="s29"></a>

07 / 验证路线 · PPT 29

## 下一步用隔离台架补齐完整证据

先协议，再通信，再业务闭环；禁止直接用教学报文驱动生产 PLC。

1. **协议层 / 固定帧回放**半包、粘包、非法长度、 198/218 B、大小端、 工程量与控制位。
2. **通信层 / 故障注入**Broker 重启、三个端口 分别断线、消息重复、 断电后队列与恢复。
3. **业务层 / 闭环验收**业务 ID ↔ PLC 序号关联； 策略中断、告警恢复、 停止时延与动作完成。

### 源码详解与边界

已完成的是静态调用链和隔离源码行为验证。台架需要独立 Broker、PLC 模拟器或离线 PLC、可记录帧的代理、预期报文表以及动作反馈模拟；测量条件与实际结果应存档。正常流与失败流都要核对 ID、主题、时间和十六进制帧，不能只看界面弹出“成功”。

**源码依据**

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S06 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs:15

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S13 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:3451

S14 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:4760

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s30"></a>

附录 A / 阅读路径 · PPT 30

## 源码学习可以按七个检查点复述

主报告之外，用这张表检查是否真正理解完整链路。

| 顺序 | 读什么 | 应能回答的问题 |
| --- | --- | --- |
| 1 | Program + App.config | 服务何时实例化？连接到哪里？ |
| 2 | MqttClient + Queue + Registry | 消息如何找到处理方法？ |
| 3 | TCPClient + Protocol | 如何拆帧、换序、定长编码？ |
| 4 | PlcToEdgeService + PLC Models | 状态、心跳、告警如何产生？ |
| 5 | CommandService 典型命令 | 参数、确认与失败如何处理？ |
| 6 | 后端 EventBus / CommandManager | 如何保存、发送、关联回执？ |
| 7 | Web API / SignalR / Store | 如何请求和更新界面？ |

### 源码详解与边界

推荐阅读顺序不等于所有代码同样重要。先掌握三条可追踪的真实样例：WalkTargetAciton、AutoStackPointCommand、AutoPickPointCommand，再按命令分组扩展阅读。完整目录和命令索引在配套 HTML 中，可搜索文件名或 Cmd，并定位到实际源码行号。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S03 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/MqttClient.cs:29

S04 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Client/TCPClient.cs:95

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

S11 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:34

S22 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Manager/CommandManager.cs:53

S23 · SRIntelligentSystem/Plugins/Dhhi.Net.EventBus/MqttEventSourceStorer.cs:23

S28 · Web/src/api/main/manualStackAction.ts:1

S29 · Web/src/stores/signalR/allData.ts:72

<a name="s31"></a>

附录 B / 报文契约 · PPT 31

## 业务报文使用 JSON，PLC 使用字节帧

以下为教学负载；Value 的具体结构随 Cmd 改变。

### CMD/C3 示例

```text
{
  "ID": 42,
  "Cmd": "WalkTargetAciton",
  "Value": "12.3"
}

ID：业务关联；不是 MQTT Packet ID
```

### RETURN/C3 示例

{"ID":42, "Cmd":"WalkTargetAciton",
 "CommandState":4, "Result":null,
 "Time":"2026-08-28T12:00:00+08:00"}

STATE：MainData + 各机构数据
以及 ErrorData、StrategyId 等。

HeartBeat：数字文本，例如 9。

### 源码详解与边界

普通数值命令 Value 多为字符串，原因是后端 CommandManager.ExcuteCommand 的 value1 接口为 string；策略命令 Value 则是 AutoStack/AutoPick 序列化后的 JSON 字符串，外层整体再序列化一次。示例 RETURN 仅演示字段与枚举值，并非现场抓包。ReturnCommandModel.Time 在不少即时命令里没有设置，可能出现默认日期；LifeCycle 也没有统一使用，不应把所有字段都称为有效业务数据。

**源码依据**

S05 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandQueueService.cs:30

S09 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:2895

S13 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:3451

S35 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs:19

<a name="s32"></a>

附录 C / 静态命令清单 · PPT 32

## 命令覆盖分布集中在堆取料与机构控制

135 个特性标记，9 个分组；不代表 135 项已通过现场验收。

| 分组 | 标记数 | 典型 Cmd |
| --- | --- | --- |
| 自动取料 / 自动堆料 | 32 / 30 | AutoPickPointCommand / AutoStackPointCommand |
| 辅助机构 / 润滑系统 | 30 / 8 | SwitchMode / TurnOnTravelingLubrication |
| 手动 / 挡板 | 12 / 9 | WalkTargetAciton / SetDamper1 |
| 流程 / 单机 | 6 / 6 | OneClickSwitchReclaimMode / SetWalkAdvanceCommand |
| 堆料流程 | 2 | SetStackRotateSpeedSetting / SetStackRotateDirection |

### 源码详解与边界

计数由当前 CommandService.cs 中有效行首 [Command(...)] 静态提取得到，完整 135 行索引保留命令名、说明、分组、方法和源码位置。注册表运行时还会检查 Enabled、签名和重名优先级，所以应称为“静态特性标记数”。本次统计没有发现重复命令名。不能用类中 public Task 方法总数代替已注册命令数。

**源码依据**

S07 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs:35

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

<a name="s33"></a>

附录 D / 证据与范围 · PPT 33

## 结论均可回到源码与验证记录

PPT 每页备注包含 [Sources]；配套 HTML 提供完整文件与命令索引。

### 本次采用的证据

- 当前工作区源码与配置。
- OASIS MQTT 3.1.1 标准。
- MQTTnet 固定版本源码， 以及隔离 .NET 源码探针。

### 本次没有证明的事项

- 现场 PLC 逻辑与联锁正确性。
- Broker 实际部署、网络和账号策略。
- net8.0 全应用构建、现场性能 及整套控制系统验收结果。

### 源码详解与边界

现有接手说明介绍了 ConsoleEdge 等其他工程，不作为本目标版本行为的唯一证据。README 所指向的在线 TCP/UDP 协议表没有纳入当前实测依据；需要现场提供确认后的协议版本进行逐字段比对。源码探针使用 SDK9，不调用 TCP/MQTT 真连接、不修改主项目。源码快照未附可用 Git 提交号，配套文档用文件 SHA-256 标识已阅读的目标版本。

**源码依据**

S38 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/SREdgeCompute_TCP.csproj:1

OASIS MQTT 3.1.1 标准 §3.1–3.3、§3.8、§4.3、§4.7：[https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) （访问 2026-08-28）

MQTTnet v4.3.6.1152 ManagedMqttClientExtensions.cs：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet.Extensions.ManagedClient/ManagedMqttClientExtensions.cs) （访问 2026-08-28）

MQTTnet v4.3.6.1152 MqttClientOptions.cs：[https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs](https://raw.githubusercontent.com/dotnet/MQTTnet/v4.3.6.1152/Source/MQTTnet/Client/Options/MqttClientOptions.cs) （访问 2026-08-28）

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="s34"></a>

08 / 学习结论 · PPT 34

## 已建立可追踪的边缘侧源码模型

现在可以解释代码在哪、消息怎么走，以及每类“完成”证明到哪一步。

1. **结构 / 找到职责与入口**Host → 单例服务 → 事件 → 队列 → 命令处理器。
2. **链路 / 贯通上下行**HTTP / MQTT / TCP 下发； STATE / RETURN / SignalR 分别承载状态与结果。
3. **边界 / 用验证收尾**先修正数据与关联问题； 再通过隔离台架 验证故障恢复与控制闭环。

### 源码详解与边界

汇报结论：该边缘工程以协议适配、参数映射和命令传输为核心；MQTT 仅承担跨进程消息传递，PLC TCP 才是设备接口。学习完成的衡量是能沿源码解释一条命令和一个状态的生命周期，并能指出未证实的部分。后续工程改进优先级可从故障清除、距离缩放、参数校验和回执关联开始，再完善队列、重连和部署保护。

**源码依据**

S01 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Program.cs:28

S08 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/CommandService.cs:104

S12 · Edge/SREdgeCompute_TCP/SREdgeCompute_TCP/Service/PlcToEdgeService.cs:632

S24 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:51

S25 · SRIntelligentSystem/SRIntelligentSystem.Application.Dispatch/Subscriber/MqttEventSubscriber.cs:251

隔离验证：.codex-build/sredge-20260828/probe/Probe.csproj 与 Program.cs；2026-08-28；.NET SDK 9.0.301；直接链接真实协议和 PlcToEdgeService 源码。仅配置、日志、JSON、网络依赖使用替身，不等于 net8.0 全应用测试。

<a name="files"></a>

## 逐文件结构索引：53 个文件

以下相对于 Edge/SREdgeCompute_TCP；链接指向当前机器源码。主程序 41 个 C# 文件，分析器 4 个。行数包含注释和空行。

```text
Edge/SREdgeCompute_TCP/
├─ SREdgeCompute_TCP.sln
├─ RequireDocumentation/       编译期规范检查（4 个 C# 文件）
└─ SREdgeCompute_TCP/          边缘主程序（41 个 C# 文件）
   ├─ Program.cs / App.config / *.csproj
   ├─ Client/                 MQTT / TCP 包装与消息事件
   ├─ Service/                配置、队列、分发与双向 PLC 服务
   ├─ Infrastructure/Commands/特性、元信息、注册表
   ├─ Protocol/               二进制结构、字节序与引导符辅助
   ├─ Model/
   │  ├─ Command/             CMD、RETURN、毫米波负载
   │  ├─ PLC/                 STATE 上报数据
   │  ├─ Auto/                堆取料策略与参数
   │  ├─ Common/              PLC 类型名常量
   │  └─ State/               另一套状态模型（当前非主链路）
   ├─ Enum/                   机构、模式、状态枚举
   ├─ Resource/               历史命令 XML
   └─ Properties/PublishProfiles/  发布配置
```

| 文件 | 行数 | 职责 / 阅读边界 |
| --- | --- | --- |
| RequireDocumentation/DocumentationRequirementAnalyzer.cs | 85 | Roslyn DOC001：对有 RequireDocumentation 标记的类型检查公有字段/属性 XML 注释。 |
| RequireDocumentation/FieldGroupAnalyzer.cs | 82 | 检查属性访问字段分组；注册了方法节点却直接转属性节点，存在分析器异常风险。 |
| RequireDocumentation/FieldGroupAttribute.cs | 17 | 标识字段或属性分组名，供静态分析器使用。 |
| RequireDocumentation/RequireDocumentation.csproj | 21 | Roslyn 分析器工程；后声明的 TargetFramework=netstandard2.0 覆盖此前 net8.0。 |
| RequireDocumentation/RequireDocumentationAttribute .cs | 14 | 标识需要成员 XML 文档的类/结构体；文件名包含尾随空格。 |
| SREdgeCompute_TCP/App.config | 28 | 当前 C3 的 PLC/MQTT 地址、端口、主题、字节序配置；LoadTime 等部分字段没有进入运行链路。 |
| SREdgeCompute_TCP/Client/MessageEventArgs.cs | 24 | Topic、Payload、QoS、Retained 事件对象；当前回调只传前两项，后两项仍使用默认值。 |
| SREdgeCompute_TCP/Client/MqttClient.cs | 101 | 托管 MQTT 客户端；5 秒重连；订阅 CMD/MMW；UTF-8 解码、消息事件、异步发布。 |
| SREdgeCompute_TCP/Client/TCPClient.cs | 421 | 三条 TCP 连接；Ctrl/Error 长度前缀拆帧；Ctrl/Cmd 发送与不同重连路径；释放资源。 |
| SREdgeCompute_TCP/Enum/CommandAffilationEnum.cs | 259 | 机构编号，含走行/回转/俯仰/辅助机构/润滑/堆取料流程；枚举底层为 byte。 |
| SREdgeCompute_TCP/Enum/CommandStatusEnum.cs | 132 | 硬件模式、软件模式、流程状态、机构状态、报文命令状态及描述扩展；CommandStatus.Normal=0。 |
| SREdgeCompute_TCP/Enum/PointsStatusEnum.cs | 22 | 点状态枚举：正常、占用、通信异常；不能与 CommandState 混用。 |
| SREdgeCompute_TCP/Infrastructure/Commands/CommandAttribute.cs | 43 | 命令名、描述、Group、Enabled、Priority；允许同方法多个别名。部分注释存在编码显示异常。 |
| SREdgeCompute_TCP/Infrastructure/Commands/CommandInfo.cs | 66 | 命令元信息与 Func&lt;CommandModel,Task&gt; 委托；提供 ExecuteAsync。部分注释存在编码显示异常。 |
| SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs | 295 | 扫描、签名校验、大小写不敏感注册、优先级冲突处理、执行与查询。 |
| SREdgeCompute_TCP/Model/Auto/AutoPick.cs | 74 | 五层取料位置、边界、参数；供 AutoPickPointCommand 反序列化。 |
| SREdgeCompute_TCP/Model/Auto/AutoStack.cs | 85 | 完整堆料策略负载：参数、走行/回转/俯仰数组与策略命令码。 |
| SREdgeCompute_TCP/Model/Auto/OutOfPileAngleCommandModel.cs | 14 | 切入/切出角等参数模型；用于边界相关命令。 |
| SREdgeCompute_TCP/Model/Auto/Point.cs | 20 | 三维坐标辅助模型，服务于点位表达；是否进入某命令须沿引用核对。 |
| SREdgeCompute_TCP/Model/Auto/ReclaimParamsModel.cs | 65 | 取料参数 DTO：层、进尺、安全距离、料流、量、速度、料堆边界、斗轮负载等。 |
| SREdgeCompute_TCP/Model/Auto/StackParamsModel.cs | 70 | 堆料参数 DTO：安全距离、目标点、高度、XYZ 步距、作业范围、左右边界及停留时间。 |
| SREdgeCompute_TCP/Model/Command/CommandModel.cs | 16 | 命令 JSON 接收模型：long ID、string Cmd、dynamic Value。 |
| SREdgeCompute_TCP/Model/Command/MMWDataModel.cs | 22 | 十路毫米波 float 距离，以 left/right、front/mid/back、advance/retreat 命名。 |
| SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs | 57 | 业务回执模型与 CommandState 1..7；包含 Result、Time、LifeCycle。 |
| SREdgeCompute_TCP/Model/Common/DynamicModel.cs | 21 | PLC 类型名称常量，如 DINT、REAL、BOOL、INT；不是当前 TCP 反序列化引擎。 |
| SREdgeCompute_TCP/Model/PLC/AllDataModel.cs | 57 | STATE 根对象：主机、故障、堆取料、辅助机构、毫米波与指令/策略状态。 |
| SREdgeCompute_TCP/Model/PLC/AuxiliaryInstitutionsDataModel.cs | 843 | 辅助机构上报状态，覆盖斗轮、皮带、夹轨、尾车、挡板、电源、照明、润滑等。 |
| SREdgeCompute_TCP/Model/PLC/MainDataModel.cs | 525 | 上报主机位置、模式、状态、联锁/流程派生标志与累计作业量等。 |
| SREdgeCompute_TCP/Model/PLC/MillimeterWaveModel.cs | 121 | 毫米波距离和防碰撞相关状态，当前距离映射存在整数除法截断。 |
| SREdgeCompute_TCP/Model/PLC/ReclaimDataModel.cs | 99 | 取料上报：就绪、暂停、料流、累计量、边界和策略过程字段。 |
| SREdgeCompute_TCP/Model/PLC/StackDataModel.cs | 103 | 堆料上报：就绪、暂停、料流、累计量、落料高度、点位相关字段。 |
| SREdgeCompute_TCP/Model/PLC/StackPointModel.cs | 20 | 堆料点状态模型：short PointID 点序、bool PointStatus 点状态；由上报模型引用。 |
| SREdgeCompute_TCP/Model/State/MainData.cs | 194 | 另一套历史状态表达（WATCHDOG_SEND、RECLAIME_Auto_MODE 等）；当前 PlcToEdgeService 使用 Protocol.MainData，不使用此类型。 |
| SREdgeCompute_TCP/Program.cs | 476 | 程序入口与 DI 注册；强制创建两个 PLC 方向服务；包含当前未启用的文档生成辅助方法。 |
| SREdgeCompute_TCP/Properties/PublishProfiles/FolderProfile.pubxml | 17 | 文件夹发布：Release、net8.0、win-x64、自包含、非单文件、无裁剪。 |
| SREdgeCompute_TCP/Properties/PublishProfiles/FolderProfile.pubxml.user | 8 | 开发机发布个人状态，不作为公共部署契约。 |
| SREdgeCompute_TCP/Protocol/ActionProcessControl.cs | 143 | 18 字节动作帧；9 个 short：Length/Guide/Mode/ID/Affiliation/Code/Value1..3。 |
| SREdgeCompute_TCP/Protocol/GuideByteCompute.cs | 27 | 从 PLC IP 最后一段计算 byte；被注入但 GuideByte() 未进入当前发送调用链。 |
| SREdgeCompute_TCP/Protocol/MainData.cs | 1060 | 当前 PLC 上行状态结构体，218 字节；接收状态体至少 198 字节，缺失扩展部分补零。 |
| SREdgeCompute_TCP/Protocol/RealTimeControl.cs | 1423 | 162 字节周期控制帧；主指令和机构位域、堆取料参数、北斗字段、10 路毫米波。 |
| SREdgeCompute_TCP/Protocol/ReclaimStrategyProcessControl.cs | 115 | 60 字节取料帧；5 个头字段与5组×5项 short 固定数组。 |
| SREdgeCompute_TCP/Protocol/StackStrategyProcessControl.cs | 117 | 250 字节堆料帧；5 个头字段与4组×30项 short 固定数组。 |
| SREdgeCompute_TCP/Protocol/Utility.cs | 136 | 按 PLC_BigOrLittleEndian 与本机字节序判断 NeedSwap，提供整数/浮点换序。 |
| SREdgeCompute_TCP/Resource/CommandXml.xml | 60 | 历史命令映射资源；当前 Registry 扫描特性，未发现运行路径加载该 XML。 |
| SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs | 24 | BackgroundService 中持续 await 消费；无队列等待机制，空队列可能忙循环。 |
| SREdgeCompute_TCP/Service/CommandQueueService.cs | 60 | 消息路由和内存队列；CMD 入队，毫米波直接映射；每次执行消费一条。 |
| SREdgeCompute_TCP/Service/CommandService.cs | 6053 | 6053 行业务实现中心；135 个命令标记；普通控制、脉冲、辅助机构、手动定位、堆取料策略。 |
| SREdgeCompute_TCP/Service/ConfigService.cs | 48 | 读取 AppSettings 并暴露只读字段；没有热更新；无效/缺失数值配置需启动校验。 |
| SREdgeCompute_TCP/Service/EdgeToPlcService.cs | 376 | 维护 RealTimeControl 结构、锁、周期发送、VerifyID 和布尔脉冲 Timer。 |
| SREdgeCompute_TCP/Service/PlcToEdgeService.cs | 672 | 状态帧解析映射、故障位处理、业务心跳、主机构回执；AllData 供命令及上行共用。 |
| SREdgeCompute_TCP/SREdgeCompute_TCP.csproj | 39 | net8.0、unsafe、XML 文档生成；MQTTnet 4.3.6.1152、DI/Host、JSON、日志以及分析器引用。 |
| SREdgeCompute_TCP/SREdgeCompute_TCP.csproj.user | 6 | 开发环境用户级工程设置，不属于业务运行逻辑。 |
| SREdgeCompute_TCP.sln | 36 | 解决方案，组织主程序和分析器项目及构建配置。 |

<a name="commands"></a>

## 完整命令索引：135 个特性标记

来源：Service/CommandService.cs。Cmd 按源码保留大小写与拼写；表内行号为标记附近。是否支持还需结合注册签名与 Enabled；此清单不是现场测试通过清单。

| 分组 | Cmd | 说明 | 方法 | 行号 |
| --- | --- | --- | --- | --- |
| 辅助机构 | `RemoteZeroingCommand` | 远程归零指令 | RemoteZeroingCommand | 237 |
| 辅助机构 | `SwitchMode` | 模式切换 | SwitchMode | 272 |
| 辅助机构 | `SetStop` | 动作急停 | SetStop | 356 |
| 辅助机构 | `SetBucket` | 斗轮指令 | SetBucket | 407 |
| 辅助机构 | `SetControlPower` | 控制电源 | SetControlPower | 466 |
| 辅助机构 | `SetLight` | 照明 | SetLight | 508 |
| 辅助机构 | `TurnOnAlarm` | 开始警铃 | TurnOnAlarm | 548 |
| 辅助机构 | `TurnOffAlarm` | 停止警铃 | TurnOffAlarm | 578 |
| 辅助机构 | `TurnOnDCS` | 开始DCS | TurnOnDCS | 605 |
| 辅助机构 | `TurnOffDCS` | 停止DCS | TurnOffDCS | 635 |
| 辅助机构 | `SetReclaimRotationSpeedModePID` | 取料回转速度-PID调速 | SetReclaimRotationSpeedModePID | 661 |
| 辅助机构 | `SetReclaimRotationSpeedModeByAngle` | 取料回转速度-角度调速 | SetReclaimRotationSpeedModeByAngle | 696 |
| 辅助机构 | `SetReclaimRotationSpeedModeFixed` | 取料回转速度-固定速度 | SetReclaimRotationSpeedModeFixed | 731 |
| 辅助机构 | `TurnOnDrainage` | 排水启动 | TurnOnDrainage | 766 |
| 辅助机构 | `TurnOffDrainage` | 排水停止 | TurnOffDrainage | 798 |
| 辅助机构 | `TurnOnAirCannon` | 空气炮启动 | TurnOnAirCannon | 826 |
| 辅助机构 | `TurnOffAirCannon` | 空气炮停止 | TurnOffAirCannon | 858 |
| 辅助机构 | `TurnOnRinse` | 冲洗启动 | TurnOnRinse | 886 |
| 辅助机构 | `TurnOffRinse` | 冲洗停止 | TurnOffRinse | 918 |
| 润滑系统 | `TurnOnTravelingLubrication` | 走行润滑启动 | TurnOnTravelingLubrication | 945 |
| 润滑系统 | `TurnOffTravelingLubrication` | 走行润滑停止 | TurnOffTravelingLubrication | 977 |
| 润滑系统 | `TurnOnSlewingLubrication` | 回转润滑启动 | TurnOnSlewingLubrication | 1005 |
| 润滑系统 | `TurnOffSlewingLubrication` | 回转润滑停止 | TurnOffSlewingLubrication | 1037 |
| 润滑系统 | `TurnOnBoomLubrication` | 悬臂润滑启动 | TurnOnBoomLubrication | 1065 |
| 润滑系统 | `TurnOffBoomLubrication` | 悬臂润滑停止 | TurnOffBoomLubrication | 1097 |
| 润滑系统 | `TurnOnTailCarLubrication` | 尾车润滑启动 | TurnOnTailCarLubrication | 1125 |
| 润滑系统 | `TurnOffTailCarLubrication` | 尾车润滑停止 | TurnOffTailCarLubrication | 1157 |
| 堆料流程 | `SetStackRotateSpeedSetting` | 堆料回转速度设定 | SetStackRotateSpeedSetting | 1183 |
| 堆料流程 | `SetStackRotateDirection` | 堆料回转方向 | SetStackRotateDirection | 1231 |
| 辅助机构 | `SetArmBelt1` | 悬臂皮带堆料正转 | SetArmBelt1 | 1280 |
| 辅助机构 | `SetArmBelt2` | 悬臂皮带取料反转 | SetArmBelt2 | 1319 |
| 辅助机构 | `SetTailBelt` | 中继皮带堆料 | SetTailBelt | 1359 |
| 辅助机构 | `SetWheelClamp` | 夹轮器 | SetWheelClamp | 1432 |
| 辅助机构 | `SetTailCar` | 尾车机构 | SetTailCar | 1522 |
| 辅助机构 | `SetCouplingHook` | 摘挂钩机构 | SetCouplingHook | 1572 |
| 辅助机构 | `SetForkedHopper` | 分叉漏斗 | SetForkedHopper | 1671 |
| 挡板 | `SetDamper1` | 斗轮挡板 | SetDamper1 | 1721 |
| 挡板 | `SetDamper2` | 悬臂挡板 | SetDamper2 | 1771 |
| 挡板 | `SetDamper3` | 斗轮格栅板 | SetDamper3 | 1820 |
| 挡板 | `SetDamper4` | 设置挡板4 | SetDamper4 | 1867 |
| 挡板 | `SetDamper5` | 设置挡板5 | SetDamper5 | 1915 |
| 挡板 | `SetDamper6` | 设置挡板6 | SetDamper6 | 1962 |
| 挡板 | `SetDamper7` | 设置挡板7 | SetDamper7 | 2010 |
| 挡板 | `SetDamper8` | 设置挡板8 | SetDamper8 | 2058 |
| 挡板 | `SetDamper9` | 设置挡板9 | SetDamper9 | 2106 |
| 辅助机构 | `SetTravelAnchor` | 走行锚定 | SetTravelAnchor | 2153 |
| 辅助机构 | `SetBoomAnchor` | 悬臂锚定 | SetBoomAnchor | 2200 |
| 辅助机构 | `SetDryFogDustSuppression` | 干雾除尘 | SetDryFogDustSuppression | 2248 |
| 辅助机构 | `SetLubrication` | 润滑 | SetLubrication | 2295 |
| 手动 | `WalkTargetAciton` | 目标走行 | WalkTargetAciton | 2894 |
| 手动 | `WalkDistanceAciton` | 距离走行 | WalkDistanceAciton | 2928 |
| 手动 | `RotateTargetAciton` | 回转绝对 | RotateTargetAciton | 2961 |
| 手动 | `RotateDistanceAciton` | 回转相对 | RotateDistanceAciton | 2994 |
| 手动 | `PitchTargetAciton` | 俯仰绝对 | PitchTargetAciton | 3027 |
| 手动 | `PitchDistanceAciton` | 俯仰相对 | PitchDistanceAciton | 3060 |
| 手动 | `WalkTargetStop` | 目标走行停止 | WalkTargetStop | 3093 |
| 手动 | `WalkDistanceStop` | 走行距离停止 | WalkDistanceStop | 3121 |
| 手动 | `RotateTargetStop` | 回转绝对停止 | RotateTargetStop | 3148 |
| 手动 | `RotateDistanceStop` | 回转相对停止 | RotateDistanceStop | 3175 |
| 手动 | `PitchTargetStop` | 俯仰绝对停止 | PitchTargetStop | 3202 |
| 手动 | `PitchDistanceStop` | 俯仰相对停止 | PitchDistanceStop | 3229 |
| 自动堆料 | `StackPause` | 堆料暂停 | StackPause | 3259 |
| 自动堆料 | `ClearStackAccumulated` | 累计堆料量清空 | ClearStackAccumulated | 3298 |
| 自动堆料 | `StackFaultReset` | 堆料故障复位 | StackFaultReset | 3331 |
| 自动堆料 | `DownLoadStackParams` | 仅下发堆料参数到PLC | DownLoadStackParams | 3412 |
| 自动堆料 | `AutoStackPointCommand` | 堆料策略点位下发 | AutoStackPointCommand | 3450 |
| 自动堆料 | `WalkAlignmentCommand` | 走行对位 | WalkAlignmentCommand | 3738 |
| 自动堆料 | `RotationAlignmentCommand` | 回转对位 | RotationAlignmentCommand | 3767 |
| 自动堆料 | `PitchAlignmentCommand` | 俯仰对位 | PitchAlignmentCommand | 3796 |
| 自动堆料 | `AutoStackBeginCommand` | 开始自动堆料 | AutoStackBeginCommand | 3826 |
| 流程 | `OneClickSwitchStackMode` | 堆料模式一键切换 | OneClickSwitchStackMode | 3856 |
| 自动堆料 | `ChangeBlankPoint` | 变更落料点序号 | ChangeBlankPoint | 3885 |
| 自动堆料 | `AutoStackBeginRightCommand` | 开始自动堆料-向右 | AutoStackBeginRightCommand | 3914 |
| 自动堆料 | `AutoStackBeginLeftCommand` | 开始自动堆料-向左 | AutoStackBeginLeftCommand | 3948 |
| 自动堆料 | `AutoStackOneKeyStartCommand` | 堆料一键启动 | AutoStackOneKeyStartCommand | 3982 |
| 自动堆料 | `AutoStackStrategyFixedPointCommand` | 策略后下发定点堆 | AutoStackStrategyFixedPointCommand | 4016 |
| 自动堆料 | `AutoStackStrategySlewLeftBackwardCommand` | 策略后下发回转堆左转（后退） | AutoStackStrategySlewLeftBackwardCommand | 4050 |
| 自动堆料 | `AutoStackStrategySlewRightBackwardCommand` | 策略后下发回转堆右转（后退） | AutoStackStrategySlewRightBackwardCommand | 4085 |
| 自动堆料 | `AutoStackStrategySlewLeftForwardCommand` | 策略后下发回转堆左转（前进） | AutoStackStrategySlewLeftForwardCommand | 4120 |
| 自动堆料 | `AutoStackStrategySlewRightForwardCommand` | 策略后下发回转堆右转（前进） | AutoStackStrategySlewRightForwardCommand | 4155 |
| 自动堆料 | `WalkAlignmentStopCommand` | 走行对位停止 | WalkAlignmentStopCommand | 4191 |
| 自动堆料 | `RotationAlignmentStopCommand` | 回转对位停止 | RotationAlignmentStopCommand | 4218 |
| 自动堆料 | `PitchAlignmentStopCommand` | 俯仰对位停止 | PitchAlignmentStopCommand | 4245 |
| 自动堆料 | `AutoStackBeginCommandStop` | 远程自动堆料停止 | AutoStackBeginCommandStop | 4271 |
| 自动堆料 | `StopChangeBlankPoint` | 停止变更落料点序号 | StopChangeBlankPoint | 4303 |
| 自动堆料 | `ChangeBlankDistance` | 变更落料口距离 | ChangeBlankDistance | 4328 |
| 自动堆料 | `ChangeXDistance` | 变更X轴点间距（落料点间距） | ChangeXDistance | 4358 |
| 自动堆料 | `ChangeLeftBoundary` | 变更左边界 | ChangeLeftBoundary | 4389 |
| 自动堆料 | `ChangeRightBoundary` | 变更右边界 | ChangeRightBoundary | 4420 |
| 自动堆料 | `ChangeLeftBoundaryStayTime` | 变更左边界停留时间 | ChangeLeftBoundaryStayTime | 4451 |
| 自动堆料 | `ChangeRightBoundaryStayTime` | 变更右边界停留时间 | ChangeRightBoundaryStayTime | 4482 |
| 自动取料 | `ScanRotateBoundaryTarget` | 扫描切入切出设置 | ScanRotateBoundaryTarget | 4519 |
| 自动取料 | `RotateBoundaryTarget` | 切入切出设置 | RotateBoundaryTarget | 4577 |
| 自动取料 | `PickClosedAngleBucketLoad` | 取料近角斗轮负载值 | PickClosedAngleBucketLoad | 4614 |
| 自动取料 | `PickFarAngleBucketLoad` | 取料远角斗轮负载值 | PickFarAngleBucketLoad | 4642 |
| 自动取料 | `HalfAutoPickCommand` | 半自动取料参数下发 | HalfAutoPickCommand | 4718 |
| 自动取料 | `AutoPickPointCommand` | 取料策略点下发 | AutoPickPointCommand | 4759 |
| 自动取料 | `AutoPickExploreCommand` | 开始探摸取料 | AutoPickExploreCommand | 4920 |
| 自动取料 | `AutoPickBeginCommand` | 开始自动取料 | AutoPickBeginCommand | 4947 |
| 自动取料 | `AutoPickExploreCommandStop` | 探摸取料停止 | AutoPickExploreCommandStop | 4981 |
| 自动取料 | `AutoPickBeginCommandStop` | 远程自动取料停止 | AutoPickBeginCommandStop | 5008 |
| 自动取料 | `RetrievalPause` | 取料暂停 | RetrievalPause | 5040 |
| 自动取料 | `ReclaimEndCurrentLoop` | 结束当前循环 | ReclaimEndCurrentLoop | 5080 |
| 自动堆料 | `StackEndCurrentLoop` | 结束当前循环 | StackEndCurrentLoop | 5106 |
| 自动取料 | `PickFaultReset` | 取料故障复位 | PickFaultReset | 5132 |
| 自动取料 | `ChangeLayers` | 取料换层 | ChangeLayers | 5158 |
| 自动取料 | `StopChangeLayers` | 取料换层停止 | StopChangeLayers | 5201 |
| 自动取料 | `ChangeCurrentLayer` | 修改当前作业层号 | ChangeCurrentLayer | 5243 |
| 自动取料 | `ChangeMaterialAmount` | 修改取料量 | ChangeMaterialAmount | 5271 |
| 自动取料 | `ChangeDefaultMaterialFlow` | 修改预设料流 | ChangeDefaultMaterialFlow | 5299 |
| 自动取料 | `ChangeFootage` | 修改进尺量 | ChangeFootage | 5327 |
| 自动取料 | `ChangeAlignSafetyDistance` | 修改对位安全距离 | ChangeAlignSafetyDistance | 5355 |
| 自动取料 | `ChangeReclaimFixedRotationSpeed` | 取料回转固定速度 | ChangeReclaimFixedRotationSpeed | 5384 |
| 自动取料 | `ClearPickAccumulated` | 累计取料量清空 | ClearPickAccumulated | 5416 |
| 自动取料 | `ReclaimBeginRightCommand` | 取料开始向右 | ReclaimBeginRightCommand | 5442 |
| 自动取料 | `ReclaimBeginLeftCommand` | 取料开始向左 | ReclaimBeginLeftCommand | 5470 |
| 自动取料 | `ReclaimLayerChangeDownCommand` | 取料换层向下 | ReclaimLayerChangeDownCommand | 5499 |
| 自动取料 | `ReclaimLayerChangeUpCommand` | 取料换层向上 | ReclaimLayerChangeUpCommand | 5528 |
| 自动取料 | `ReclaimModeOneKeySwitchCommand` | 取料模式一键切换 | ReclaimModeOneKeySwitchCommand | 5557 |
| 自动取料 | `ReclaimOneKeyStartCommand` | 取料一键启动 | ReclaimOneKeyStartCommand | 5586 |
| 自动取料 | `ReclaimStrategyInitialLeftLayerUpCommand` | 策略后下发初始左、换层上 | ReclaimStrategyInitialLeftLayerUpCommand | 5615 |
| 自动取料 | `ReclaimStrategyInitialRightLayerUpCommand` | 策略后下发初始右、换层上 | ReclaimStrategyInitialRightLayerUpCommand | 5645 |
| 自动取料 | `ReclaimStrategyInitialLeftLayerDownCommand` | 策略后下发初始左、换层下 | ReclaimStrategyInitialLeftLayerDownCommand | 5675 |
| 自动取料 | `ReclaimStrategyInitialRightLayerDownCommand` | 策略后下发初始右、换层下 | ReclaimStrategyInitialRightLayerDownCommand | 5705 |
| 单机 | `SetWalkAdvanceCommand` | 走行前进速度设定 | SetWalkAdvanceCommand | 5738 |
| 单机 | `SetWalkBackCommand` | 走行后退速度设定 | SetWalkBackCommand | 5767 |
| 单机 | `SetRotateRightCommand` | 回转右转速度设定 | SetRotateRightCommand | 5796 |
| 单机 | `SetRotateLeftCommand` | 回转左转速度设定 | SetRotateLeftCommand | 5825 |
| 单机 | `SetPitchUpCommand` | 俯仰上升速度设定 | SetPitchUpCommand | 5854 |
| 单机 | `SetPitchDownCommand` | 俯仰下降速度设定 | SetPitchDownCommand | 5883 |
| 流程 | `OneClickSwitchReclaimMode` | 取料模式一键切换 | OneClickSwitchReclaimMode | 5914 |
| 流程 | `OneClickSwitchReclaimModeStop` | 取料模式一键切换停止 | OneClickSwitchReclaimModeStop | 5943 |
| 流程 | `OneClickReclaimJobStop` | 取料作业一键停止 | OneClickReclaimJobStop | 5970 |
| 流程 | `OneClickSwitchStackModeStop` | 堆料模式一键切换停止 | OneClickSwitchStackModeStop | 5999 |
| 流程 | `OneClickStackJobStop` | 堆料作业一键停止 | OneClickStackJobStop | 6025 |

<a name="reading"></a>

## 学习验收与联调排查

### 一、能否从一条命令倒查全部证据？

使用教学 ID=42（不向设备发送），解释 Web 请求的 equipmentCode/value，后端生成的真实 long ID，CMD 主题、Value 类型，边缘 Cmd 注册名，PLC GuideByte/ModeCheck/Affiliation/ActionCommandCode，以及回执如何映射回前端。必须说明业务 ID、MQTT Packet Identifier、PLC ActionControlID、StrategyVerifyID、VerifyID 的不同用途。当前 QoS 0 的 PUBLISH 没有 QoS1/2 的 Packet Identifier 确认流程。

### 二、若界面不更新，按边界分段排查

1. 核对 PLC 状态到帧、长度头、状态体长度、字节序与 VerifyID。
2. 检查 CatchData、AllData.MainData.ID、STATE 主题与有效值。
3. 检查 Broker 上实际发布、客户端连接与后端配置订阅；注意 localhost 的机器含义。
4. 检查 EventSourceStorer 的 Channel 消费、MachineStateData 与设备编码查找。
5. 检查 SignalR AllData_设备号、前端设备选择、事件订阅与 mock 模式。

### 三、若 HTTP 显示成功而设备没动作

先判断接口是否等待边缘回执；再查命令表/业务 ID、CMD 发布、边缘队列积压、注册匹配、参数转换、TCP 2002 发送结果与 PLC 模式/联锁。最后核对 RETURN 的确认层级以及机构真实运行/完成反馈。不要把重发当作通用修复，必须先确认设备动作幂等性与操作授权。

### 四、建议的隔离台架用例与预期

| 用例 | 应核对内容 |
| --- | --- |
| 半包 / 两帧粘包 | 不丢帧、不多消费；长度头与缓冲区一致 |
| 非法 JSON / null / 非法数值 | 拒绝并回错误，不产生默认零动作 |
| 重复业务 ID | 同一操作不会重复执行；记录去重结果 |
| 策略传到中段断线 | 明确失败段、PLC 已接收状态与恢复策略 |
| 故障位 1→0 | 活动告警清空并正确通知恢复 |
| PLC 单通道断线 | 状态、命令、故障三条连接分别恢复 |
| 停止命令遇到策略长等待 | 测量排队与实际控制时延，并由安全规范判定 |

### 五、模式和工程量不要按名称猜测

SwitchMode 中自动取料设置 IsReclaimMode=false、IsManualMode=true；自动堆料设置 IsReclaimMode=true、IsManualMode=true。这里的布尔字段是协议位值，不能根据英文变量名把 true 直接解释成“手动”或“取料”。上报走行位置除以10，回转/俯仰位置除以100；手动目标命令和策略点的比例则按各命令代码处理，不能统一假设全部除以100。MainData.Pause 在走行、回转、俯仰三个映射区重复赋值，最后结果受俯仰状态覆盖，也需结合前端含义确认。

### 六、三个线程边界

MQTT 消息回调负责入队；BackgroundService 串行 await 消费；TCP 状态与故障是两条异步接收循环。周期发送运行于 Task.Run，脉冲复位由 Timer 回调完成。RealTimeControl 有 _dataLock；_allData 状态对象和普通主机回执槽位没有同样完整的跨通道快照保护。ConcurrentQueue 不会自动保护其他共享对象，也不提供事务。

### 七、术语速查

Broker：MQTT 消息路由服务器；Topic：主题路由键；Payload：负载；DTO：跨边界数据模型；DI：依赖注入；ACK：确认；位域：一个整数中用不同位表示开关；大小端：多字节数值在线路上的字节顺序；粘包/半包：TCP 是字节流而不是报文边界，因此应用需要自行拆帧。

<a name="config"></a>

## 当前配置快照（不是部署建议）

来自 App.config；地址和设备号仅描述当前源码配置。切勿直接据此连接生产 PLC。Config_File_Path 与 LoadTime 等字段的使用边界见正文。

| 配置键 | 值 |
| --- | --- |
| Config_File_Path | `Resource/Data.xlsx` |
| PLC_IP | `192.168.10.118` |
| PLC_Cmd_Port | `2002` |
| PLC_Ctrl_Port | `2001` |
| PLC_Error_Port | `2004` |
| PLC_BigOrLittleEndian | `0` |
| MQTT_ID | `EdgeC3` |
| MQTT_IP | `127.0.0.1` |
| MQTT_Port | `1883` |
| LoadTime | `500` |
| MQTT_STATE_TOPIC | `/MACHINE/STATE/C3` |
| MQTT_RETURN_TOPIC | `/MACHINE/RETURN/C3` |
| MQTT_CMD_TOPIC | `/MACHINE/CMD/C3` |
| HeartBeat_ID | `C3` |
| HeartBeat_Topic | `/HeartBeat/Edge/C3` |
| MMWData_Topic | `/Data/MMWave/C3` |

<a name="probe"></a>

## 隔离源码探针原始输出

直接链接 Protocol、Enum、Model/PLC、ReturnCommandModel、PlcToEdgeService 与文档标记源码；使用 .NET 9 编译，网络/配置/日志/JSON 为最小替身。探针不启动项目 Host，不连接 Broker 或 PLC。以下结果不是现场抓包，也不是整套 net8.0 工程测试。

### 小端配置

```text
Offline source probe; .NET 9; real protocol + PlcToEdgeService sources; transport/log/JSON/config stubs. No network.
Sizes MainData=218, RealTimeControl=162, Action=18, Stack=250, Reclaim=60
Endian=0; ActionFrame=1200030001000000010003007B0000000000
ControlFirst8=A200020001000100
197 bytes rejected: PASS
198 bytes padded: PASS; VerifyID=0
Fault nonzero count=1
Fault all-zero count=1 (stale fault reproduced if 1)
MMW raw=123; mapped=1; expected engineering value=1.23
Return count=1; payload={"ID":42,"Cmd":"WalkTargetAciton","Result":null,"CommandState":4,"Time":"2026-08-28T21:25:29.8457286+08:00","LifeCycle":0}; incoming ActionCommandID=0
```

### 大端配置

```text
Offline source probe; .NET 9; real protocol + PlcToEdgeService sources; transport/log/JSON/config stubs. No network.
Sizes MainData=218, RealTimeControl=162, Action=18, Stack=250, Reclaim=60
Endian=1; ActionFrame=001200030001000000010003007B00000000
ControlFirst8=00A2000200010100
197 bytes rejected: PASS
198 bytes padded: PASS; VerifyID=0
Fault nonzero count=1
Fault all-zero count=1 (stale fault reproduced if 1)
MMW raw=123; mapped=1; expected engineering value=1.23
Return count=1; payload={"ID":42,"Cmd":"WalkTargetAciton","Result":null,"CommandState":4,"Time":"2026-08-28T21:25:30.4329549+08:00","LifeCycle":0}; incoming ActionCommandID=0
```

<a name="hashes"></a>

## 源码快照与文件指纹

工作区没有提供可用 Git 提交号。本清单使用 SHA-256 标识本次阅读的源码文件版本。生成日期 2026-08-28。后续文件变化后应重新核对行号和结论。

**展开全部文件指纹**

```text
c7b65a7874b5cdf1c5fdf270900d1c8c8ab6e7f03bc19267d40c338e1d041f74  RequireDocumentation/DocumentationRequirementAnalyzer.cs
91355fe79959a0d3c812866f5ae1f2afb5326d444ac7c363ea993b9299602ca1  RequireDocumentation/FieldGroupAnalyzer.cs
da335ebb54806e49c910243c780e7aeec0c5f634dfa62c66e49b19051af6de97  RequireDocumentation/FieldGroupAttribute.cs
74da1e17e0ec0567ad9e954855c0c9c56b660e83524f35c42bd29dfddecb6553  RequireDocumentation/RequireDocumentation.csproj
6264e1f213aa8d83c38881081efc55b503f61d7e4655bd36706529ce7794164c  RequireDocumentation/RequireDocumentationAttribute .cs
62149415f2d817e2069e07292cfbc86b54ec279ae2509d27cf4d46d1a0513b1e  SREdgeCompute_TCP/App.config
2dcd02b7889f9a53949ac916eddb57eda493d157be8f9ab97af0464f79698da2  SREdgeCompute_TCP/Client/MessageEventArgs.cs
200ed65e9da36f10e4ba0b1a59f49b4476eaec331b75ddfe2c197cfabfb692f7  SREdgeCompute_TCP/Client/MqttClient.cs
3685b7b778877216e7e085b229bc3ac8f477ae7d9d44ca9ee9eaee3a08e2270d  SREdgeCompute_TCP/Client/TCPClient.cs
ed4a91f4efd67e94a1861c7a47605a8ae309b64e7c915636518642300b134c06  SREdgeCompute_TCP/Enum/CommandAffilationEnum.cs
ac8e063d9a820190c0f5eacfc4fa7c21342ebd096d0f0ce22bd85d0819ea9485  SREdgeCompute_TCP/Enum/CommandStatusEnum.cs
49c9a5e61ea0eaa8ccb2f97b0dd128140aa65ecfb2f908db752b3062999559d5  SREdgeCompute_TCP/Enum/PointsStatusEnum.cs
353a6527f4099c7d6986d23d4596122734c5cc4ea535e03c39226d3a27e93834  SREdgeCompute_TCP/Infrastructure/Commands/CommandAttribute.cs
1b09fc3697f7c5f4d502564284d7ce94b6bd8f45dff5126cbc0c918ca4952cab  SREdgeCompute_TCP/Infrastructure/Commands/CommandInfo.cs
47c71a02e87f30193249125807e75487b879f4d4e9711d1c34bbc0e4df516380  SREdgeCompute_TCP/Infrastructure/Commands/CommandRegistry.cs
8451f4f56fc3aa3930f88dc7c108110bcaa126e0b38a1a348c8006ecdd9c9e33  SREdgeCompute_TCP/Model/Auto/AutoPick.cs
2cffa2c4a40572973cc4e9603b0c68631385e4030a7da7152ba1b13cf5478396  SREdgeCompute_TCP/Model/Auto/AutoStack.cs
83561853ae3f4fe8daa464e7b9212c4d3b2e73fcd80f6e44c6983679b6a572e6  SREdgeCompute_TCP/Model/Auto/OutOfPileAngleCommandModel.cs
1b74bb693d3665534144f917f8ad50ee4fb5facd1e3dfb039b40aa19e3b74ece  SREdgeCompute_TCP/Model/Auto/Point.cs
79ca0f2d5533d872bc1297c159176e3d0e7f913e68679fc1990b7644bc078e14  SREdgeCompute_TCP/Model/Auto/ReclaimParamsModel.cs
8bc0bd98898eb92d858cbedeef374bf349759e98add83a8603925ede59e3100b  SREdgeCompute_TCP/Model/Auto/StackParamsModel.cs
f52383b7e15f228f5bfd1b559436242857367ced111dafe967fe84e33305690b  SREdgeCompute_TCP/Model/Command/CommandModel.cs
16fa326292ce014ce244ab845a813224cae4dddd2038099ba8817460cdfa31bf  SREdgeCompute_TCP/Model/Command/MMWDataModel.cs
f6d359b31891e05fd7c79df7afa6d7f47de76fc57f7a60523b35fb649bde983a  SREdgeCompute_TCP/Model/Command/ReturnCommandModel.cs
89d06283cb0418727e7bb109f95cf0b476ce3e4c9bf6cb6921eed49af415b65a  SREdgeCompute_TCP/Model/Common/DynamicModel.cs
554d3ee60d516d3891ec3236677945406777cc678f90dac7c7dbe42ae106fd17  SREdgeCompute_TCP/Model/PLC/AllDataModel.cs
d7375e9dd3fecc35fa3ea89b40502843e500f1bb4f5c60fb31d1f3093cb11084  SREdgeCompute_TCP/Model/PLC/AuxiliaryInstitutionsDataModel.cs
b7879b09f368e3d8a26f23d62fa1f733adbd96bc3e815c2c3d3b9fa5ef8ca2f7  SREdgeCompute_TCP/Model/PLC/MainDataModel.cs
f184d7ffade7474f4960749d782a58194bd5f1f80bf295d26d2a9423182cbc02  SREdgeCompute_TCP/Model/PLC/MillimeterWaveModel.cs
db71edab970557ec25fc74484e047a59c1dde04a1d639414c0fc4832ad4078ff  SREdgeCompute_TCP/Model/PLC/ReclaimDataModel.cs
27cf483fc09a8d06fb952d076bb2c2f34e78f71e848908675fd1670ab87938e7  SREdgeCompute_TCP/Model/PLC/StackDataModel.cs
f8ba905a54490462b47f63619f757739a02938edf873a39828b0be37a86aa65a  SREdgeCompute_TCP/Model/PLC/StackPointModel.cs
a8de97cbe7c3bb86129c97f1d582fc62815b357507a8ec0db26566763fc6c74d  SREdgeCompute_TCP/Model/State/MainData.cs
7702fbe397b69e2b908e9e25839b05320a971862abe4762c54b9e9872d8d7dac  SREdgeCompute_TCP/Program.cs
1036affe4d58f96a8cbcd9cc429f8d68902b1f6f5589086f2d79ff67f2302cf8  SREdgeCompute_TCP/Properties/PublishProfiles/FolderProfile.pubxml
ce79cb10d225dfefc42d67b3d8e5833ae0b4321c9903ff542b2f1de9cd9eb372  SREdgeCompute_TCP/Properties/PublishProfiles/FolderProfile.pubxml.user
0ddca7b7905240c5d132cfeeac3c41cc44a35ed63331f9f948891b5d0170449e  SREdgeCompute_TCP/Protocol/ActionProcessControl.cs
ccf1943003e37fa7a9447aca1a94b15267ec53512cf421af9d9bf0c339989eae  SREdgeCompute_TCP/Protocol/GuideByteCompute.cs
39f6b39b7df21a0b1601850556e6fc36159b12c5c69611fff5b6dd85a02d9a76  SREdgeCompute_TCP/Protocol/MainData.cs
faeb502c190623e04bad5364b86a59bff17f3fac7b24384a32ac612e40b1930f  SREdgeCompute_TCP/Protocol/RealTimeControl.cs
7aaafd416ac62fd850e858e3a542788ad339f02d9265ee0b56b359762181aa91  SREdgeCompute_TCP/Protocol/ReclaimStrategyProcessControl.cs
ae31e958ad59151536e371e8277d0990bcf316e0605b30fab767a7703d2ac875  SREdgeCompute_TCP/Protocol/StackStrategyProcessControl.cs
9ba5ed2c4b25c3c26fd0ff3dbed93fabb007f4f3583b310371571060609405cd  SREdgeCompute_TCP/Protocol/Utility.cs
eed8c140279872dcbe1b59b6b05eb98fa00141964c5441e32883ce3430957112  SREdgeCompute_TCP/Resource/CommandXml.xml
99a58e1b052584a10362545b6066de898da4ccaa00cefcdea1ede52fcb655ec9  SREdgeCompute_TCP/Service/CommandQueueBackgroundService.cs
0756739e69e716989a51174b7958da1911b5f8c5958be1b949ed1740a52bebd6  SREdgeCompute_TCP/Service/CommandQueueService.cs
75d1dc0b32eccf282c35dca0cbff0545f8bc80d0235bc64598909ec98b9db4d5  SREdgeCompute_TCP/Service/CommandService.cs
59b283d370a2b3802a4a72a6d0aaa7a043d1e45ed3e4ddbb3f9536f97100b6eb  SREdgeCompute_TCP/Service/ConfigService.cs
2633c4fcc3d5ddd9431f2e70170524ad19f1279d0796f1182460082d89205f9e  SREdgeCompute_TCP/Service/EdgeToPlcService.cs
4e88760c54e83b7d8426401483202bb18875b81645385af28807fa5685c90a79  SREdgeCompute_TCP/Service/PlcToEdgeService.cs
9fe9a90f3132f60612b38cede736c504856f744186df1e8c1aeab33677e4475d  SREdgeCompute_TCP/SREdgeCompute_TCP.csproj
70af0dd0478632a19da106f2c643e7bf2cd2c8eda707b96ddd5d89b37e30ea34  SREdgeCompute_TCP/SREdgeCompute_TCP.csproj.user
b3b01b2c5a9e47af9f005f71731df04e10fe13847c45451f0ba2edb683c02bbb  SREdgeCompute_TCP.sln
```
