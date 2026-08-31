# HarmonyOS app targeting: HAP vs installed bundle

HMDroidbot can test a HarmonyOS app from a local `.hap` file **or** from a **bundle name already installed on the device**. Detection is a suffix check: if `-a` / YAML `app_path` does not end with `.hap`, startup treats the value as a bundle name.

This is HarmonyOS-only (`AppHM`). Android always parses `-a` as an APK via androguard.

## Intent

- Re-test an app that is already on the device without keeping a HAP on disk.
- Understand when `hdc install` runs, when it is skipped, and when the app is uninstalled after the run.
- Avoid deleting a user-installed app at shutdown (default behavior).

## Requirements

1. HarmonyOS mode: `-is_harmonyos` or YAML `system: harmonyOS`.
2. An output directory: `-o <dir>` or YAML `output_dir`. `DeviceHM.install_app` always reads `dumpsys_package_<bundle>.txt` from that directory.
3. For a **bundle name**: the app must already be installed. Metadata is queried during `DroidBot` construction, **before** `install_app`.
4. For a **`.hap` file**: the path must exist (relative paths are joined with the current working directory).

```bash
# HAP file (installs if the bundle is not already on the device)
python3 -m droidbot.start \
  -a app/sample.hap \
  -o output \
  -is_harmonyos \
  -count 100

# Already-installed bundle (no HAP). Keep the app when the run ends.
python3 -m droidbot.start \
  -a com.example.myapp \
  -o output \
  -is_harmonyos \
  -keep_app \
  -count 100
```

Equivalent YAML. After the built-in key mappings, other keys are applied with `setattr(opts, key, value)` using the YAML key as written — use `keep_app` (the argparse destination), not `-keep_app`:

```yaml
env: Linux
system: harmonyOS
app_path: com.example.myapp
output_dir: output
count: 100
keep_app: true
```

Startup prints `[Warning] The given app ... is probably a package name.` when the value does not end with `.hap`. That warning is expected for bundle targeting.

## How targeting is chosen

```text
parse_args / load_yml_args
        v
check_package()
  ends with ".hap"?  --yes-->  join cwd if needed; require file exists
                     --no--->  warn; leave value as-is (bundle name)
        v
DroidBot.__init__
  AppHM.device_serial = <serial>     # class attribute, used by bm dump
  AppHM(app_path)
        |  *.hap  -->  _hap_init()     unzip module.json + pack.info
        |  else   -->  _package_init()  bm dump -a / bm dump -n  (must be installed)
        v
DroidBot.start()
  DeviceHM.install_app(app)          # hdc install only if bundle missing
        v
DroidBot.stop()
  uninstall unless -keep_app / keep_app: true
```

`AppHM` is constructed in `DroidBot.__init__`, which runs before `device.connect()` and `device.install_app()`. Bundle-name targeting therefore cannot install a missing app: `_package_init` reads `_dumpsys_package_info`, which raises `RuntimeError("<bundle> not installed on device.")` if `bm dump -a` does not list the name.

## HAP vs installed-bundle metadata

| Field | `.hap` (`_hap_init`) | Bundle name (`_package_init`) |
| --- | --- | --- |
| `package_name` | `pack.info` `summary.app.bundleName` | the string you passed as `-a` |
| `main_activity` | first module `mainAbility` in `pack.info` | `hapModuleInfos[0]["mainAbility"]` from `bm dump -n` |
| `activities` (coverage denominator) | first module `abilities` only | **all** modules' `abilityInfos[].name` |
| `hashes` (UTG `app_sha256`) | MD5 / SHA-1 / SHA-256 of the HAP file | `["", "", ""]` (empty SHA-256 in `utg.js`) |

Coverage still counts reached abilities independently (see `UTG.add_node`). The denominator `app_num_total_abilities` is `len(app.activities)`, so a multi-module app can show a different ratio for HAP vs bundle targeting.

## Install skip

`DeviceHM.install_app` calls `hdc bm dump -a` (`HDC.get_installed_apps`). If `app.get_package_name()` is already in that list, **`hdc install` is not run** — even when you passed a newer HAP. The install command includes `-r` (replace), but that command is only built in the not-installed branch, so replace never happens for an existing bundle.

To force a HAP refresh: uninstall the bundle first (or run without `-keep_app` so the previous session uninstalls it), then start with the new `.hap`.

If you pass a bundle name that is **not** installed, the process never reaches `install_app`.

## `-keep_app` and uninstall

Default is **uninstall at the end of the run** (`DroidBot.stop`: `if not self.keep_app: self.device.uninstall_app(self.app)`). `-keep_app` is `store_true` (CLI default `False`).

For bundle-name targeting, always pass `-keep_app` or YAML `keep_app: true` unless you intend to remove the app from the device.

`hdc uninstall <bundleName>` runs only if the bundle is still listed by `bm dump -a`. HarmonyOS `keep_env` is unrelated (adapter `tear_down` is a no-op for HDC/Hilog).

## Constraints and pitfalls

- **Suffix is case-sensitive.** `sample.HAP` is treated as a bundle name, not a HAP file.
- **Android does not support bundle names.** Without `-is_harmonyos`, `App` always opens the path as an APK.
- **Exactly one connected target** is still required (`identify_device_serial`). Serial is taken from `hdc list targets` when HarmonyOS mode is on.
- **`-o` is required on HarmonyOS.** After the optional write, `install_app` always opens `output_dir/dumpsys_package_<bundle>.txt` and parses `hapModuleInfos[0]["mainAbility"]` into `app.dumpsys_main_activity`. `AppHM.get_main_activity()` does **not** fall back to that field; launch still uses `main_activity` from HAP/`bm dump`.
- **`bm dump -n` JSON** must be valid after the first line (bundle name). A dump format change will fail `_package_init` or the dumpsys parse in `install_app`.
- **Do not pass a HAP path without the `.hap` suffix** (renamed file). Startup will try `bm dump -n` with that string and fail.

## Codepaths

- `droidbot/start.py` — `-a` / `-keep_app` / `-is_harmonyos`; calls `check_package` then `DroidBot`
- `droidbot/utils.py` — `check_package` (`.hap` vs bundle), YAML `app_path` → `apk_path`
- `droidbot/droidbot.py` — sets `AppHM.device_serial`, constructs `AppHM`, `install_app` in `start()`, uninstall in `stop()`
- `droidbot/app_hm.py` — `_hap_init` vs `_package_init` / `_dumpsys_package_info`
- `droidbot/device_hm.py` — `install_app` skip + dumpsys file, `uninstall_app`
- `droidbot/adapter/hdc.py` — `HDC.get_installed_apps` (`bm dump -a`), `get_relative_path` for HAP install
- `droidbot/utg.py` — `app_sha256` / `app_num_total_abilities` from the `AppHM` fields above
