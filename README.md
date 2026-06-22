### :book: [中文文档](https://github.com/XixianLiang/HMDroidbot/blob/master/README_cn.md)


## :memo: About
HMDroidbot (HM stands for HarmonyOS, Droid stands for Android) is a lightweight test input generator for HarmonyOS (and Android). It forks from [Droidbot](https://github.com/honeynet/droidbot) and supports HarmonyOS NEXT devices.
It can send random or scripted input events to test an HarmonyOS app, achieve higher code coverage more quickly, and generate a UI transition graph (UTG) after testing.

## :smiling_face_with_three_hearts: Awesome Features 
:boom: Support coverage report for HarmonyOS now! Including ability, page, *etc.* and some HarmonyOS-specific metrics! See the sample report below!

![image](https://github.com/user-attachments/assets/1dfbb6f8-c9ab-48b2-8043-5474719a7466)

:boom: Support both Android and HarmonyOS NEXT devices. Use the flag `-is_harmonyos` to specify the target system.

:boom: Support configuring with YAML file. Easy to get start with.

:boom: Source code improvement. Easier to read and debug. Added typing to the source code and colorized the log. Use `-debug` flag to print the debug level log to the terminal!

:boom: Use `-log` flag to get the hilog from the device. Check it in the report directory!

## Prerequisite

1. `Python 3.10 ~ 3.11`. Some required official packages have been removed in Python 3.13; Python 3.11 is recommended for stable usage.
2. `HDC cmdtool 3.1.0a+`.
3. For HarmonyOS runs, a `.hap` file or an already installed bundle name, plus one connected HarmonyOS device or emulator visible in `hdc list targets`.

## How to use

1. **Install HMDroidbot**

    Clone this repo and install it with `pip`.

    :one: *(Optional)* Set up a virtual environment before installation. See the [venv module](https://realpython.com/python-virtual-environments-a-primer/) for details.

    In macOS or Linux:
    ```shell
    python3 -m venv droidenv
    source droidenv/bin/activate
    ```

    In Windows:
    ```powershell
    python3 -m venv droidenv
    .\droidenv\Scripts\activate
    ```

    :two: Install the package in editable mode.

    ```shell
    git clone https://github.com/XixianLiang/HMDroidbot.git
    cd HMDroidbot
    pip install -e .
    ```

    :three: If installation succeeds, `droidbot -h` should work. If the command is not on your PATH, use `python3 -m droidbot.start -h`.

2. **Quick Start (only available in WSL now)**

    :wave: Run the provided `run_sample.sh` file to download a sample HAP and try HMDroidbot.
    ```bash
    bash run_sample.sh
    ```

    We also provide sample HAPs for testing [here](https://github.com/XixianLiang/HarmonyOS_NEXT_apps).

3. **Set up `config.yml`**

    `config.yml` or `config.yaml` must exist in the current working directory before HMDroidbot starts. The startup code loads it after parsing CLI options, and the HDC adapter also reads `env` while importing.

    The required key is `env`, which selects the host command used to call HDC:
    ```yaml
    # config.yml
    env: Linux      # Linux/macOS use hdc; windows uses hdc.exe
    ```

    You can also move common CLI options into YAML. Values in YAML override matching CLI options.

    | YAML key | Equivalent CLI / behavior |
    | --- | --- |
    | `system: harmonyOS` | Enables HarmonyOS mode, equivalent to `-is_harmonyos`. |
    | `app_path` | Target `.hap` path or installed bundle name; maps to `-a`. |
    | `output_dir` | Report directory; maps to `-o`. |
    | `device`, `target`, `device_serial` | Target from `hdc list targets`; maps to `-t`. |
    | `count` | Number of input events; maps to `-count`. |
    | `policy` | Input policy, such as `dfs_greedy`, `bfs_greedy`, `random`, `manual`, or `none`; maps to `-policy`. |
    | `save_log: true` | Saves device logs; maps to `-log`. |

    Example HarmonyOS configuration:
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

4. **Start HMDroidbot**

    With the YAML configuration above, start directly:
    ```bash
    droidbot
    # or
    python -m droidbot.start
    ```

    Or pass the main options on the command line:
    ```bash
    droidbot -a app/sample.hap -o output -t 23E**********1843 -count 1000 -is_harmonyos -debug
    python -m droidbot.start -a app/sample.hap -o output -t 23E**********1843 -count 1000 -is_harmonyos -debug
    ```

    Notes:

    + If exactly one device is connected, HMDroidbot auto-fills the serial from `hdc list targets`. If no device or multiple devices are connected, specify `-t <device_serial>` or `device: <device_serial>`.
    + For HarmonyOS emulators, `hdc list targets` usually returns a loopback target such as `127.0.0.1:5555`; pass that value with `-t 127.0.0.1:5555`. The emulator must be configured in DevEco Studio. See the [configuration tutorial](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/ide-emulator-create-V5).
    + Use `-policy random` or `policy: random` to choose the random policy. The separate `-random` flag only adds randomness inside supported policies.
    + Use `-debug` while debugging HMDroidbot itself.
    + Use `-log` or `save_log: true` to save HarmonyOS hilog output to `<output_dir>/hilog.txt`.
    + Run `droidbot -h` for the full option list.

    **vscode `launch.json` example**

   <img width="1134" alt="image" src="https://github.com/user-attachments/assets/bffde3f3-deea-41fb-9087-fb7eb3772bd5">

## HarmonyOS runtime model

HMDroidbot keeps the DroidBot input-policy model, but swaps in HarmonyOS-specific app, device, and UI adapters when `system: harmonyOS` or `-is_harmonyos` is set.

+ **HAP metadata and lifecycle**: `AppHM` reads `.hap` archives from `module.json` and `pack.info` to find the bundle name, main ability, abilities, and file hashes. `DeviceHM` installs HAPs with `hdc install -r`, refreshes bundle metadata with `bm dump -n <bundleName>`, starts apps with `aa start`, and stops them with `aa force-stop`.
+ **UI hierarchy capture**: `HDC.get_views()` uses `HmDriverDumper` by default. On first use it starts `HmClient`, pushes the CPU-ABI-specific `agent.so` to `/data/local/tmp/agent.so`, runs `uitest start-daemon singleness`, forwards port `8012`, and requests `captureLayout` through Hypium's `hypiumApiHelper`.
+ **View normalization**: HarmonyOS layout nodes are normalized to DroidBot-style view dictionaries so existing input policies and event generation can keep using fields such as `package`, `class`, `resource_id`, `bounds`, `clickable`, and `children`. HarmonyOS `bundleName` becomes `package`, and `pagePath` is preserved for page coverage.
+ **Input and state**: taps, long taps, drags, key events, and text input are sent through `uitest uiInput`. Foreground ability detection uses `aa dump --mission-list`; display size uses `hidumper -s RenderService -a screen`; screenshots use `snapshot_display`.

## Output and coverage report

When `output_dir` is set, HMDroidbot creates an interactive report in that directory. Open `<output_dir>/index.html` in a browser after or during a run.

+ `utg.js` contains the generated UI transition graph. In HarmonyOS mode it includes `num_reached_abilities`, `num_reached_pages`, `app_num_total_abilities`, device information, and HAP hashes.
+ `states/` contains state JSON files and screen images. HarmonyOS screenshots are saved as `.jpeg`.
+ `views/` contains cropped widget images referenced by the transition details in the report.
+ `events/` contains event logs, and `temp/` is used for temporary layout and screenshot files while the run is active.
+ `dumpsys_package_<bundleName>.txt` records the `bm dump -n` output after install.
+ `hilog.txt` is created only when `-log` or `save_log: true` is enabled.

Terminology in the HarmonyOS report follows HarmonyOS concepts: Android "activity" maps to HarmonyOS "ability", and a "page" is derived from the ArkUI `pagePath` found in the captured view hierarchy.

## Trouble shooting

+ **`config.yml not found`**: run HMDroidbot from the repository or another directory that contains `config.yml`/`config.yaml`. At minimum, the file needs `env: Linux`, `env: macOS`, or `env: windows`.
+ **Wrong HDC command**: `env` controls whether HMDroidbot calls `hdc` or `hdc.exe`. On WSL, install HDC on Windows and expose the host `hdc.exe` through the WSL PATH, for example with a `/mnt/.../hdc.exe` path, because the phone is connected to the Windows host.
+ **Multiple or missing targets**: check `hdc list targets`. If more than one target is listed, set `device` in YAML or pass `-t`.
+ **HmDriver or UITest startup failures**: confirm the device supports UITest/Hypium APIs, the HDC version is `3.1.0a+`, and the ABI-specific `agent.so` exists under `droidbot/adapter/hmdriver/assets/so/<cpu_abi>/agent.so`.
+ **No Hilog file**: add `-log` or `save_log: true`, and make sure `output_dir` is set.

## :mega: Info
Currently, HMDroidbot is maintained by [华东师范大学-移动软件分析与测试小组](https://mobile-app-analysis.github.io/). 

This project is led by [Xixian Liang](https://xixianliang.github.io/resume/) and [Mengli Ming](https://ml-ming.dev/). We are supervised by Prof. [Ting Su](https://tingsu.github.io/). Feel free to contact us if you have any questions or advices.

Join the QQ group (ID 904153331) to contact us and get the latest info.

<img width="284" alt="image" src="https://github.com/user-attachments/assets/c42c2bdf-6c3d-4774-a2a4-34adcc84cfe7">

## Acknowledgement

- [Droidbot](https://github.com/honeynet/droidbot)
- [awesome-hdc](https://github.com/codematrixer/awesome-hdc)
- The development of this project receives generous help and advice from the HUAWEI engineers.

## License

This project is based on the original MIT License (see `LICENSE` file) and includes my contributions, which are governed by [Xixian Liang](https://github.com/XixianLiang) (see `LICENSE_NEW` file).
