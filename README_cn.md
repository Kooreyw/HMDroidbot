## :memo: 关于
HMDroidbot（HM代表HarmonyOS，Droid代表Android）是一个轻量级的测试输入生成器，适用于HarmonyOS（兼容Android）。它是从[Droidbot](https://github.com/honeynet/droidbot) fork出来的，支持HarmonyOS NEXT设备。
它可以发送随机或脚本化的输入事件来测试HarmonyOS应用程序，更快地实现更高的代码覆盖率，并在测试后生成一个UI跳转关系图（UTG）。

## :smiling_face_with_three_hearts: 功能特性
:boom: 现已支持HarmonyOS的覆盖率报告！包括ability、pages等以及一些HarmonyOS特定的指标！请参见下面的示例报告！

![image](https://github.com/user-attachments/assets/1dfbb6f8-c9ab-48b2-8043-5474719a7466)

:boom: 支持Android和HarmonyOS NEXT设备。使用标志 `-is_harmonyos` 来指定目标系统。

:boom: 支持使用YMAL文件进行配置，简化启动方式。

:boom: 源代码改进。更易于阅读和调试。为源代码添加了类型注解并对日志进行了上色。使用 `-debug` 标志将调试级别日志打印到终端！

:boom: 使用 `-log` 标志从设备获取hilog。请在报告目录中查看！

## 前提条件

1. `Python 3.10 ~ 3.11`。一些必要的官方库在 Python 3.13 中被移除了，为了稳定运行，推荐使用 3.11。
2. `HDC cmdtool 3.1.0a+`。
3. HarmonyOS 测试需要一个 `.hap` 文件或已安装应用的 bundle name，并且目标真机或模拟器需要能被 `hdc list targets` 看到。

## 使用方法

1. **安装 HMDroidbot**

    Clone 此仓库并使用 `pip` 安装。

    :one: *（可选）* 你可以先设置一个虚拟环境，详情请见 Python 的 [venv 模块](https://realpython.com/python-virtual-environments-a-primer/)。

    **macOS 或 Linux 系统:**
    ```shell
    python3 -m venv droidenv
    source droidenv/bin/activate
    ```

    **Windows 系统:**
    ```powershell
    python3 -m venv droidenv
    .\droidenv\Scripts\activate
    ```

    :two: 以 editable mode 安装。

    ```shell
    git clone https://github.com/XixianLiang/HMDroidbot.git
    cd HMDroidbot
    pip install -e .
    ```

    :three: 如果安装成功，您应该能够执行 `droidbot -h`。如果无法运行 `droidbot` 命令，请尝试使用 `python3 -m droidbot.start -h`。

2. **快速开始（目前仅在 WSL 中可用）**

    :wave: 运行我们提供的 `run_sample.sh` 文件来下载示例 HAP 并尝试 HMDroidbot。
    ```bash
    bash run_sample.sh
    ```

    我们也提供了一些[用于测试的示例 HAP](https://github.com/XixianLiang/HarmonyOS_NEXT_apps)。

3. **配置 `config.yml`**

    HMDroidbot 启动前，当前工作目录必须存在 `config.yml` 或 `config.yaml`。启动流程会在解析命令行参数后读取该文件，HDC adapter 也会在 import 阶段读取其中的 `env`。

    必选参数是 `env`，用于选择宿主机上调用 HDC 的命令：
    ```yaml
    # config.yml
    env: Linux      # Linux/macOS 使用 hdc；windows 使用 hdc.exe
    ```

    常用命令行参数也可以放进 YAML。YAML 中的值会覆盖对应的命令行参数。

    | YAML key | 对应命令行参数 / 行为 |
    | --- | --- |
    | `system: harmonyOS` | 启用 HarmonyOS 模式，等价于 `-is_harmonyos`。 |
    | `app_path` | 目标 `.hap` 路径或已安装 bundle name；对应 `-a`。 |
    | `output_dir` | 报告输出目录；对应 `-o`。 |
    | `device`, `target`, `device_serial` | `hdc list targets` 中的目标设备；对应 `-t`。 |
    | `count` | 输入事件数量；对应 `-count`。 |
    | `policy` | 输入策略，例如 `dfs_greedy`、`bfs_greedy`、`random`、`manual`、`none`；对应 `-policy`。 |
    | `save_log: true` | 保存设备日志；对应 `-log`。 |

    HarmonyOS 配置示例：
    ```yaml
    env: macOS
    system: harmonyOS
    device: 23E**********1843
    output_dir: output
    app_path: app/sample.hap
    count: 1000
    policy: dfs_greedy
    save_log: true
    ```

4. **启动 HMDroidbot**

    使用上面的 YAML 配置后，可直接启动：
    ```bash
    droidbot
    # 或
    python -m droidbot.start
    ```

    也可以通过命令行传入主要参数：
    ```bash
    droidbot -a app/sample.hap -o output -t 23E**********1843 -count 1000 -is_harmonyos -debug
    python -m droidbot.start -a app/sample.hap -o output -t 23E**********1843 -count 1000 -is_harmonyos -debug
    ```

    注意事项：

    + 如果只连接了一台设备，HMDroidbot 会从 `hdc list targets` 自动填充 serial。如果没有设备或连接了多台设备，请在 YAML 中设置 `device`，或使用 `-t <device_serial>`。
    + HarmonyOS 模拟器在 `hdc list targets` 中通常显示为类似 `127.0.0.1:5555` 的本地回环地址，请通过 `-t 127.0.0.1:5555` 指定。模拟器需要在 DevEco Studio 中配置，具体参考[官方教程](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ide-emulator-create-V5)。
    + 使用 `-policy random` 或 `policy: random` 选择随机策略。单独的 `-random` 标志只是在支持的策略内部增加随机性。
    + 调试 HMDroidbot 源码时，`-debug` 很有用。
    + 使用 `-log` 或 `save_log: true` 可将 HarmonyOS hilog 保存到 `<output_dir>/hilog.txt`。
    + 您可以在 `droidbot -h` 中查看完整参数列表。

    **vscode `launch.json` 文件示例**

   <img width="1134" alt="image" src="https://github.com/user-attachments/assets/bffde3f3-deea-41fb-9087-fb7eb3772bd5">

## HarmonyOS 运行模型

当设置 `system: harmonyOS` 或 `-is_harmonyos` 后，HMDroidbot 会保留 DroidBot 的输入策略模型，同时切换到 HarmonyOS 专用的 app、device 和 UI adapter。

+ **HAP 元数据和生命周期**：`AppHM` 从 `.hap` 中的 `module.json` 和 `pack.info` 读取 bundle name、main ability、abilities 和文件哈希。`DeviceHM` 使用 `hdc install -r` 安装 HAP，通过 `bm dump -n <bundleName>` 刷新包信息，通过 `aa start` 启动应用，并通过 `aa force-stop` 停止应用。
+ **UI 层级采集**：`HDC.get_views()` 默认使用 `HmDriverDumper`。首次使用时会启动 `HmClient`，将与 CPU ABI 匹配的 `agent.so` 推送到 `/data/local/tmp/agent.so`，执行 `uitest start-daemon singleness`，转发端口 `8012`，并通过 Hypium 的 `hypiumApiHelper` 请求 `captureLayout`。
+ **视图归一化**：HarmonyOS layout node 会被转换成 DroidBot 风格的 view dict，因此已有输入策略和事件生成逻辑可以继续使用 `package`、`class`、`resource_id`、`bounds`、`clickable`、`children` 等字段。HarmonyOS 的 `bundleName` 会映射为 `package`，`pagePath` 会保留下来用于页面覆盖率。
+ **输入和状态**：点击、长按、拖拽、按键和文本输入通过 `uitest uiInput` 发送。前台 ability 通过 `aa dump --mission-list` 获取，屏幕尺寸通过 `hidumper -s RenderService -a screen` 获取，截图通过 `snapshot_display` 获取。

## 输出目录和覆盖率报告

设置 `output_dir` 后，HMDroidbot 会在该目录生成交互式报告。运行中或运行结束后，可在浏览器中打开 `<output_dir>/index.html`。

+ `utg.js` 包含生成的 UI transition graph。HarmonyOS 模式下还包括 `num_reached_abilities`、`num_reached_pages`、`app_num_total_abilities`、设备信息和 HAP 哈希。
+ `states/` 包含状态 JSON 和截图。HarmonyOS 截图保存为 `.jpeg`。
+ `views/` 包含控件裁剪图，报告中的 transition details 会引用这些图片。
+ `events/` 包含事件日志，`temp/` 用于运行期间的临时 layout 和截图文件。
+ `dumpsys_package_<bundleName>.txt` 记录安装后 `bm dump -n` 的输出。
+ 只有启用 `-log` 或 `save_log: true` 时，才会生成 `hilog.txt`。

HarmonyOS 报告中的术语会使用 HarmonyOS 概念：Android 的 "activity" 对应 HarmonyOS 的 "ability"，"page" 来自采集到的 ArkUI `pagePath`。

## 故障排除

+ **`config.yml not found`**：请在包含 `config.yml`/`config.yaml` 的目录中运行 HMDroidbot。该文件至少需要包含 `env: Linux`、`env: macOS` 或 `env: windows`。
+ **HDC 命令不正确**：`env` 决定 HMDroidbot 调用 `hdc` 还是 `hdc.exe`。WSL 场景下，建议在 Windows 主系统安装 HDC，并通过 WSL PATH 暴露主系统的 `hdc.exe`（例如 `/mnt/.../hdc.exe`），因为手机连接在 Windows 主系统上。
+ **设备缺失或多设备冲突**：先检查 `hdc list targets`。如果列出了多台设备，请在 YAML 中设置 `device` 或传入 `-t`。
+ **HmDriver 或 UITest 启动失败**：确认设备支持 UITest/Hypium API，HDC 版本为 `3.1.0a+`，并且 `droidbot/adapter/hmdriver/assets/so/<cpu_abi>/agent.so` 存在。
+ **没有 Hilog 文件**：添加 `-log` 或 `save_log: true`，并确认设置了 `output_dir`。

## :mega: 信息
目前，HMDroidbot由[华东师范大学-移动软件分析与测试小组](https://mobile-app-analysis.github.io/)维护。

该项目的主要负责人是[梁锡贤](https://xixianliang.github.io/resume/)和[明孟立](https://ml-ming.dev/)。我们的指导老师是[苏亭教授](https://tingsu.github.io/)。如果您有任何问题或建议，请随时与我们联系。

加入QQ交流群 （群号 904153331）来联系我们并获取最新资讯。

<img width="284" alt="image" src="https://github.com/user-attachments/assets/c42c2bdf-6c3d-4774-a2a4-34adcc84cfe7">

## 致谢

- [Droidbot](https://github.com/honeynet/droidbot)
- [awesome-hdc](https://github.com/codematrixer/awesome-hdc)
- 本项目的开发得到了华为工程师的慷慨帮助和建议。

## 许可证

本项目基于原始MIT许可证（请参见 `LICENSE` 文件），并包括我的贡献，受[Xixian Liang](https://github.com/XixianLiang)的管理（请参见 `LICENSE_NEW` 文件）。
