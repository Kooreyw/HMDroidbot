# Configuration and HDC environment runbook

This guide documents the startup path that prepares HMDroidbot to talk to
Android Debug Bridge (`adb`) or HarmonyOS Device Connector (`hdc`). Use it when
setting up a new workstation, debugging device discovery, or adding new CLI/YAML
options.

Source anchors:

- `droidbot/start.py` parses CLI flags, loads YAML, selects a device, then starts
  `DroidBot`.
- `droidbot/utils.py` implements YAML loading, option overrides, device
  auto-detection, and HAP path checks.
- `droidbot/adapter/hdc.py` reads `env` at import time to choose `hdc` or
  `hdc.exe`.

## Startup flow

1. Run `droidbot` or `python3 -m droidbot.start` from the repository or another
   working directory that contains `config.yml` or `config.yaml`.
2. `parse_args()` parses CLI flags such as `-a`, `-o`, `-policy`, `-count`,
   `-log`, and `-is_harmonyos`.
3. `get_yml_config()` opens `config.yml`/`config.yaml` from the current working
   directory.
4. `load_yml_args()` applies YAML values over matching CLI values.
5. `identify_device_serial()` lists connected targets with `adb devices` or
   `hdc list targets`.
6. `check_package()` validates a `.hap` path when one is provided.

Because YAML is loaded during startup and the HDC adapter also reads it during
module import, keep a minimal `config.yml` available even when most values are
passed on the command line.

## Minimal HarmonyOS configuration

```yaml
# Selects the host command name used by droidbot/adapter/hdc.py.
# mac, macOS, linux, and unix use "hdc"; win and windows use "hdc.exe".
env: Linux

# Maps to the -is_harmonyos option when the value is harmonyOS.
system: harmonyOS
```

Run with CLI values:

```bash
python3 -m droidbot.start \
  -a app/sample.hap \
  -o output \
  -count 1000 \
  -policy random \
  -log
```

Run mostly from YAML:

```yaml
env: Linux
system: harmonyOS
app_path: app/sample.hap
output_dir: output
count: 1000
policy: random
save_log: true
```

```bash
python3 -m droidbot.start
```

## YAML keys used by startup

`load_yml_args()` treats YAML as an override layer after CLI parsing. Prefer the
canonical keys below in shared examples.

| YAML key | Effect | Notes |
| --- | --- | --- |
| `env` | Selects the HDC executable. | Required before HarmonyOS HDC code can run. Accepted values are `mac`, `macOS`, `linux`, `unix`, `win`, and `windows`. |
| `system` | Sets `opts.is_harmonyos` when the value is `harmonyOS`. | Any other value leaves HarmonyOS mode disabled. |
| `app_path` | Sets the app package path used by `-a`. | Use this key for HAP files in YAML. The CLI destination is named `apk_path`, but YAML maps `app_path` to it. |
| `policy` | Sets the input policy used by `-policy`. | Current help lists `none`, `monkey`, `dfs`, `greedy_dfs`, `bfs`, `greedy_bfs`, and `random`. |
| `output_dir` | Sets the report/output directory used by `-o`. | HDC layout dumps use a `temp` subdirectory under this output path. |
| `count` | Sets the total event count used by `-count`. | CLI parsing expects an integer; keep YAML values numeric. |
| `target`, `device`, `device_serial` | Sets `opts.device_serial`. | See the current multi-device constraint below. |
| Other keys except `env` | Applied as attributes on the parsed options object. | Only add keys that are consumed by `DroidBot`, `DroidMaster`, or related startup code. |

## HDC executable selection

`droidbot/adapter/hdc.py` maps `env` to the command name:

| `env` value | Command HMDroidbot runs |
| --- | --- |
| `mac`, `macOS`, `linux`, `unix` | `hdc` |
| `win`, `windows` | `hdc.exe` |

Practical implications:

- Native macOS/Linux setups should have `hdc` on `PATH`.
- Native Windows setups should have `hdc.exe` on `PATH`.
- WSL setups often expose the Windows host tool through `/mnt/.../hdc.exe`.
  In that case, set `env: windows` or create a wrapper named `hdc` and use
  `env: Linux`.
- `run_sample.sh` assumes the WSL/host pattern and calls `hdc.exe list targets`
  directly.

Validate the command before starting HMDroidbot:

```bash
hdc list targets      # env: Linux/macOS/unix
hdc.exe list targets  # env: windows/win, including many WSL setups
```

## Device selection constraints

Startup currently auto-detects the connected target after applying CLI and YAML
options:

- Android mode runs `adb devices`.
- HarmonyOS mode runs `<HDC_EXEC> list targets`.
- Zero connected targets raises `No connected device`.
- More than one connected target raises
  `More than one attached devices, please specify one device serial`.
- With exactly one target, that serial is assigned to `options.device_serial`.

Current implementation note: `identify_device_serial()` performs the one-target
check unconditionally, so `-t`, `-d`, `device`, `target`, and `device_serial`
do not bypass the multi-target error in this codepath. If multiple devices or
emulators are attached, disconnect extras before running this version.

## HAP path rules

Use `app_path` in YAML or `-a` on the CLI for the target package.

- Paths ending in `.hap` are checked before the run starts.
- Relative `.hap` paths are resolved from the current working directory.
- Absolute `.hap` paths are preserved.
- Non-`.hap` values are treated as package names and only produce a warning.

Examples:

```yaml
app_path: app/sample.hap
```

```bash
python3 -m droidbot.start -a /absolute/path/to/sample.hap -o output
```

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Startup fails before CLI values appear to take effect. | Confirm `config.yml` or `config.yaml` exists in the current working directory; startup always calls `get_yml_config()`. |
| `hdc command not found` or `hdc.exe command not found`. | Match `env` to the command available on `PATH`, then verify with `hdc list targets` or `hdc.exe list targets`. |
| WSL cannot see a USB phone. | Prefer the host Windows HDC installation and expose `hdc.exe` inside WSL, as shown by `run_sample.sh`, because the phone is connected to the host OS. |
| `No connected device`. | Run the same device-list command HMDroidbot will use and confirm the target is online. |
| `More than one attached devices...`. | This version requires exactly one connected target during startup; disconnect extra devices/emulators. |
| HAP path does not exist. | Run from the directory expected by `config.yml`, or use an absolute `app_path`/`-a` value. |
| YAML `apk_path` is ignored. | Use `app_path`; the loader maps `app_path` onto the internal `apk_path` option. |

