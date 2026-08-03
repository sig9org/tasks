# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is `sig9org/tasks` — a library of reusable [go-task](https://taskfile.dev) Taskfile fragments. Other projects consume it by `include`-ing files straight from GitHub over HTTPS:

```yaml
includes:
  _time: https://raw.githubusercontent.com/sig9org/tasks/main/tasks/_time.yml
```

This repo's own root `Taskfile.yml` and `examples/*/Taskfile.yml` do the same thing — they include their own published copies from GitHub `main`, rather than the local files in the working tree. This has an important consequence: **editing a file in `tasks/` here has no effect on `task` runs in this repo (or in any consumer project) until it is committed and pushed to `main`, and the consumer's local `.task/remote/` cache is cleared/refreshed.** When iterating on a shared fragment, either test it against a scratch Taskfile whose `includes:` point at the local relative path (e.g. `./tasks/_time.yml`) instead of the GitHub URL, or push to `main` and force a refresh (see `task cleanup` below).

Remote includes are an experimental go-task feature and require `TASK_X_REMOTE_TASKFILES=1` to be set (either exported, or passed via `--global` env) — otherwise `task` fails with "Remote taskfiles are not enabled."

## Repository structure

- `templates/` — hand-edited **source** fragments, combined by an external generator tool called `gotasky` (not part of this repo) into the files below. `config.yml`'s top-level `templates:` list names which directories under here are template roots; each root has the same four subdirectory kinds (`headers/`, `vars/`, `tasks/`, `includes/`), and fragment filenames only need to be unique within their root, not across roots:
  - `templates/tasks/` — sources for this repo's own published internal libraries (`tasks/_*.yml`). `templates/tasks/tasks/_*.yml` are the task bodies, `templates/tasks/vars/_*.yml` their vars, `templates/tasks/includes/_*.yml` are the `includes:` blocks (pointing at this repo's own published `tasks/_*.yml`, for consumer Taskfiles that want to pull one in) and `templates/tasks/headers/` holds shared header blocks (`version`, `silent`, `dotenv`, ...).
  - `templates/example/` — sources for the example consumer Taskfiles under `examples/` and the root `Taskfile.yml`, wiring the internal libraries above together into runnable demos.
- `config.yml` — declares, per output file, which header/vars/tasks/includes fragments to concatenate (via named `sets:` and/or ad hoc per-file `includes:`/`tasks:` lists), and the output path. This file is **gitignored** (machine-local); `config.yml.example` is the version-controlled template to copy from and keep in sync when adding a new fragment/output file/template root.
- `tasks/_*.yml`, `examples/*/Taskfile.yml`, and the root `Taskfile.yml` — **generated output**, produced by `task generate-tasks`, which runs `find examples/ -mindepth 1 -delete` and `find tasks/ -mindepth 1 -delete` followed by the external `gotasky` binary reading `config.yml` + `templates/`. These generated files are what actually gets published to GitHub and fetched by consumers' `includes:`.
  - Note: `gotasky` itself is not available in this environment, so in practice both a `templates/**/*.yml` source and its corresponding generated `tasks/_*.yml` / `examples/*/Taskfile.yml` output must currently be hand-edited in parallel to keep them in sync.
- `tasks/_cleanup.yml`, `_go.yml`, `_os.yml`, `_time.yml`, `_uv.yml` — the internal (`internal: true`) task libraries, namespaced by whatever alias a consumer's `includes:` gives them (conventionally `_cleanup`, `_go`, `_os`, `_time`, `_uv`).
- `examples/cleanup/`, `go/`, `os/`, `uv/` (each a `Taskfile.yml`) — example consumer Taskfiles showing how to wire up one or more of the internal libraries above (aliased short commands like `c`, `gb`, `ou`, `us`, ...).

## Commands

There is no application build/lint/test toolchain here (no `go.mod`, `package.json`, etc.) — the only "build" is regenerating the Taskfiles themselves:

- `task` / `task --list` — list available top-level tasks
- `task generate-tasks` (alias `g`) — regenerate `tasks/_*.yml`, `examples/*/Taskfile.yml`, and the root `Taskfile.yml` from `templates/` + `config.yml` via `gotasky`
- `task cleanup` (alias `c`) — delete `.idea`/`.terraform`/`.vscode`/OS junk files, then force-refresh (`--download --force`) this project's cached remote Taskfiles in `.task/remote/`

## Conventions and gotchas specific to this codebase

These came up repeatedly while debugging tasks in this repo and are easy to get wrong because the failure only shows up at `task` runtime, not at edit time:

- **`defer:` must come *before* the command it's meant to protect, not after.** go-task's `defer` behaves like Go's `defer`: it only guards against failures in commands listed *after* it in the same `cmds:` block. Every consumer task in this repo follows the pattern
  ```yaml
  cmds:
    - defer: { task: _time:end, vars: { START_TIME: '{{.START_TIME}}' } }
    - task: _go:build
  ```
  so that the elapsed-time display still runs (and the task still correctly reports failure) even when the real command fails. The old order (`task: _go:build` then `defer:`) silently skips the deferred step on any failure.
- **A parent Taskfile's `vars:` are not visible inside an `includes:`'d file**, especially a remote one — go-task does not propagate them across the include boundary. Use go-task's built-in `{{.ROOT_DIR}}` (root Taskfile's directory) and `{{.USER_WORKING_DIR}}` (invoking cwd) instead of inventing custom passthrough vars.
- **`vars:` are template-substitution only** (`{{.VAR}}`) and are *not* exported as OS environment variables to `cmds:` or nested `task` subprocesses. Use `env:` when a shell command needs a real environment variable (e.g. `TASK_X_REMOTE_TASKFILES: 1` to enable the remote-taskfiles experiment for a nested `task` invocation).
- **`dir:` on a task applies to `preconditions.sh` as well as `cmds:`**, so prefer setting `dir: "{{.ROOT_DIR}}"` once at the task level over repeating `{{.ROOT_DIR}}/...` in every command.
- The `_cleanup:refresh` task forces a remote-cache refresh via a nested `task --yes --download --force --list-all > /dev/null 2>&1` run from `{{.ROOT_DIR}}`. `--list-all` is used (instead of no task name) so it doesn't depend on the consuming project having a `default` task; both stdout and stderr are redirected because go-task's trust-confirmation and list output are not reliably confined to stdout across versions.
- **`_os.yml`'s per-distro tasks (`check:*`, `update:*`, `remove:*`) use `status:` as a guard, not a real up-to-date check.** Each variant's `status:` shell condition is true (skip) when the detected OS/distro doesn't match, so `task _os:check` fans out to `check:amazon-linux`, `check:macos`, and `check:ubuntu` and only the matching one actually runs.
- **`_go.yml`'s `VERSION`/`RELEASE_VERSION`/`MODULE`/`VERSION_PKG` vars assume a consumer's own Go module** (`GITHUB_USER`/`GITHUB_REPO`/`BINARY_NAME` provided via that consumer's `vars:` or dotenv, e.g. `examples/go/Taskfile.yml`'s `project.ini`) — they're meaningless when testing `_go.yml` in isolation without those set.
