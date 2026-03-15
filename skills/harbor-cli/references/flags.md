# Harbor CLI Complete Flag Reference

Detailed flags for every Harbor CLI command. This file is the authoritative reference for flag names, short forms, types, and defaults.

## Table of Contents

- [harbor run / harbor jobs start](#harbor-run--harbor-jobs-start)
- [harbor jobs resume](#harbor-jobs-resume)
- [harbor jobs summarize](#harbor-jobs-summarize)
- [harbor trials start](#harbor-trials-start)
- [harbor trials summarize](#harbor-trials-summarize)
- [harbor datasets list](#harbor-datasets-list)
- [harbor datasets download](#harbor-datasets-download)
- [harbor tasks init](#harbor-tasks-init)
- [harbor tasks check](#harbor-tasks-check)
- [harbor tasks start-env](#harbor-tasks-start-env)
- [harbor tasks debug](#harbor-tasks-debug)
- [harbor tasks migrate](#harbor-tasks-migrate)
- [harbor view](#harbor-view)
- [harbor sweeps run](#harbor-sweeps-run)
- [harbor traces export](#harbor-traces-export)
- [harbor cache clean](#harbor-cache-clean)

---

## harbor run / harbor jobs start

`harbor run` is an alias for `harbor jobs start`.

**Defaults:** agent = `oracle`, n-concurrent = `4`, orchestrator = `local`, environment = `docker`, jobs-dir = `./jobs`.

### Essential flags

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--path` | `-p` | Path | | Local path to a task directory |
| `--dataset` | `-d` | str | | Dataset name from registry (e.g., `terminal-bench@2.0`) |
| `--config` | `-c` | Path | | Path to job config YAML/JSON (`JobConfig` schema) |
| `--agent` | `-a` | str | `oracle` | Agent name. Can repeat for multiple agents |
| `--model` | `-m` | str list | | Model for the agent. Can repeat for multiple models |
| `--n-concurrent` | `-n` | int | `4` | Number of concurrent trials |
| `--env` | `-e` | str | `docker` | Environment backend |
| `--jobs-dir` | `-o` | Path | `./jobs` | Output directory for job results |

### Agent flags

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--agent-env` | `--ae` | str list | | Env var for agent as `KEY=VALUE`. Can repeat |
| `--agent-kwarg` | `--ak` | str list | | Agent kwarg as `key=value`. Can repeat |
| `--agent-import-path` | | str | | Python import path for custom agent class |
| `--agent-config-path` | | Path | | Path to custom agent config file |
| `--agent-image` | | str | | Docker image for the agent |
| `--agent-name-override` | | str | | Override agent name in results |

### Environment flags

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--environment-import-path` | | str | | Python import path for custom environment |
| `--environment-image` | | str | | Docker image for the environment |
| `--environment-cpus` | | int | | CPU count for environment |
| `--environment-memory-mb` | | int | | Memory in MB |
| `--environment-storage-mb` | | int | | Storage in MB |
| `--environment-gpus` | | int | | GPU count |
| `--environment-network-mode` | | str | | Docker network mode |
| `--environment-kwarg` | `--ek` | str list | | Environment kwarg as `key=value`. Can repeat |
| `--environment-env` | `--ee` | str list | | Env var for environment as `KEY=VALUE`. Can repeat |
| `--environment-name-override` | | str | | Override environment name in results |

### Verifier flags

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--verifier-import-path` | | str | | Python import path for custom verifier |
| `--verifier-kwarg` | `--vk` | str list | | Verifier kwarg as `key=value`. Can repeat |

### Job control flags

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--job-name` | | str | auto | Custom job name |
| `--n-attempts` | `-k` | int | | Number of attempts per trial |
| `--max-retries` | `-r` | int | `0` | Max retry attempts on failure |
| `--retry-include` | | str list | | Exception types to retry on. Can repeat |
| `--retry-exclude` | | str list | see below | Exception types to NOT retry on. Can repeat |
| `--task-name` | `-t` | str list | | Include specific tasks (glob patterns). Can repeat |
| `--exclude-task-name` | `-x` | str list | | Exclude tasks (glob patterns). Can repeat |
| `--n-tasks` | `-l` | int | | Max tasks to run (applied after filters) |
| `--shuffle` | | bool | `false` | Randomize task order |
| `--seed` | | int | | Random seed for shuffling |
| `--orchestrator` | | str | `local` | Orchestrator type: `local` or `queue` |
| `--orchestrator-kwarg` | `--ok` | str list | | Orchestrator kwarg as `key=value`. Can repeat |

Default `--retry-exclude`: `AgentTimeoutError`, `VerifierTimeoutError`, `RewardFileNotFoundError`, `RewardFileEmptyError`, `VerifierOutputParseError`.

### Timeout multipliers

All default to `1.0`. Specific multipliers override the general `--timeout-multiplier`.

| Flag | Type | Description |
|------|------|-------------|
| `--timeout-multiplier` | float | General multiplier for all timeouts |
| `--agent-timeout-multiplier` | float | Agent execution timeout |
| `--verifier-timeout-multiplier` | float | Verifier (test) timeout |
| `--agent-setup-timeout-multiplier` | float | Agent setup timeout |
| `--environment-build-timeout-multiplier` | float | Docker build timeout |
| `--environment-start-timeout-multiplier` | float | Environment start timeout |

### Behavioral flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--disable-verification` | bool | `false` | Skip running tests after agent completes |
| `--no-cache` | bool | `false` | Disable Docker build cache |
| `--no-pull` | bool | `false` | Don't pull Docker images |
| `--no-rm` | bool | `false` | Don't remove containers after trial |
| `--no-stream-logs` | bool | `false` | Don't stream logs to console |
| `--no-color` | bool | `false` | Disable rich console colors |
| `--artifact` | str list | | Environment path to download as artifact. Can repeat |
| `--quiet` | bool | `false` | Suppress trial progress displays. Short: `-q` |
| `--debug` | bool | `false` | Enable debug logging |
| `--dry-run` | bool | `false` | Print job config and exit |
| `--pdb` | bool | `false` | Drop into debugger on exception |
| `--profile` | bool | `false` | Enable profiling |

### Trace export flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--export-traces` / `--no-export-traces` | bool | `false` | Export traces after job completes |
| `--sharegpt` / `--no-sharegpt` | bool | `false` | Emit ShareGPT-formatted conversations |
| `--export-repo` | str | | HF repo id for pushing traces |

### Registry flags

| Flag | Type | Description |
|------|------|-------------|
| `--registry-url` | str | Remote registry URL |
| `--registry-path` | Path | Local registry path |

---

## harbor jobs resume

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--job-path` | `-p` | Path | | Path to job directory with `config.json` (required) |
| `--filter-error-type` | `-f` | str list | `CancelledError` | Remove trials with this error type before resuming. Can repeat |

---

## harbor jobs summarize

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `job_path` | | Path | | Path to job dir or parent (positional arg) |
| `--model` | `-m` | str | `haiku` | Model: `haiku`, `sonnet`, `opus` |
| `--n-concurrent` | `-n` | int | `5` | Max concurrent summarization queries |
| `--all` / `--failed` | | bool | `--failed` | Analyze all trials or only failed |
| `--overwrite` | | bool | `false` | Overwrite existing `summary.md` files |

---

## harbor trials start

Shares many flags with `harbor jobs start` but operates on a single task. Key difference: environment flag is `--environment-type` (not `--env`), output goes to `--trials-dir`.

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--path` | `-p` | Path | | Path to local task directory |
| `--config` | `-c` | Path | | Trial config YAML/JSON (`TrialConfig` schema) |
| `--agent` | `-a` | str | `oracle` | Agent name |
| `--model` | `-m` | str | | Model for the agent |
| `--environment-type` | `-e` | str | `docker` | Environment type |
| `--trial-name` | | str | auto | Custom trial name |
| `--trials-dir` | | Path | `./trials` | Output directory |
| `--agent-env` | `--ae` | str list | | Env var for agent as `KEY=VALUE`. Can repeat |
| `--agent-kwarg` | `--ak` | str list | | Agent kwarg as `key=value`. Can repeat |
| `--agent-import-path` | | str | | Custom agent import path |
| `--environment-import-path` | | str | | Custom environment import path |
| `--environment-kwarg` | `--ek` | str list | | Environment kwarg as `key=value`. Can repeat |
| `--task-git-url` | | str | | Git URL for remote task repo |
| `--task-git-commit-id` | | str | | Git commit for remote task |
| `--task-name` | | str | | Name of the task |
| `--task-kwarg` | `--tk` | str list | | Task kwarg as `key=value`. Can repeat |
| `--no-cleanup` | | bool | `false` | Keep environment after trial |
| `--no-cache` | | bool | `false` | Skip Docker build cache |
| `--no-pull` | | bool | `false` | Don't pull Docker images |
| `--no-build` | | bool | `false` | Don't build Docker images |
| `--no-verify` | | bool | `false` | Skip running verifier/tests |
| `--no-trajectory` | | bool | `false` | Don't record trajectory |
| `--disable-verification` | | bool | `false` | Skip verification step |
| Timeout multipliers | | | | Same as `harbor jobs start` |

---

## harbor trials summarize

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `trial_path` | | Path | | Path to trial directory (positional arg) |
| `--model` | `-m` | str | `haiku` | Model: `haiku`, `sonnet`, `opus` |
| `--overwrite` | | bool | `false` | Overwrite existing `summary.md` |

---

## harbor datasets list

| Flag | Type | Description |
|------|------|-------------|
| `--registry-url` | str | URL of remote `registry.json` |
| `--registry-path` | Path | Path to local `registry.json` |

Mutually exclusive. Default: Harbor's public registry.

---

## harbor datasets download

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `DATASET` | | str | | Dataset as `name` or `name@version` (positional arg) |
| `--output-dir` | `-o` | Path | `~/.cache/harbor/tasks` | Download directory |
| `--overwrite` | | bool | `false` | Re-download even if cached |
| `--registry-url` | | str | | Remote registry URL |
| `--registry-path` | | Path | | Local registry path |

---

## harbor tasks init

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `name` | | str | | Task name (positional arg) |
| `--tasks-dir` | `-p` | Path | `.` | Output directory |
| `--no-pytest` | | bool | `false` | Skip pytest scaffolding in test.sh |
| `--no-solution` | | bool | `false` | Skip solution directory |
| `--include-canary-strings` | | bool | `false` | Add canary strings for anti-cheating |
| `--include-standard-metadata` | | bool | `false` | Add full metadata template to task.toml |

---

## harbor tasks check

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `task` | | str | | Task name or path (positional arg) |
| `--model` | `-m` | str | `sonnet` | Claude model: `sonnet`, `opus`, `haiku` |
| `--output-path` | `-o` | Path | | Write JSON results to this path |
| `--rubric_path` | `-r` | Path | | Custom rubric file (`.toml`, `.yaml`, `.yml`, `.json`) |

---

## harbor tasks start-env

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--path` | `-p` | Path | | Path to task directory |
| `--env` | `-e` | str | `docker` | Environment type |
| `--all` | `-a` | bool | `true` | Add solution and tests to environment |
| `--interactive` / `--non-interactive` | `-i` | bool | `true` | Interactive mode |
| `--agent` | | str | | Agent to install in environment |
| `--model` | `-m` | str | | Model for the agent |
| `--agent-import-path` | | str | | Custom agent import path |
| `--agent-kwarg` | `--ak` | str list | | Agent kwarg as `key=value` |
| `--environment-import-path` | | str | | Custom environment import path |
| `--environment-kwarg` | `--ek` | str list | | Environment kwarg as `key=value` |

---

## harbor tasks debug

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `task_id` | | str | | Task ID to analyze (positional arg) |
| `--model` | `-m` | str | | Model for analysis |
| `--job-id` | | str | | Specific job ID to analyze |
| `--jobs-dir` | | Path | | Directory containing jobs |

---

## harbor tasks migrate

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--input` | `-i` | Path | | Terminal-Bench task directory or parent |
| `--output` | `-o` | Path | | Output directory for Harbor format |
| `--cpus` | | int | | Override CPU count for all tasks |
| `--memory-mb` | | int | | Override memory (MB) for all tasks |
| `--storage-mb` | | int | | Override storage (MB) for all tasks |
| `--gpus` | | int | | Override GPU count for all tasks |

---

## harbor view

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `folder` | | Path | | Directory with trajectories (positional arg) |
| `--port` | `-p` | str | `8080-8089` | Port or range |
| `--host` | | str | `127.0.0.1` | Host to bind to |
| `--dev` | | bool | `false` | Development mode with hot reloading |
| `--build` | | bool | `false` | Force rebuild viewer |
| `--no-build` | | bool | `false` | Skip auto-building |

---

## harbor sweeps run

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--config` | `-c` | Path | | Job config file (YAML/JSON) |
| `--max-sweeps` | | int | `3` | Max number of sweeps |
| `--trials-per-task` | | int | `2` | Trials per task per sweep |
| `--hint` | | str | | Generic hint string for agent kwargs |
| `--hints-file` | | Path | | JSON mapping task name to hint string |
| `--export-repo` | | str | | HF repo for DatasetDict |
| `--export-repo-success` | | str | | HF repo for successes only |
| `--export-repo-failure` | | str | | HF repo for failures only |
| `--push` / `--no-push` | | bool | `false` | Push to HF Hub |
| `--export-splits` / `--export-separate` | | bool | `true` | One repo with splits vs two repos |

---

## harbor cache clean

| Flag | Short | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--force` | `-f` | bool | `false` | Skip confirmation prompt |
| `--dry` | | bool | `false` | Preview without deleting |
| `--no-docker` | | bool | `false` | Skip Docker image removal |
| `--no-cache-dir` | | bool | `false` | Skip `~/.cache/harbor` removal |
