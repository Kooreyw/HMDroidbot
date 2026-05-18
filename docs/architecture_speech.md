# HMDroidbot 架构演讲稿


## 开场

大家好，今天我介绍 HMDroidbot 的UI测试架构。

HMDroidbot 是一个面向 HarmonyOS 和 Android 的轻量级 UI **自动探索**与**测试输入**生成器。它的目标不是简单地随机点击界面，而是在测试过程中持续观察应用状态，选择下一步输入事件，并把界面状态之间的跳转关系沉淀为 UTG，也就是 UI Transition Graph。

从整体上看，这个项目的核心链路可以概括成一句话：**命令行和配置文件**提供运行参数，`DroidBot` 负责统一调度，**输入策略**生成测试事件，**设备适配层**把事件发送到 Android 或 HarmonyOS 设备，最后在输出目录中生成**截图、日志、状态和 UTG 报告**。

接下来我会按五个部分展开：第一是总体架构，第二是单机运行流程，第三是关键模块职责，第四是 Android 和 HarmonyOS 的平台差异，第五是分布式模式。

## 第一部分：总体架构

请看第一张总览架构图。整个系统可以分成六层。

最上面是入口与配置层。用户可以通过 `droidbot` 命令、`python -m droidbot.start`，或者直接运行脚本进入系统。`config.yml` 用来补充默认配置，例如目标系统、设备序列号、应用路径、输出目录和事件数量。

第二层是核心调度层。`droidbot.start` 负责解析参数并决定启动模式。如果是普通模式，就创建 `DroidBot`；如果是分布式 master 模式，就创建 `DroidMaster`。`DroidBot` 是单机模式的核心协调器，它把设备、应用、**环境管理器**和**输入管理器**组合起来，控制完整生命周期。分布式模式下，`DroidMaster` 会通过 `DroidBotConn` 启停 worker 子进程；每个 worker 内部仍是完整的 `DroidBot` 探索链路。

第三层是领域模型层。这里包含 `App`、`AppHM`、`Device`、`DeviceHM`、`InputPolicy`、`DeviceState`、`InputEvent` 和 `UTG`。这些对象表达的是测试系统里的核心概念：被测应用是什么，设备处于什么状态，当前界面有哪些可操作控件，下一步应该执行什么事件，以及这些事件最终形成怎样的状态图。

第四层是设备适配层。它把上层的抽象操作转换成具体协议调用。Android 主要依赖 ADB、DroidBot 伴随 APK、Minicap、Logcat、Telnet 和输入法相关适配器；HarmonyOS 主要依赖 HDC 和 Hilog。【分布式场景下的 `QEMUConn` 也归在这一层；`DroidBotConn` 则画在核心调度层，因为它表达的是 Master 对 worker 的调度关系，而不是对真机 UI 的协议接入。】

第五层是外部工具与服务。比如 Androguard 用于 APK 解析，`adb` 和 `hdc` 用于设备通信，QEMU 用于分布式设备池，Humanoid 可以作为可选的 XML-RPC 服务**给输入事件排序**，Frida Monitor 可以用于敏感 API 监控。

最下面是设备运行时，也就是被测 APK 或 HAP、Android 伴随 APK 和系统日志、页面、Ability 或 Activity 信息。

所以，这套架构的特点是：核心调度逻辑保持在 Python 进程里，具体平台能力放到适配器层，**输入生成策略作为可替换策略存在**，最终所有运行证据都汇聚到输出目录。

## 第二部分：单机运行流程

接下来请看单机运行时序图。

一次普通运行从用户执行命令开始，比如 `droidbot -a app -o output -is_harmonyos`。入口模块会先解析 CLI 参数，再合并 `config.yml` 中的配置，然后构造 `DroidBot`。

`DroidBot` 初始化时会根据 `is_harmonyos` 选择不同的平台路径。如果目标是 Android，它创建 `Device` 和 `App`；如果目标是 HarmonyOS，它创建 `DeviceHM` 和 `AppHM`。与此同时，它还会初始化 `AppEnvManager` 和 `InputManager`，并把报告页面所需的 HTML、样式和脚本复制到输出目录。

启动阶段按照比较清晰的顺序执行。第一步是设备 `set_up`，准备底层适配器；第二步是 `connect`，建立设备连接并读取必要设备信息；第三步是安装或确认被测应用；第四步是部署测试环境；第五步进入输入管理器的事件循环。

deploy() 的含义是：在正式开始 UI 探索之前，向设备预置一套"测试环境数据"，让被测应用启动后能遇到真实数据而不是空设备。

**事件循环**是系统最关键的运行部分。`InputManager` 调用当前**输入策略**的 `generate_event`。策略会先从设备获取当前 `DeviceState`，也就是界面树、截图、Activity 或页面信息。然后策略把上一状态、上一事件和当前状态交给 `UTG`，更新状态图。接着策略根据当前状态选择一个 `InputEvent`，例如触摸、按键、启动 Intent、文本输入或者杀进程。

事件选中之后，`InputManager` 会通过 `EventLog` 执行事件。`EventLog` 负责事件前后的记录、可选 profiling，以及等待设备从暂停状态恢复。事件执行后，新的界面状态又会进入下一轮循环。这个循环一直持续到达到事件数量、超时、手动停止或策略中断。

停止阶段，`DroidBot` 会依次停止输入管理器和环境管理器，断开设备连接，按配置决定是否清理测试环境和卸载应用。

## 第三部分：关键模块职责

下面看模块职责表。

`droidbot/start.py` 是入口模块，它做的事情类似控制台前台：接收用户参数，识别运行模式，并把参数传给后面的调度器。

`droidbot/droidbot.py` 是单机核心。它不直接实现所有设备细节，而是负责把应用模型、设备模型、环境管理和输入管理组合成一次完整测试。

`droidbot/input_manager.py` 是事件循环管理器。它根据策略名称创建具体策略，也处理 JSON 脚本、事件间隔、Monkey 模式和手动模式。

`droidbot/input_policy.py` 是输入策略集合。默认策略是 `dfs_greedy`，此外还支持随机、DFS、BFS、回放、手动等模式。策略的共同目标是根据当前界面状态生成下一步事件，并通过 UTG 避免盲目重复。

`droidbot/utg.py` 是状态图核心。它使用 `networkx.DiGraph` 保存状态节点和事件边，并把状态图输出为报告页面可以读取的 `utg.js`。

`droidbot/device.py` 和 `droidbot/device_hm.py` 分别是 Android 与 HarmonyOS 的设备抽象。上层调用的是统一能力，比如启动应用、获取当前状态、执行事件；底层实际由 ADB 或 HDC 等适配器完成。

`droidbot/app.py` 和 `droidbot/app_hm.py` 则负责被测应用元数据。Android 通过 Androguard 解析 APK 的包名、主 Activity、权限和广播；HarmonyOS 解析 HAP 包，或者通过设备上的 `bm dump` 获取已安装包信息。

可以看到，项目把“调度”“策略”“状态图”“设备通信”和“应用元数据”拆成了相对独立的模块，这让新增策略或扩展平台能力时不需要改动所有层。

## 第四部分：关键数据流

接下来请看关键数据流图。

数据的起点是 CLI 参数和 `config.yml`。它们会形成一组运行选项，传入 `DroidBot`。

`DroidBot` 进一步派生出两类基础数据。一类是应用元数据，包括包名、入口、权限和哈希；另一类是设备信息，包括屏幕尺寸、连接状态和平台特征。

随后进入 `InputManager`。如果用户提供了 JSON Script，脚本会影响某些状态下的操作选择；否则主要由输入策略决定事件。输入策略每一轮都消费 `DeviceState`，获取界面树、截图、Activity 或 Page，再生成候选事件。候选事件经过策略选择后变成最终的 `InputEvent`。

最终事件会通过 `EventLog` 和适配器发送到真实设备。设备状态改变之后，系统再次采集新的 `DeviceState`，并把状态与事件写入 `UTG`。报告目录中的 `index.html`、`utg.js`、截图、日志和状态文件，就是这条数据流持续运行后的结果。

这也是 HMDroidbot 与普通随机测试工具的区别：它不仅发送输入，还持续构建“输入事件导致界面如何变化”的结构化证据。

## 第五部分：平台差异

接下来请看平台差异图。

Android 路径和 HarmonyOS 路径在核心调度层保持统一，都由 `DroidBot` 驱动，也都通过 `InputManager` 和输入策略产生事件。差异主要发生在应用模型、设备抽象和底层适配器。

在 Android 路径中，应用模型是 `App`，输入文件是 APK。`App` 借助 Androguard 读取包名、主 Activity、权限和广播，并生成 Android 风格的 Intent。设备模型是 `Device`，它聚合 ADB、DroidBot 伴随 APK、Minicap、Logcat、Telnet、进程监控、用户输入监控和 IME 等能力。

在 HarmonyOS 路径中，应用模型是 `AppHM`，输入可以是 HAP 文件或已安装包名。启动入口使用 bundle name 和 ability。设备模型是 `DeviceHM`，它主要通过 HDC 与设备交互，并用 Hilog 收集日志。

这种设计的好处是，上层输入策略可以尽量复用。策略关心的是当前状态和下一步事件，而不是事件最终通过 ADB 还是 HDC 发送。平台差异被限制在设备、应用和适配器层。

## 第六部分：分布式模式

最后看分布式模式。

分布式模式由 `DroidMaster` 负责。它维护一个 QEMU 设备池，默认有多个 slot。每个 slot 会关联一个 QEMU 实例和一个 DroidBot worker 子进程。

Master 通过设备适配层的 `QEMUConn` 管理虚拟设备镜像、端口和快照，通过核心调度层的 `DroidBotConn` 启动或停止 worker。Worker 本质上仍然是一个普通的 DroidBot，只是它启动时会带上 master 的 XML-RPC 地址。

当某个 worker 在探索过程中发现适合派生的新状态时，可以通过 XML-RPC 请求 master spawn 新 worker。Master 会基于当前 QEMU 快照创建新镜像和初始化脚本，再把任务分配给空闲设备 slot。

因此，分布式模式不是重写一套测试逻辑，而是在单机 DroidBot 外面增加设备池、worker 管理和远程协调能力。核心探索逻辑仍然复用输入策略、设备抽象和 UTG。

## 总结

总结一下，HMDroidbot 的架构可以用四个关键词概括。

第一是集中调度。`DroidBot` 统一管理设备、应用、环境和输入生命周期。

第二是策略驱动。输入生成由 `InputPolicy` 负责，支持随机、DFS、BFS、回放、手动和 memory-guided 等策略。

第三是适配器隔离。Android 和 HarmonyOS 的差异主要被封装在 `Device`、`DeviceHM`、`App`、`AppHM` 和 `adapter` 目录中。

第四是报告沉淀。每次测试不只产生操作结果，还会形成 UTG、截图、日志和状态文件，便于后续分析覆盖情况和页面跳转路径。

所以，如果后续要扩展这个项目，可以按目标选择切入点：新增探索算法主要改输入策略；增强设备能力主要改适配器；扩展报告主要看 `UTG` 和 `resources`；支持新的平台能力则优先从应用模型和设备抽象入手。

我的介绍到这里，谢谢大家。

## 配图提示

| 演讲段落 | 建议配图 |
| --- | --- |
| 开场与总体架构 | `architecture.md` 的“总览架构”图 |
| 单机运行流程 | `architecture.md` 的“单机运行时序”图 |
| 数据如何流动 | `architecture.md` 的“关键数据流”图 |
| Android 与 HarmonyOS 对比 | `architecture.md` 的“平台差异”图 |
| 分布式能力 | `architecture.md` 的“分布式模式”图 |
