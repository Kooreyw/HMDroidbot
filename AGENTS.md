# AGENTS.md

## Cursor Cloud specific instructions

### Overview

HMDroidbot is a Python CLI tool for automated UI testing of HarmonyOS and Android apps. It is **not** a web service — there are no Docker containers, databases, or web servers. The tool interacts with physical/emulated mobile devices via `hdc` (HarmonyOS) or `adb` (Android).

### Python environment

- **Python 3.11** is required (installed via `deadsnakes` PPA). The system default is 3.12 but the project requires 3.10–3.11.
- A virtualenv lives at `/workspace/.venv` (created with `python3.11 -m venv .venv`).
- Activate it: `source /workspace/.venv/bin/activate`
- The package is installed in editable mode (`pip install -e .`).

### Running the tool

- `droidbot -h` — show CLI help
- `python -m droidbot.start -h` — alternative invocation
- Full usage requires a connected HarmonyOS or Android device. Without a device, you can only test imports and lint.

### Linting

- `flake8 droidbot/ --max-line-length=120` — full lint (note: the codebase has pre-existing lint warnings)
- `flake8 droidbot/ --select=E9,F63,F7,F82` — critical errors only

### Testing

- `pytest` is installed but the codebase has no automated test suite yet.
- Core functionality verification: run `python -c "from droidbot.droidbot import DroidBot; print('OK')"` to confirm imports work.

### Key caveats

- `droidbot/monitor.py` requires the optional `frida` package. It will fail to import without it. This does not affect core functionality.
- `config.yml` at the repo root must have `env: Linux` for cloud environments (already set).
- `.venv/` is gitignored — the update script recreates it each session.
