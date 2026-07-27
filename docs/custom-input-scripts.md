# Custom input scripts (`-script`)

HMDroidbot can inject state-specific events via a JSON **DroidBotScript**. Use this when exploration policies get stuck on login walls, welcome carousels, or other deterministic UI gates.

## When scripts run

1. CLI: `-script <path.json>` (`dest=script_path` in `droidbot/start.py`).
2. YAML: set `script_path` in `config.yml` / `config.yaml` (generic key passthrough in `utils.load_yml_args`).
3. `InputManager` loads the JSON into `DroidBotScript` (`droidbot/input_manager.py`).
4. Scripts attach only when the selected policy is an instance of `UtgBasedInputPolicy` (`InputManager.get_input_policy`):
   - `dfs_greedy` (default), `dfs_naive`
   - `bfs_greedy`, `bfs_naive`
   - `random`
   - `manual`
   - `memory_guided` (requires optional ML deps)
5. Policies such as `monkey`, `none`, and `replay` do **not** receive a script object.

On each exploration step (`UtgBasedInputPolicy.generate_event`):

1. Continue any in-progress scripted operation.
2. Else match the current UI state against `main` and start a matched operation.
3. Else fall through to the normal UTG / random exploration event.

Scripts therefore **override** exploration for matched states and otherwise leave the policy alone.

## Script shape

Top-level keys required by `DroidBotScript.script_grammar`:

| Key | Role |
| --- | --- |
| `views` | Named view selectors (regex / coordinates) |
| `states` | Named UI-state selectors |
| `operations` | Named ordered event lists |
| `main` | Map `state_id` → action (round-robin or probabilistic) |

Identifiers must match `^[^\d\W]\w*$` and must not use the reserved id `default`.

### View selectors

Optional fields (all regex strings unless noted):

- `text`, `resource_id`, `content_desc`, `class`
- `in_coordinates` / `out_coordinates`: lists of `[x, y]` integer pairs

A view matches only if required dict keys (`text`, `resource_id`, `class`, `bounds`) exist and every configured selector matches.

### State selectors

Optional fields:

- `activity`: regex against `DeviceState.foreground_activity`
- `services`: list of regexes; each must match some background service
- `views`: list of view ids; **every** listed view must be present

### Operations and events

Each operation is a list of event dicts. Common fields:

- `event_type` (required): see table below
- `target_view`: view id; resolved against the last/current device state before send
- Event-specific fields (`text`, `direction`, `name`, …)

Supported `event_type` values (`InputEvent.from_dict`):

| `event_type` | Notes |
| --- | --- |
| `touch` | Tap matched view or coordinates |
| `long_touch` | Long press |
| `scroll` | Direction must be uppercase `UP` / `DOWN` / `LEFT` / `RIGHT` (see `ScrollEvent.send`) |
| `swipe` | Swipe gesture |
| `set_text` | Requires `text` |
| `key` | Key event (`name`) |
| `select` / `unselect` | Selection toggles |
| `intent` | Intent / ability launch payload |
| `exit` | Stop input generation |
| `spawn` | Distributed worker spawn; needs DroidMaster |

If `target_view` does not match any current view, HMDroidbot logs a warning and still builds the event without a bound view.

### `main` actions

- **Round-robin**: `"state_id": ["op_a", "op_b"]` — cycles operations on each match.
- **Probabilistic**: `"state_id": [{"op_id": "op_a", "prob": 0.6}, ...]` — probabilities must sum to ≤ 1; remainder is a no-op (fall through to exploration).

## Minimal example (login gate)

```json
{
  "views": {
    "login_email": { "resource_id": ".*email", "class": ".*EditText" },
    "login_password": { "resource_id": ".*password", "class": ".*EditText" },
    "login_button": { "resource_id": ".*next", "class": ".*Button" }
  },
  "states": {
    "login_state": {
      "views": ["login_email", "login_password", "login_button"]
    }
  },
  "operations": {
    "login_operation": [
      { "event_type": "set_text", "target_view": "login_email", "text": "sample@email.com" },
      { "event_type": "set_text", "target_view": "login_password", "text": "sample_password" },
      { "event_type": "touch", "target_view": "login_button" }
    ]
  },
  "main": {
    "login_state": ["login_operation"]
  }
}
```

Repo samples (validated against `DroidBotScript`):

- `script_samples/pass_login_script.json`
- `script_samples/pass_welcome_script.json`
- `script_samples/probabilistic_script.json`
- `script_samples/spawn.json` (distributed `spawn` events)

## Run examples

```bash
# CLI
python3 -m droidbot.start \
  -a app/sample.hap -o output -is_harmonyos \
  -script script_samples/pass_login_script.json \
  -policy dfs_greedy -count 200

# YAML (cwd must contain config.yml / config.yaml)
# script_path: script_samples/pass_welcome_script.json
# policy: random
python3 -m droidbot.start
```

## Constraints and pitfalls

- Script matching depends on a successful UI hierarchy capture. On HarmonyOS that path goes through HmDriver / UITest; empty views mean states never match.
- Prefer regex view selectors that survive localization and id prefixes (`.*email` rather than a fully hard-coded id).
- `scroll` directions in scripts should be uppercase; lowercase values do not move the gesture in `ScrollEvent.send`.
- `spawn` only makes sense under DroidMaster distributed mode.
- Probabilities > 1 raise `ScriptSyntaxError`; sum < 1 intentionally allows exploration on some matches.
- Duplicate ids across `views` / `states` / `operations` raise `ScriptSyntaxError`.

## Code map

| Concern | Location |
| --- | --- |
| CLI `-script` | `droidbot/start.py` |
| YAML load / `script_path` | `droidbot/utils.py` (`load_yml_args`) |
| JSON load + policy attach | `droidbot/input_manager.py` |
| Match / transform events | `droidbot/input_script.py` |
| Priority over exploration | `droidbot/input_policy.py` (`UtgBasedInputPolicy.generate_event`) |
| Event constructors | `droidbot/input_event.py` (`InputEvent.from_dict`) |
