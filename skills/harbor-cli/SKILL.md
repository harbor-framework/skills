---
name: harbor-cli
description: "Harbor CLI command reference and usage patterns. Covers harbor run, harbor jobs, harbor trials, harbor datasets, harbor adapters, harbor tasks, harbor view, harbor sweeps, harbor traces, harbor cache, and harbor admin commands. Use this skill whenever running Harbor evaluations, managing datasets, viewing results, debugging tasks, exporting traces, or working with any harbor CLI command. Also use when constructing harbor command lines, looking up flag names, or troubleshooting CLI errors."
---

# Harbor CLI Reference

Harbor CLI manages the full evaluation lifecycle: creating tasks, running agents, viewing results. Install with `uv tool install harbor`. Global option: `--version` / `-v`.

For complete flag tables with types and defaults for every command, read `references/flags.md`.

## Quick Reference

| Command | Description |
|---------|-------------|
| `harbor run` | Run evaluations (alias for `harbor jobs start`) |
| `harbor jobs start` | Start an evaluation job |
| `harbor jobs resume` | Resume an interrupted job |
| `harbor jobs summarize` | AI-powered failure summaries for a job |
| `harbor trials start` | Run a single trial (debugging) |
| `harbor trials summarize` | AI-powered summary of a single trial |
| `harbor datasets list` | List available datasets |
| `harbor datasets download` | Download a dataset |
| `harbor adapters init` | Scaffold a new benchmark adapter |
| `harbor adapters validate` | Validate an adapter |
| `harbor tasks init` | Scaffold a new task |
| `harbor tasks check` | AI quality assessment of a task |
| `harbor tasks start-env` | Launch task environment interactively |
| `harbor tasks debug` | Analyze failing trials for a task |
| `harbor tasks migrate` | Convert Terminal-Bench tasks to Harbor |
| `harbor view` | Browse job/trial results in web UI |
| `harbor sweeps run` | Run successive evaluation sweeps |
| `harbor traces export` | Export trace data in ATIF format |
| `harbor cache clean` | Clean Docker images and cache directory |
| `harbor admin` | Administrative commands (hidden) |

## harbor run / harbor jobs start

The primary command. `harbor run` is an alias for `harbor jobs start`.

**Defaults:** agent = `oracle`, n-concurrent = `4`, environment = `docker`, output = `./jobs`.

**Essential flags:**

| Flag | Short | Description |
|------|-------|-------------|
| `--path` | `-p` | Local path to task directory |
| `--dataset` | `-d` | Dataset from registry (e.g., `terminal-bench@2.0`) |
| `--config` | `-c` | Job config YAML/JSON file |
| `--agent` | `-a` | Agent name (default: `oracle`). Repeatable |
| `--model` | `-m` | Model identifier. Repeatable |
| `--n-concurrent` | `-n` | Concurrent trials (default: `4`) |
| `--env` | `-e` | Environment backend (default: `docker`) |
| `--jobs-dir` | `-o` | Output directory (default: `./jobs`) |
| `--agent-env` | `--ae` | Pass env var to agent: `KEY=VALUE`. Repeatable |
| `--agent-kwarg` | `--ak` | Agent kwarg: `key=value`. Repeatable |
| `--task-name` | `-t` | Include tasks by glob pattern. Repeatable |
| `--exclude-task-name` | `-x` | Exclude tasks by glob. Repeatable |
| `--n-tasks` | `-l` | Max tasks to run |
| `--quiet` | `-q` | Suppress trial progress |
| `--debug` | | Enable debug logging |
| `--dry-run` | | Print config and exit |
| `--disable-verification` | | Skip running tests |
| `--no-cache` | | Skip Docker build cache |
| `--artifact` | | Download path from env after trial. Repeatable |
| `--timeout-multiplier` | | Scale all timeouts (default: `1.0`) |

Additional flags for environment kwargs (`--ek`), environment env vars (`--ee`), verifier kwargs (`--vk`), orchestrator kwargs (`--ok`), agent images, retry config, trace export, and per-phase timeout multipliers are in `references/flags.md`.

### Examples

```bash
# Run local task with oracle (validates solution + tests)
harbor run -p ./my-task

# Run with a real agent
harbor run -p ./my-task -a claude-code -m anthropic/claude-sonnet-4-1

# Dataset evaluation with concurrency
harbor run -d terminal-bench@2.0 -a claude-code -m anthropic/claude-sonnet-4-1 -n 8

# Cloud environment
harbor run -d my-dataset -a claude-code -m anthropic/claude-sonnet-4-1 -e daytona

# From config file
harbor run -c eval-config.yaml

# Pass API key to agent
harbor run -p ./my-task -a claude-code -m anthropic/claude-sonnet-4-1 \
  --ae ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY

# Run subset of tasks
harbor run -d terminal-bench@2.0 -a claude-code -m anthropic/claude-sonnet-4-1 \
  -t "bash-*" -l 10

# Dry run
harbor run -d terminal-bench@2.0 -a claude-code -m anthropic/claude-sonnet-4-1 --dry-run
```

## harbor jobs resume

Resume an interrupted or partially completed job.

```bash
harbor jobs resume -p ./jobs/my-job-2025-01-15
harbor jobs resume -p ./jobs/my-job -f AgentTimeoutError
```

| Flag | Short | Description |
|------|-------|-------------|
| `--job-path` | `-p` | Path to job directory with `config.json` (required) |
| `--filter-error-type` | `-f` | Remove trials matching this error type before resuming. Repeatable. Default: `CancelledError` |

## harbor jobs summarize

Generate AI-powered failure summaries for trials in a job.

```bash
harbor jobs summarize ./jobs/my-job
harbor jobs summarize ./jobs/my-job -m sonnet --all --overwrite
```

| Flag | Short | Description |
|------|-------|-------------|
| `job_path` | | Path to job dir or parent (positional) |
| `--model` | `-m` | Model: `haiku`, `sonnet`, `opus` (default: `haiku`) |
| `--n-concurrent` | `-n` | Max concurrent queries (default: `5`) |
| `--all` / `--failed` | | Analyze all or only failed trials (default: `--failed`) |
| `--overwrite` | | Overwrite existing `summary.md` files |

## harbor trials start

Run a single trial. Useful for debugging and task development.

**Key difference from `harbor jobs start`:** the environment flag is `--environment-type` (not `--env`), and output goes to `--trials-dir` (default: `./trials`).

| Flag | Short | Description |
|------|-------|-------------|
| `--path` | `-p` | Path to local task directory |
| `--config` | `-c` | Trial config YAML/JSON |
| `--agent` | `-a` | Agent name (default: `oracle`) |
| `--model` | `-m` | Model for the agent |
| `--environment-type` | `-e` | Environment type (default: `docker`) |
| `--trials-dir` | | Output directory (default: `./trials`) |
| `--agent-env` | `--ae` | Env var for agent: `KEY=VALUE`. Repeatable |
| `--agent-kwarg` | `--ak` | Agent kwarg: `key=value`. Repeatable |
| `--no-cleanup` | | Keep environment after trial |
| `--no-verify` | | Skip running tests |

Full flag list (task kwargs, git options, etc.) in `references/flags.md`.

```bash
# Test with oracle (validates solution + tests)
harbor trials start -p ./my-task

# Test with real agent
harbor trials start -p ./my-task -a claude-code -m anthropic/claude-sonnet-4-1

# Keep container for inspection
harbor trials start -p ./my-task -a claude-code -m anthropic/claude-sonnet-4-1 --no-cleanup
```

The oracle agent runs `solution/solve.sh` inside the environment, then the verifier runs `tests/test.sh`. If oracle does not get reward 1.0, your tests or solution have a bug.

## harbor trials summarize

```bash
harbor trials summarize ./trials/my-trial
harbor trials summarize ./trials/my-trial -m sonnet --overwrite
```

| Flag | Short | Description |
|------|-------|-------------|
| `trial_path` | | Path to trial directory (positional) |
| `--model` | `-m` | Model: `haiku`, `sonnet`, `opus` (default: `haiku`) |
| `--overwrite` | | Overwrite existing `summary.md` |

## harbor datasets

### harbor datasets list

```bash
harbor datasets list
harbor datasets list --registry-url https://custom-registry.example.com
harbor datasets list --registry-path ./local-registry.json
```

Flags `--registry-url` and `--registry-path` are mutually exclusive. Default: Harbor's public registry.

### harbor datasets download

```bash
harbor datasets download terminal-bench@2.0
harbor datasets download terminal-bench@2.0 -o ./my-tasks --overwrite
```

| Flag | Short | Description |
|------|-------|-------------|
| `DATASET` | | Name or `name@version` (positional) |
| `--output-dir` | `-o` | Download dir (default: `~/.cache/harbor/tasks`) |
| `--overwrite` | | Re-download even if cached |

## harbor adapters

### harbor adapters init

```bash
harbor adapters init my-benchmark
```

Interactive wizard that prompts for benchmark name, adapter ID, class name, description, source URL, and license. Creates the adapter directory with template files.

### harbor adapters validate

```bash
harbor adapters validate adapters/my-benchmark
```

## harbor tasks

### harbor tasks init

```bash
harbor tasks init my-task
harbor tasks init my-task -p ./custom-tasks
```

| Flag | Short | Description |
|------|-------|-------------|
| `name` | | Task name (positional) |
| `--tasks-dir` | `-p` | Output directory (default: `.`) |
| `--no-pytest` | | Skip pytest in test.sh |
| `--no-solution` | | Skip solution directory |
| `--include-canary-strings` | | Add anti-cheating canary strings |
| `--include-standard-metadata` | | Add full metadata template to task.toml |

### harbor tasks check

AI-powered quality assessment against a rubric.

```bash
harbor tasks check ./my-task
harbor tasks check ./my-task -m opus -o results.json
harbor tasks check ./my-task -r custom-rubric.toml
```

| Flag | Short | Description |
|------|-------|-------------|
| `task` | | Task name or path (positional) |
| `--model` | `-m` | Model: `sonnet`, `opus`, `haiku` (default: `sonnet`) |
| `--output-path` | `-o` | Write JSON results here |
| `--rubric_path` | `-r` | Custom rubric (`.toml`, `.yaml`, `.yml`, `.json`) |

### harbor tasks start-env

Launch task environment interactively for manual inspection.

```bash
harbor tasks start-env -p ./my-task
harbor tasks start-env -p ./my-task -e daytona
harbor tasks start-env -p ./my-task --agent claude-code -m anthropic/claude-sonnet-4-1
```

| Flag | Short | Description |
|------|-------|-------------|
| `--path` | `-p` | Path to task directory |
| `--env` | `-e` | Environment type (default: `docker`) |
| `--all` | `-a` | Include solution and tests (default: `true`) |
| `--interactive` / `--non-interactive` | `-i` | Interactive shell (default: `true`) |
| `--agent` | | Agent to install in environment |
| `--model` | `-m` | Model for the agent |

Full flag list in `references/flags.md`.

### harbor tasks debug

Analyze failing trials for a task using AI.

```bash
harbor tasks debug my-task-id -m sonnet
harbor tasks debug my-task-id --job-id my-job --jobs-dir ./jobs
```

| Flag | Short | Description |
|------|-------|-------------|
| `task_id` | | Task ID (positional) |
| `--model` | `-m` | Model for analysis |
| `--job-id` | | Specific job to analyze |
| `--jobs-dir` | | Directory containing jobs |

### harbor tasks migrate

Convert Terminal-Bench tasks to Harbor format.

```bash
harbor tasks migrate -i ./old-tasks -o ./harbor-tasks
harbor tasks migrate -i ./old-tasks -o ./harbor-tasks --cpus 2 --memory-mb 4096
```

| Flag | Short | Description |
|------|-------|-------------|
| `--input` | `-i` | Terminal-Bench task dir or parent |
| `--output` | `-o` | Output directory |
| `--cpus` | | Override CPUs for all tasks |
| `--memory-mb` | | Override memory (MB) |
| `--storage-mb` | | Override storage (MB) |
| `--gpus` | | Override GPUs |

## harbor view

Browse job/trial results in a web UI.

```bash
harbor view ./jobs/my-job
harbor view ./jobs --port 9000
harbor view ./jobs --dev
```

| Flag | Short | Description |
|------|-------|-------------|
| `folder` | | Directory with trajectories (positional) |
| `--port` | `-p` | Port or range (default: `8080-8089`) |
| `--host` | | Bind host (default: `127.0.0.1`) |
| `--dev` | | Hot-reload development mode |
| `--build` | | Force rebuild viewer |
| `--no-build` | | Skip auto-build |

Shows job summaries, per-trial reward values, agent trajectories, and test output.

## harbor sweeps run

Run successive sweeps. Each sweep drops tasks that already have at least one success, focusing effort on remaining failures.

```bash
harbor sweeps run -c sweep-config.yaml
harbor sweeps run -c sweep-config.yaml --max-sweeps 5 --trials-per-task 3
harbor sweeps run -c sweep-config.yaml --hint "Try grep to find the file first"
```

| Flag | Short | Description |
|------|-------|-------------|
| `--config` | `-c` | Job config file (YAML/JSON) |
| `--max-sweeps` | | Max sweeps (default: `3`) |
| `--trials-per-task` | | Trials per task per sweep (default: `2`) |
| `--hint` | | Hint string passed to agent kwargs |
| `--hints-file` | | JSON mapping task name to hint |
| `--push` / `--no-push` | | Push exported datasets to HF Hub |

Additional export flags in `references/flags.md`.

## harbor traces export

Export ATIF trajectories as training datasets.

```bash
harbor traces export -p ./jobs/my-job
harbor traces export -p ./trials --filter success --push --repo my-org/traces
harbor traces export -p ./jobs/my-job --sharegpt --episodes last
```

| Flag | Short | Description |
|------|-------|-------------|
| `--path` | `-p` | Path to trial dir or root containing trials |
| `--recursive` / `--no-recursive` | | Search recursively (default: recursive) |
| `--episodes` | | `all` or `last` per trial (default: `all`) |
| `--filter` | | `success`, `failure`, or `all` (default: `all`) |
| `--sharegpt` / `--no-sharegpt` | | ShareGPT-formatted conversations |
| `--push` / `--no-push` | | Push to Hugging Face Hub |
| `--repo` | | HF repo id (`org/name`) |

Additional flags (`--subagents`, `--instruction-metadata`, `--verifier-metadata`, `--verbose`) in `references/flags.md`.

## harbor cache clean

Remove Harbor Docker images and `~/.cache/harbor`.

```bash
harbor cache clean
harbor cache clean --force
harbor cache clean --dry
harbor cache clean --no-docker      # Only clean ~/.cache/harbor
harbor cache clean --no-cache-dir   # Only clean Docker images
```

| Flag | Short | Description |
|------|-------|-------------|
| `--force` | `-f` | Skip confirmation |
| `--dry` | | Preview without deleting |
| `--no-docker` | | Skip Docker image removal |
| `--no-cache-dir` | | Skip `~/.cache/harbor` removal |

## Supported Agents

| Agent | CLI value | Notes |
|-------|-----------|-------|
| Oracle | `oracle` | Runs `solution/solve.sh`. **Default agent** |
| Nop | `nop` | Does nothing (baseline) |
| Claude Code | `claude-code` | Needs `ANTHROPIC_API_KEY` |
| Aider | `aider` | |
| Codex | `codex` | |
| Cline CLI | `cline-cli` | Model format: `provider:model-id` |
| Cursor CLI | `cursor-cli` | |
| Gemini CLI | `gemini-cli` | Needs `GOOGLE_API_KEY` or `GEMINI_API_KEY` |
| Goose | `goose` | Model format: `provider/model_name` |
| Mini SWE-agent | `mini-swe-agent` | |
| SWE-agent | `swe-agent` | |
| OpenCode | `opencode` | |
| OpenHands | `openhands` | Needs `LLM_API_KEY` and `LLM_MODEL` |
| OpenHands SDK | `openhands-sdk` | |
| Qwen Coder | `qwen-coder` | |
| Terminus | `terminus` | Harbor's built-in agent |
| Terminus 1 | `terminus-1` | |
| Terminus 2 | `terminus-2` | |

## Environment Backends

| Backend | CLI value | Description |
|---------|-----------|-------------|
| Docker | `docker` | Default. Local containers |
| Daytona | `daytona` | Cloud sandbox with Docker-in-Docker |
| E2B | `e2b` | E2B cloud sandboxes |
| GKE | `gke` | Google Kubernetes Engine |
| Modal | `modal` | Serverless with GPU support |
| Runloop | `runloop` | Runloop cloud environments |

Note: `harbor run` uses `--env` / `-e`. `harbor trials start` uses `--environment-type` / `-e`.

## Common Workflows

### Create and test a new task

```bash
harbor tasks init my-task
# Edit: task.toml, instruction.md, Dockerfile, tests/test.sh, solution/solve.sh
harbor tasks check ./my-task
harbor trials start -p ./my-task                    # oracle validates tests
harbor trials start -p ./my-task -a claude-code -m anthropic/claude-sonnet-4-1
harbor view ./trials
```

### Run a benchmark evaluation

```bash
harbor datasets list
harbor datasets download terminal-bench@2.0
harbor run -d terminal-bench@2.0 -a claude-code -m anthropic/claude-sonnet-4-1 -n 8
harbor view ./jobs
harbor traces export -p ./jobs/my-job --push --repo my-org/traces
```

### Debug a failing task

```bash
harbor tasks start-env -p ./my-task         # interactive shell in the container
# Inside: bash /tests/test.sh && bash /solution/solve.sh
harbor tasks debug my-task-id -m sonnet     # AI analysis of failures
harbor trials start -p ./my-task            # re-test with oracle
```

### Resume a failed job

```bash
harbor jobs resume -p ./jobs/my-job-2025-01-15
harbor jobs resume -p ./jobs/my-job -f AgentTimeoutError
harbor jobs summarize ./jobs/my-job
```

## Common Gotchas

**API keys:** Most agents need keys passed via `--ae`. Claude Code needs `ANTHROPIC_API_KEY`, OpenHands needs `LLM_API_KEY`, Goose needs provider-specific keys. If the agent fails immediately, check the key.

**Docker must be running.** Harbor uses Docker for sandboxed environments. Network exhaustion from many concurrent trials can cause failures -- `harbor cache clean` helps.

**Model name format varies by agent.** Claude Code uses `anthropic/claude-sonnet-4-1`, Cline CLI uses `provider:model-id` (e.g., `openrouter:anthropic/claude-opus-4.5`), Goose uses `provider/model_name`. A `ValueError` about model format usually means the wrong format for that agent.

**`--env` vs `--environment-type`:** `harbor run` uses `--env` / `-e`. `harbor trials start` uses `--environment-type` / `-e`. Same short flag, different long name.

**Default agent is oracle.** Forgetting `-a` gives you the oracle agent, which just runs `solve.sh`. Fine for task validation, not for real evaluations.

**`--no-cleanup` for debugging.** Pass this to `harbor trials start` to keep the container running after a failed trial so you can inspect it.

**Retry defaults skip common errors.** `--retry-exclude` defaults to `AgentTimeoutError`, `VerifierTimeoutError`, `RewardFileNotFoundError`, `RewardFileEmptyError`, `VerifierOutputParseError`. These usually indicate task bugs, not transient failures.
