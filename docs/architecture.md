# HMDroidbot 架构图

HMDroidbot 是一个 Python 实现的 HarmonyOS / Android UI 自动探索与测试输入生成器，核心模型是“命令行配置 -> DroidBot 调度 -> 输入策略生成事件 -> 设备适配层执行 -> 输出 UTG 和报告”。

## 总览架构

```mermaid
flowchart TB
    subgraph Entry["入口与配置"]
        CLI["droidbot / python -m droidbot.start\nconfig.yml"]
    end

    subgraph Orchestration["调度层"]
        Start["droidbot.start\n解析参数并选择模式"]
        DroidBot["DroidBot\n单机生命周期调度"]
        DroidMaster["DroidMaster\n分布式/QEMU 调度"]
    end

    subgraph Runtime["核心运行时"]
        AppModel["App / AppHM\n应用元数据"]
        DeviceModel["Device / DeviceHM\n设备抽象"]
        EnvManager["AppEnvManager\n环境部署"]
        InputLoop["InputManager\n事件循环"]
        Policy["InputPolicy\n事件生成策略"]
        StateGraph["DeviceState + UTG\n状态采集与转移图"]
    end

    subgraph IO["平台适配层"]
        AndroidIO["Android adapters\nADB / Companion / Minicap / Logcat"]
        HarmonyIO["HarmonyOS adapters\nHDC / Hilog"]
        DistributedIO["Distributed adapters\nQEMUConn / DroidBotConn"]
    end

    subgraph External["外部运行环境"]
        AndroidDevice["Android 设备\nAPK / adb / 无障碍"]
        HarmonyDevice["HarmonyOS 设备\nHAP / hdc / Ability"]
        OptionalServices["可选服务\nHumanoid / Frida"]
        Output["output_dir\nHTML / UTG / 截图 / 日志"]
    end

    CLI --> Start
    Start --> DroidBot
    Start --> DroidMaster
    DroidBot --> AppModel
    DroidBot --> DeviceModel
    DroidBot --> EnvManager
    DroidBot --> InputLoop
    InputLoop --> Policy
    Policy --> StateGraph
    StateGraph --> InputLoop
    InputLoop --> DeviceModel
    DeviceModel --> AndroidIO
    DeviceModel --> HarmonyIO
    DroidMaster --> DistributedIO
    DistributedIO --> DroidBot
    AndroidIO --> AndroidDevice
    HarmonyIO --> HarmonyDevice
    Policy -.-> OptionalServices
    DeviceModel -.-> OptionalServices
    DroidBot --> Output
    StateGraph --> Output
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
| `droidbot/droidmaster.py` | 分布式模式控制器，维护 QEMU 设备池，通过 XML-RPC 与 worker 协作。 |
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
| `droidbot/adapter/` | 封装 ADB、HDC、日志、截图、QEMU、DroidBot 伴随服务和 HarmonyOS driver 等底层协议。 |
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

- 默认输入策略是 `dfs_greedy`，入口参数也支持 `none`、`monkey`、`random`、`dfs_naive`、`bfs_naive`、`bfs_greedy`、`replay`、`manual` 和 `memory_guided`。
- `input_manager.py` 引用了 `llm_guided` 对应的 `input_policy3.py`，但当前仓库未包含该文件，因此该策略在当前源码树中不可用。
- 项目没有内置数据库，主要持久化目标是 `output_dir` 中的报告资源、UTG、截图、日志和状态文件。
