<p align="center">
  <img src="https://raw.githubusercontent.com/sig9org/tasks/main/assets/logo.png" alt="tasks">
</p>

# tasks

A collection of reusable [go-task](https://taskfile.dev) Taskfile fragments. Include this repository directly from GitHub (`main` branch) in your project's `Taskfile.yml` to pull in common tasks — cleanup, Go builds, OS package updates, uv sync, and elapsed-time reporting.

```yaml
includes:
  _time: https://raw.githubusercontent.com/sig9org/tasks/main/tasks/_time.yml
  _go: https://raw.githubusercontent.com/sig9org/tasks/main/tasks/_go.yml
```

## Prerequisites

Remote Taskfiles are an experimental go-task feature, so the consuming project must enable `TASK_X_REMOTE_TASKFILES=1` (otherwise `task` fails with `Remote taskfiles are not enabled.`). See https://taskfile.dev/experiments/remote-taskfiles for details.

## Internal task libraries (`tasks/_*.yml`)

Each library is provided as `internal: true` tasks. Include it under any name (by convention `_cleanup`, etc.) and define your own tasks that call into it (see `examples/`).

| Library | What it provides |
| --- | --- |
| `_cleanup.yml` | Removes `.idea` / `.terraform` / `.vscode` and OS junk files, and force-refreshes the remote Taskfile cache |
| `_go.yml` | Build for the current platform, cross-compile for all platforms, clean `dist`, `go vet` / `go test` |
| `_os.yml` | Detects Amazon Linux / macOS (Homebrew) / Ubuntu and checks for, applies, or removes unnecessary packages accordingly |
| `_time.yml` | Records a task's start/end time and prints the elapsed time |
| `_uv.yml` | Freeze/sync packages with [uv](https://docs.astral.sh/uv/) |

## Examples (`examples/`)

`examples/` contains sample Taskfiles wiring up each library.

- `examples/cleanup.yml` — `cleanup` (`c`)
- `examples/go.yml` — `cleanup` (`c`), `go-all-build` (`ga`), `go-build` (`gb`), `go-clean` (`gc`), `go-test` (`gt`)
  - Expects `BINARY_NAME` / `VERSION_PKG` to be provided via `project.ini` (dotenv).
- `examples/os.yml` — `cleanup` (`c`), `os-check` (`oc`), `os-remove` (`or`), `os-update` (`ou`)
- `examples/uv.yml` — `uv-freeze` (`uf`), `uv-sync` (`us`)

Every task depends on `_time:start` and places `defer: { task: _time:end }` first in `cmds:`, so the elapsed-time display always runs regardless of whether the task succeeds or fails.

## Developing this repository

`tasks/*.yml`, `examples/*.yml`, and the root `Taskfile.yml` are all generated output. Don't edit them directly — edit the fragments under `templates/` and `config.yml`, then regenerate.

1. Copy `config.yml.example` to `config.yml` and edit the output file definitions (the combination of `headers` / `vars` / `tasks` / `includes`) as needed.
2. Edit the fragments under `templates/headers/`, `templates/includes/`, `templates/tasks/`, and `templates/vars/`.
3. Run `task build` (alias `b`) to regenerate `tasks/*.yml`, `examples/*.yml`, and `Taskfile.yml` via `gotasky`.

This repository's own `Taskfile.yml` also provides:

- `task cleanup` (alias `c`) — removes unnecessary files and force-refreshes the remote Taskfile cache

## License

[MIT License](LICENSE)
