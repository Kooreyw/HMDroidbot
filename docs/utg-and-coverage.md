# UTG and HarmonyOS coverage reports

HMDroidbot writes a UI transition graph (UTG) and coverage summary into the output directory. The HarmonyOS report is **UI exploration coverage** (abilities, pages, states), not instrumented code coverage.

## Intent

Use the report to:

- see which abilities and pages the run reached
- inspect screenshots and events that moved the UI
- debug exploration (stuck states, ineffective taps, out-of-app screens)

Generation is implemented in `droidbot/utg.py` (`UTG.__output_utg_hm` for HarmonyOS). The HTML viewer is `droidbot/resources/index.html` plus `droidbot/resources/stylesheets/droidbotUI_hm.js`.

## Requirements

1. An output directory: `-o <dir>` or YAML `output_dir`.
2. HarmonyOS mode: `-is_harmonyos` or YAML `system: harmonyOS`.
3. A UTG-based policy (the default). Replay does **not** grow the UTG.

Without `-o` / `output_dir`, `UTG.__output_utg_hm` returns immediately and no `utg.js` is written.

```bash
python3 -m droidbot.start \
  -a app/sample.hap \
  -o output \
  -is_harmonyos \
  -policy dfs_greedy \
  -count 100
```

Equivalent YAML keys:

```yaml
env: Linux
system: harmonyOS
app_path: app/sample.hap
output_dir: output
policy: dfs_greedy
count: 100
```

Open `output/index.html` in a browser after (or during) the run. HarmonyOS copies `index.html` then rewrites the script tag to `stylesheets/droidbotUI_hm.js`. The graph reads `utg.js` (`var utg = { ... }`), which is rewritten after each **effective** transition.

## Output layout

| Path | Produced by | Contents |
| --- | --- | --- |
| `index.html` | `DroidBot.__init__` | Report shell |
| `stylesheets/` | copied from package resources | Viewer, vis.js, CSS |
| `utg.js` | `UTG.__output_utg_hm` | Graph JSON + coverage counters |
| `states/state_*.json` + `screen_*.jpeg` | `DeviceState.save2dir` | First time a `state_str` is seen |
| `events/event_*.json` | `EventLog.save2dir` | Every sent event (UTG-based or not) |
| `views/view_*.jpeg` | `DeviceState.save_view_img` | Crops of views used in events |
| `dumpsys_package_<bundle>.txt` | `DeviceHM.install_app` | `bm dump -n` of the target bundle |
| `hilog.txt` | `Hilog` adapter | Only when `-log` / `save_log: true` |
| `temp/` | `HDC.set_up` | Transient layout / screenshot pulls |

`utg.js` is the source of truth for the sidebar metrics. State JSON is for debugging a single screen.

## How the UTG is built

```text
DeviceHM.get_current_state()
        |  aa dump --mission-list  -->  bundle/ability  (foreground_activity)
        |  HmDriver captureLayout  -->  views (+ pagePath when present)
        v
DeviceState  (state_str, structure_str, pagePath)
        v
UtgBasedInputPolicy.__update_utg()
        v
UTG.add_transition(event, old_state, new_state)
        |  same state_str  -->  ineffective (no new edge)
        |  different       -->  effective edge + rewrite utg.js
```

Foreground identity is `bundleName/abilityName` from `aa dump --mission-list` (`DeviceHM.get_top_activity_name`). Home / lock screen (no `#FOREGROUND` mission) yields `foreground_activity is None`; views are skipped and `state_str` becomes `home_page_or_lock_screen`.

`state_str` is an MD5 of `foreground_activity` plus sorted view signatures (class, resource id, short text, enabled/checked/selected/visible). HarmonyOS excludes `com.ohos.sceneboard` (status bar) from that hash. `structure_str` is a content-free MD5 (class + resource id only) used for structure clustering.

An event is **effective** when `old_state.state_str != new_state.state_str`. Same-hash taps are stored in `ineffective_event_strs` and any existing edge that used that event string is removed.

Only `UtgBasedInputPolicy` subclasses update the graph (`dfs_naive`, `dfs_greedy`, `bfs_naive`, `bfs_greedy`, `random`, `manual`, `memory_guided`). `UtgReplayPolicy` subclasses `InputPolicy` directly, so `-policy replay` does not rewrite HarmonyOS coverage the same way.

## Coverage metrics

Rendered by `getOverallResult()` in `droidbotUI_hm.js` from fields in `utg.js`:

| Report label | `utg.js` field | How it is computed |
| --- | --- | --- |
| Package | `app_package` | `AppHM.package_name` (HAP `bundleName` or installed bundle name) |
| Main ability | `app_main_ability` | HAP `mainAbility` / `bm dump` `mainAbility` |
| # Total abilities | `app_num_total_abilities` | `len(AppHM.activities)` |
| # input events | `num_input_events` | Incremented once per `UtgBasedInputPolicy.generate_event` call |
| # UTG states | `num_nodes` | Unique `state_str` nodes |
| # UTG edges | `num_edges` | Unique `state_str` → `state_str` pairs with at least one effective event |
| # Reached pages | `num_reached_pages` | `len(reached_pages)` |
| Ability_coverage | `num_reached_abilities / app_num_total_abilities` | Unique in-app `foreground_activity` strings vs HAP/`bm dump` ability count |

**Ability coverage is a ratio of two independent counts**, not a set-inclusion check. The numerator is unique `bundle/ability` strings whose prefix is `app.package_name`. The denominator is `len(app.activities)`:

- `.hap` target: abilities from `pack.info` **first module only** (`AppHM.read_hap_info`).
- Non-`.hap` target (installed bundle name): ability names from **all** modules in `bm dump -n` (`AppHM._package_init`).

A reached ability is recorded in `UTG.add_node` only when `foreground_activity.startswith(package_name)`. System UI and other apps do not increase ability coverage.

**Page count is not a percentage.** There is no total-page field in HAP metadata. `DeviceState.pagePath` is the first view attribute named `pagePath` after HmDriver capture (parent `pagePath` is copied onto children when the parent has `bundleName`). Unique values are added to `reached_pages` together with the ability. If capture does not populate `pagePath`, the set may contain `None` and the report number will not reflect distinct ArkUI pages.

Android reports use `droidbotUI.js` and `num_reached_activities` / `app_num_total_activities` instead of ability/page fields.

## App metadata sources

| `-a` / `app_path` | Init | SHA-256 in report | Ability denominator |
| --- | --- | --- | --- |
| Path ending in `.hap` | Unzip `module.json` + `pack.info` | Hash of the HAP file | First module's `abilities` |
| Anything else | Treated as an already-installed bundle; `hdc bm dump -n` | Empty strings | All modules' `abilityInfos[].name` |

`check_package` only warns when the value does not end with `.hap`; it does not resolve the bundle until `AppHM` runs. `DeviceHM.install_app` skips `hdc install` when the bundle is already on the device. Without `-keep_app`, `DroidBot.stop` still uninstalls the bundle after the run.

## Report UI

- **Overall** — coverage table above.
- **Original UTG** — one node per `state_str`, screenshot as the node image. `<FIRST>` / `<LAST>` mark the first seen state and the latest effective destination.
- **Cluster structures** — groups nodes that share `structure_str` (works on HarmonyOS).
- **Cluster activities** — HarmonyOS nodes store the ability under `ability`, but `clusterActivities()` reads `node.activity`. Expect a single `undefined` cluster on HarmonyOS reports; use Overall + Original UTG instead.
- Search matches `node.content` (bundle, ability, `state_str`, resource ids, text).

Click a node for the HTML table of package/ability/`page_path`/`state_str`. Click an edge for the effective events (id, type, view crop, `event_str`).

## Constraints and pitfalls

- **Not code coverage.** Ability/page numbers come from UI dumps and HAP/`bm dump` metadata. They do not reflect executed ArkTS lines or test-kit coverage.
- **Output dir required.** No `-o` means no `index.html` / `utg.js`.
- **Live file.** `utg.js` is overwritten after each effective transition; copy the directory if you need a snapshot mid-run.
- **First-module denominator.** Multi-module HAPs under-count `app_num_total_abilities` relative to `_package_init`.
- **Prefix match.** `com.example.app.debug` matching `com.example.app` depends on the bundle string from `aa dump`; a mismatch yields 0% ability coverage with many UTG states.
- **Ineffective events vanish from the graph.** Repeated taps that do not change `state_str` are dropped from edges even if they appear under `events/`.
- **Replay / monkey / none.** Those policies do not attach to `UtgBasedInputPolicy` (monkey/none have no policy object). Do not expect a growing HarmonyOS coverage graph.
- **`# input events` vs `-count`.** The counter increases inside `generate_event`, which runs after the initial `KillAppEvent` in `InputPolicy.start`. It is not a count of unique screens.
- **Sceneboard filtered from `state_str` only.** Status-bar views are omitted from the hash, not from saved state JSON.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Empty graph / missing `utg.js` | `-o` set; policy is UTG-based; at least one effective transition completed |
| Ability_coverage 0/N | `aa dump --mission-list` bundle prefix vs `AppHM.package_name`; app not foregrounded |
| `# Reached pages` is 0 or 1 | Capture may lack `pagePath`; confirm a `states/state_*.json` view has `"pagePath"` |
| `Error when getting views` | HmDriver / UITest capture failed; see HmDriver notes in the README |
| Cluster activities is one blob | Expected on HarmonyOS (`ability` vs `activity` in `droidbotUI_hm.js`) |
| SHA-256 blank | Targeted an installed bundle name, not a `.hap` |
| Report still looks Android | `is_harmonyos` was false; `index.html` still references `droidbotUI.js` |

## Related codepaths

- `droidbot/droidbot.py` — copy `index.html`, swap in `droidbotUI_hm.js`
- `droidbot/app_hm.py` — HAP vs installed-bundle metadata (`_hap_init`, `_package_init`)
- `droidbot/device_hm.py` — `get_top_activity_name`, `get_current_state`, `install_app`
- `droidbot/adapter/hdc.py` — `HmDriverDumper`, `pagePath` copy during view transfer
- `droidbot/device_state.py` — `state_str`, `structure_str`, `pagePath`, `save2dir`
- `droidbot/input_policy.py` — `UtgBasedInputPolicy.__update_utg`, `num_input_events`
- `droidbot/utg.py` — graph, ability/page sets, `__output_utg_hm`
- `droidbot/resources/stylesheets/droidbotUI_hm.js` — report metrics and clustering
