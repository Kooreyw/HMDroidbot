# Event replay (`-policy replay`)

Replay previously recorded HMDroidbot / DroidBot runs by replaying event JSON files from an earlier output directory.

## Intent

Use replay when you want to:

- reproduce a prior exploration sequence against the same app build
- debug a specific transition that appeared in a previous report
- compare behavior after a small app or device change (see matching constraints below)

Replay is implemented by `UtgReplayPolicy` in `droidbot/input_policy.py`. It is **not** listed in the abbreviated `-policy` help text in `droidbot/start.py`, but `InputManager` accepts `policy: replay` / `-policy replay`.

## Requirements

1. A prior run that wrote event logs under `<prior_output>/events/event_*.json`.
2. Explicit policy selection: `-policy replay` (CLI) or `policy: replay` (YAML).
3. The prior output path: `-replay_output <prior_output>` or YAML `replay_output: <prior_output>`.
4. The same app target (`-a` / `app_path`) and platform mode (`-is_harmonyos` / `system: harmonyOS`) as the original run when possible.

`-replay_output` alone does **not** switch the policy. Without `-policy replay`, the path is stored but unused by the default DFS policy.

## CLI and YAML examples

Record a run (any automated UTG policy works; events are written whenever `output_dir` is set):

```bash
python3 -m droidbot.start \
  -a app/sample.hap \
  -o run1 \
  -is_harmonyos \
  -policy dfs_greedy \
  -count 50
```

Replay that run into a new report directory:

```bash
python3 -m droidbot.start \
  -a app/sample.hap \
  -o run1_replay \
  -is_harmonyos \
  -policy replay \
  -replay_output run1 \
  -count 50
```

Equivalent `config.yml` keys:

```yaml
env: Linux
system: harmonyOS
app_path: app/sample.hap
output_dir: run1_replay
policy: replay
replay_output: run1
count: 50
```

`policy` maps to `opts.input_policy` in `utils.load_yml_args`. `replay_output` is passed through as a normal option attribute.

## How replay works

```text
prior_output/events/*.json  -->  UtgReplayPolicy  -->  device.send_event
                                      |
                                      +--> match start_state == live state_str
                                      +--> reconstruct InputEvent.from_dict(event)
```

1. `InputManager` constructs `UtgReplayPolicy(device, app, replay_output)` when `policy_name == "replay"`.
2. The policy lists `<replay_output>/events`, keeps files ending in `.json`, and sorts them lexicographically (timestamps in `event_YYYY-mm-dd_HHMMSS.json` names usually preserve order).
3. `event_idx` starts at `2`, intentionally skipping the first two recorded files. Historically those were HOME + start-app; current recording also begins with a stop/kill-style bootstrap plus an early start intent.
4. Independently, `InputPolicy.start` still emits a `KillAppEvent` as action `0` before calling `generate_event`, so every replay begins from a stopped app / home-oriented state.
5. For each candidate file, replay loads JSON shaped like:

   ```json
   {
     "tag": "2026-08-03_160122",
     "event": { "event_type": "touch", "...": "..." },
     "start_state": "<md5 state_str>",
     "stop_state": "<md5 state_str>",
     "event_str": "TouchEvent(...)"
   }
   ```

6. An event is replayed only when `event["start_state"]` equals the live device state's `state_str` (MD5 of the captured UI). Matching files are reconstructed with `InputEvent.from_dict`.
7. If the app is not in the foreground, replay returns an `IntentEvent` built as `package[/main_ability]` instead of reading the next file.
8. If no matching event is found, the policy sleeps 5 seconds and retries. After `MAX_REPLY_TRIES` (5) failed attempts at the same position, replay stops producing events.

Custom `-script` / `script_path` attachments do **not** apply: scripts are only wired onto `UtgBasedInputPolicy` subclasses, and `UtgReplayPolicy` subclasses `InputPolicy` directly.

## Event file contract

Produced by `EventLog.save2dir()` during the original run (`droidbot/input_event.py`):

| Field | Meaning |
| --- | --- |
| `event` | Serialized event payload, including `event_type` |
| `start_state` | `state_str` before the event was sent |
| `stop_state` | `state_str` after the event finished |
| `event_str` | Human-readable action string |
| `tag` | Timestamp used in the filename |

Supported `event_type` values for `InputEvent.from_dict` include `key`, `touch`, `long_touch`, `select` / `unselect`, `swipe`, `scroll`, `set_text`, `intent`, `exit`, and `spawn`. `kill_app` is not reconstructed by `from_dict`; skipping the first two files usually avoids replaying that bootstrap event from disk.

## Constraints and pitfalls

- **Exact state matching.** Replay is brittle to UI drift. Different layout dumps, dynamic text, or a different device density can change `state_str` and prevent matches.
- **Same app / similar device.** Replaying against a different HAP build or ABI often fails the `start_state` check even when the screens look similar.
- **Missing `events/` directory.** Construction walks `replay_output/events`; a missing or empty directory raises during policy init.
- **Policy must be set.** Default `dfs_greedy` ignores `-replay_output`.
- **Foreground recovery on HarmonyOS.** When the app leaves the foreground, `UtgReplayPolicy` builds `Intent(suffix="bundle[/ability]")` without `is_harmonyos=True`, which differs from `AppHM.get_start_intent()` (`aa start -b ... -a ...`). Prefer keeping the app foregrounded during replay, or verify that recovery still works on your device.
- **Count still applies.** `-count` / `count` limits total actions in `InputPolicy.start`, including the initial `KillAppEvent`.
- **New output is still written.** Point `-o` / `output_dir` at a fresh directory so the replay run gets its own `index.html`, `utg.js`, and `states/` artifacts.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Replay never sends recorded touches | Confirm `-policy replay` and that `replay_output/events` contains `event_*.json`. |
| Logs show repeated waits / Back events | Live `state_str` does not match any remaining `start_state`; compare UI hierarchy with the original run (`-debug`). |
| `Loading ... failed` | Corrupt or partial JSON in `events/`; skip or repair that file. |
| Stops after a few retries | Hit `MAX_REPLY_TRIES` (5) without a match; reset the app UI closer to the recorded path or re-record. |
| Script events never fire during replay | Expected: scripts attach only to UTG-based policies, not replay. |

## Related codepaths

- `droidbot/start.py` — `-policy`, `-replay_output`
- `droidbot/utils.py` — YAML `policy` / `replay_output` loading
- `droidbot/input_manager.py` — policy selection
- `droidbot/input_policy.py` — `UtgReplayPolicy`, `MAX_REPLY_TRIES`
- `droidbot/input_event.py` — `EventLog` persistence and `InputEvent.from_dict`
