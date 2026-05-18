# HMDroidbot 架构图

HMDroidbot 是一个 Python 实现的 HarmonyOS / Android UI 自动探索与测试输入生成器，核心模型是“命令行配置 -> DroidBot 调度 -> 输入策略生成事件 -> 设备适配层执行 -> 输出 UTG 和报告”。

## 总览架构

```mermaid
flowchart TB
    subgraph CLI["入口与配置"]
        Console["droidbot 命令\nsetup.py console_scripts"]
        Module["python -m droidbot.start\n或 droidbot/start.py"]
        RootStart["start.py\n直接脚本入口"]
        Config["config.yml\nutils 加载默认配置"]
    end

    subgraph Core["核心调度层"]
        Start["droidbot.start\nparse_args/main"]
        DroidBot["DroidBot\n单机调度器"]
        DroidMaster["DroidMaster\n分布式/QEMU 调度器"]
        WorkerConn["DroidBotConn\nWorker 子进程"]
        Env["AppEnvManager\n测试环境管理"]
        InputManager["InputManager\n事件循环管理"]
    end

    subgraph Model["领域模型"]
        App["App\nAPK 元数据/Intent"]
        AppHM["AppHM\nHAP/Bundle 元数据"]
        Device["Device\nAndroid 设备抽象"]
        DeviceHM["DeviceHM\nHarmonyOS 设备抽象"]
        Policy["InputPolicy\nRandom/DFS/BFS/Replay/Manual/Memory"]
        State["DeviceState\n界面状态与可触发事件"]
        Event["InputEvent/EventLog\n输入事件与执行日志"]
        UTG["UTG\nnetworkx UI 转移图"]
    end

    subgraph Adapter["设备适配层"]
        ADB["ADB"]
        HDC["HDC"]
        DroidbotApp["DroidBotAppConn\n伴随 APK/无障碍"]
        Minicap["Minicap\n截图"]
        Log["Logcat/Hilog\n设备日志"]
        Telnet["TelnetConsole\n模拟器控制"]
        Process["ProcessMonitor/UserInputMonitor/IME"]
        QEMUConn["QEMUConn"]
    end

    subgraph External["外部工具与服务"]
        Androguard["Androguard\nAPK 解析"]
        HostADB["adb/monkey"]
        HostHDC["hdc"]
        QEMU["QEMU/qemu-img"]
        Humanoid["Humanoid XML-RPC\n可选事件排序"]
        Frida["Frida Monitor\n可选敏感 API 监控"]
    end

    subgraph DeviceRuntime["设备运行时"]
        TargetApp["被测应用\nAPK/HAP"]
        Companion["DroidBot Companion APK\nAndroid 无障碍服务"]
        OSLogs["系统日志/页面/Ability/Activity"]
    end

    Console --> Start
    Module --> Start
    RootStart --> DroidBot
    Config --> Start
    Start --> DroidBot
    Start --> DroidMaster
    DroidBot --> Device
    DroidBot --> DeviceHM
    DroidBot --> App
    DroidBot --> AppHM
    DroidBot --> Env
    DroidBot --> InputManager
    InputManager --> Policy
    InputManager --> Event
    Policy --> State
    Policy --> UTG
    Policy --> Event
    Event --> Device
    Event --> DeviceHM
    App --> Androguard
    Device --> ADB
    Device --> DroidbotApp
    Device --> Minicap
    Device --> Log
    Device --> Telnet
    Device --> Process
    DeviceHM --> HDC
    DeviceHM --> Log
    ADB --> HostADB
    HDC --> HostHDC
    HostADB --> TargetApp
    HostHDC --> TargetApp
    DroidbotApp --> Companion
    Log --> OSLogs
    Minicap --> OSLogs
    DroidMaster --> QEMUConn
    DroidMaster --> WorkerConn
    QEMUConn --> QEMU
    WorkerConn --> DroidBot
    Policy -. 可选 .-> Humanoid
    State -. 可选 .-> Humanoid
    DroidBot -. 可选 .-> Frida
```

## 单机运行时序

```mermaid
sequenceDiagram
    autonumber
    participant User as 用户/CI
    participant Start as droidbot.start
    participant Bot as DroidBot
    participant Dev as Device/DeviceHM
    participant App as App/AppHM
    participant Env as AppEnvManager
    participant IM as InputManager
    participant Policy as InputPolicy
    participant UTG as UTG
    participant Out as output_dir 报告

    User->>Start: droidbot -a app -o output [-is_harmonyos]
    Start->>Start: 解析 CLI 并合并 config.yml
    Start->>Bot: 构造 DroidBot
    Bot->>Dev: 选择 Android Device 或 HarmonyOS DeviceHM
    Bot->>App: 解析 APK/HAP 或包名
    Bot->>Out: 复制报告 HTML/静态资源
    Bot->>Dev: set_up/connect
    Bot->>Dev: install_app(app)
    Bot->>Env: deploy()
    Bot->>IM: start()
    loop event_count 或直到停止
        IM->>Policy: generate_event()
        Policy->>Dev: get_current_state()
        Policy->>UTG: add_transition(last_event,last_state,current_state)
        Policy-->>IM: InputEvent
        IM->>Dev: EventLog.start 执行事件
        IM->>Dev: 按 interval 等待设备可继续发送事件
        IM->>Dev: EventLog.stop
        UTG->>Out: 输出 utg.js/截图/状态信息
    end
    Bot->>IM: stop()
    Bot->>Env: stop()
    Bot->>Dev: disconnect/tear_down
    Bot->>Dev: 按 keep_app 决定是否卸载
```

## 模块职责

| 模块 | 职责 |
| --- | --- |
| `droidbot/start.py` | 包装 CLI 参数、YAML 配置和启动模式，选择单机 `DroidBot` 或分布式 `DroidMaster`。 |
| `droidbot/droidbot.py` | 主协调器，负责输出目录、设备/应用模型、环境管理器和输入管理器的生命周期。 |
| `droidbot/droidmaster.py` | 分布式模式控制器，维护 QEMU 设备池，通过 `DroidBotConn` 启停 worker，并通过 XML-RPC 与 worker 协作。 |
| `droidbot/adapter/droidbot.py` | `DroidBotConn`：由 `DroidMaster` 调用的 worker 生命周期封装（`subprocess` 启动 `droidbot -distributed worker`）。架构上归入核心调度层，与 `DroidMaster` 同级。 |
| `droidbot/device.py` | Android 设备抽象，聚合 ADB、Minicap、Logcat、伴随 APK、IME、进程/输入监控等适配器。 |
| `droidbot/device_hm.py` | HarmonyOS 设备抽象，主要通过 HDC/Hilog 与设备通信。 |
| `droidbot/app.py` | 使用 Androguard 解析 APK 包名、Activity、权限、广播和文件哈希。 |
| `droidbot/app_hm.py` | 解析 HAP 包或通过 `bm dump` 读取已安装 HarmonyOS 包信息。 |
| `droidbot/input_manager.py` | 创建输入策略，维护事件列表、脚本输入、事件间隔和 Monkey/手动模式。 |
| `droidbot/input_policy.py` | 定义随机、DFS/BFS、回放、手动等输入策略，并把界面状态转化为下一步事件。 |
| `droidbot/input_policy2.py` | 扩展的 memory-guided 输入策略。 |
| `droidbot/utg.py` | 基于 `networkx.DiGraph` 维护 UI 转移图，并输出测试报告中的 UTG 数据。 |
| `droidbot/device_state.py` | 表示当前界面树、截图、页面/Activity 信息和可执行输入事件。 |
| `droidbot/input_event.py` | 定义触摸、按键、Intent、文本、杀进程等事件及事件执行日志。 |
| `droidbot/adapter/` | 封装 ADB、HDC、日志、截图、QEMU、DroidBot 伴随 APK 和 HarmonyOS driver 等设备/虚拟机侧协议（不含 `DroidBotConn`，见上）。 |
| `droidbot/resources/` | 报告页面、样式、前端脚本、伴随 APK 和 JS hook 资源。 |

## 关键数据流

```mermaid
flowchart LR
    Args["CLI 参数 / config.yml"] --> Options["运行选项"]
    Options --> Bot["DroidBot"]
    Bot --> AppMeta["App/AppHM 元数据\n包名/入口/权限/哈希"]
    Bot --> DeviceInfo["Device/DeviceHM\n屏幕/系统/连接状态"]
    Bot --> Manager["InputManager"]
    Manager --> Script["可选 JSON Script"]
    Manager --> Policy["InputPolicy"]
    Policy --> CurrentState["DeviceState\n视图树/截图/Activity/Page"]
    CurrentState --> CandidateEvents["候选 InputEvent"]
    CandidateEvents --> Policy
    Policy --> ChosenEvent["选中事件"]
    ChosenEvent --> EventLog["EventLog\n执行前后采样/ profiling"]
    EventLog --> Adapter["ADB/HDC/Companion/Minicap/Logcat/Hilog"]
    Adapter --> DeviceRuntime["设备运行时"]
    DeviceRuntime --> CurrentState
    CurrentState --> UTG["UTG\n状态节点/事件边"]
    UTG --> Report["output_dir\nindex.html/utg.js/截图/日志"]
```

## 平台差异

```mermaid
flowchart TB
    DroidBot["DroidBot(is_harmonyos)"]

    subgraph Android["Android 路径"]
        Device["Device"]
        App["App(APK)"]
        ADB["ADB"]
        Companion["DroidBotAppConn + 无障碍"]
        Minicap["Minicap"]
        Logcat["Logcat"]
        AndroidIntent["Intent\nam start/force-stop/broadcast"]
    end

    subgraph Harmony["HarmonyOS 路径"]
        DeviceHM["DeviceHM"]
        AppHM["AppHM(HAP/包名)"]
        HDC["HDC"]
        Hilog["Hilog"]
        AbilityIntent["Intent\naa -b bundle -a ability"]
    end

    DroidBot -- false --> Device
    DroidBot -- false --> App
    Device --> ADB
    Device --> Companion
    Device --> Minicap
    Device --> Logcat
    App --> AndroidIntent

    DroidBot -- true --> DeviceHM
    DroidBot -- true --> AppHM
    DeviceHM --> HDC
    DeviceHM --> Hilog
    AppHM --> AbilityIntent
```

## 分布式模式

```mermaid
flowchart TB
    CLI["droidbot -distributed master"] --> Master["DroidMaster"]
    Master --> Pool["设备池\n默认 6 个 slot"]
    Pool --> QEMUConn["QEMUConn"]
    Pool --> WorkerConn["DroidBotConn"]
    QEMUConn --> QEMU["QEMU/qemu-img\n镜像/快照"]
    WorkerConn --> Worker["DroidBot worker 子进程"]
    Master --> RPC["SimpleXMLRPCServer"]
    Worker -. master=http://host:port .-> RPC
    Worker --> Policy["UTG 输入策略"]
    Policy -. spawn/stop_worker .-> RPC
    RPC --> Master
    Master --> Output["共享 output_dir\nworker 初始化脚本/报告"]
```

## 备注

- 总览图中 `DroidBotConn` 画在核心调度层：它由 `DroidMaster` 直接持有，负责 worker 子进程启停，属于分布式调度基础设施；`QEMUConn` 仍放在设备适配层，对应虚拟机资源接入。二者源码均在 `droidbot/adapter/` 目录，分层按职责而非目录。
- 默认输入策略是 `dfs_greedy`，入口参数也支持 `none`、`monkey`、`random`、`dfs_naive`、`bfs_naive`、`bfs_greedy`、`replay`、`manual` 和 `memory_guided`。
- `input_manager.py` 引用了 `llm_guided` 对应的 `input_policy3.py`，但当前仓库未包含该文件，因此该策略在当前源码树中不可用。
- 项目没有内置数据库，主要持久化目标是 `output_dir` 中的报告资源、UTG、截图、日志和状态文件。
