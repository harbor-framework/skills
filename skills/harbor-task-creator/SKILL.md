---
name: harbor-task-creator
description: "Create Harbor evaluation tasks from scratch. Generates task.toml configuration, instruction.md for agents, environment/Dockerfile setup, tests/test.sh verification scripts, and solution/solve.sh reference solutions. Use this skill whenever creating, scaffolding, or authoring new Harbor benchmark tasks, evaluation environments, or agent challenges. Also use when fixing broken tasks, debugging reward file issues, or structuring multi-container evaluation environments."
---

# Creating Harbor Evaluation Tasks

Harbor tasks are self-contained evaluation challenges for AI agents. Each task provides an instruction, a sandboxed Docker environment, automated tests, and optionally a reference solution. Understanding how Harbor wires these pieces together at runtime is key to writing tasks that work correctly.

## Quick Start

Scaffold a new task with the CLI:

```bash
harbor tasks init my-task
# Flags:
#   --tasks-dir (-p)              Parent directory (default: current dir)
#   --no-pytest                   Use plain bash tests instead of pytest template
#   --no-solution                 Skip generating solution/solve.sh
#   --include-canary-strings      Embed tracking GUIDs in generated files
#   --include-standard-metadata   Add Terminal Bench metadata fields to task.toml
```

Or create the directory structure manually:

```
my-task/
├── task.toml              # Configuration (timeouts, resources, metadata)
├── instruction.md         # Natural language prompt for the agent
├── environment/
│   ├── Dockerfile         # Container image definition
│   ├── docker-compose.yaml  # Multi-container setup (optional)
│   └── ...                # Files COPYed into the image
├── tests/
│   ├── test.sh            # Verification entry point (required)
│   └── test_*.py          # Pytest files (optional)
├── solution/
│   └── solve.sh           # Reference solution (optional)
└── README.md              # Human documentation (optional)
```

**Required files:** `task.toml`, `instruction.md`, `environment/` (with Dockerfile or docker-compose.yaml), `tests/test.sh`.

**Validation:** Harbor validates tasks by checking that `task.toml`, `environment/`, `instruction.md`, and `tests/test.sh` all exist (via `TaskPaths.is_valid()`).

Run the AI-powered quality checker after creating a task:

```bash
harbor tasks check ./my-task    # LLM-based rubric assessment
```

## How Harbor Executes a Task (Runtime Model)

Understanding the runtime flow prevents most common authoring mistakes:

1. **Build** — Harbor builds the Docker image from `environment/Dockerfile` (or starts compose services from `environment/docker-compose.yaml`).
2. **Mount** — Three host directories are bind-mounted into the container:
   - `/logs/verifier/` — where test.sh writes reward files and stdout
   - `/logs/agent/` — agent logs
   - `/logs/artifacts/` — any artifacts
3. **Agent runs** — The agent connects to the container, reads instruction.md (provided separately, not in the container), and works in the environment.
4. **Upload tests** — After the agent finishes, Harbor uploads `tests/` to `/tests/` inside the container using `docker compose cp`.
5. **Execute test.sh** — Harbor runs `chmod +x /tests/test.sh` then executes it. Working directory is `/tests/`. Environment variables from `[verifier].env` in task.toml are injected.
6. **Parse reward** — Harbor reads `/logs/verifier/reward.txt` (priority) or `/logs/verifier/reward.json`. If neither exists, the trial errors with `RewardFileNotFoundError`.

This means: the agent never sees `tests/` or `solution/`. Tests run after the agent is done. Reward files must always be written.

## Writing instruction.md

The instruction is the only thing the agent sees. It must be completely self-contained.

**Guidelines:**
- Specify exact file paths the agent should create or modify (e.g., "Write your answer to `/app/output.txt`")
- State the expected output format precisely (JSON schema, line format, etc.)
- Include all necessary context — the agent cannot ask clarifying questions
- Do NOT mention tests, scoring, or evaluation mechanics
- Do NOT leak file paths from `tests/` or `solution/`
- If the task requires structured data output, document the exact schema in the instruction — the quality checker flags this

**Anti-patterns to avoid:**
- "Submit your answer" — agents don't submit; they write files that tests check
- "You may use any approach" — too vague; specify constraints when they matter
- Referencing files outside the environment (agents only see the container)
- Describing behavior in the instruction that isn't actually tested (and vice versa)

**Example:**

```markdown
# Fix the Sorting Bug

The file `/app/sort.py` contains a merge sort implementation with a bug
that causes incorrect results on arrays with duplicate elements.

Fix the bug so that `sort_array()` correctly sorts any list of integers.
The fix should be in `/app/sort.py`. Do not rename the function.
```

## Configuring task.toml

The task.toml file controls timeouts, resources, metadata, and MCP server configuration. It is parsed by `TaskConfig.model_validate_toml()`.

**Minimal example:**

```toml
version = "1.0"

[metadata]
difficulty = "easy"
category = "programming"

[agent]
timeout_sec = 300.0

[verifier]
timeout_sec = 120.0
```

**Full configuration — see [references/task-toml-reference.md](references/task-toml-reference.md) for every field, type, and default.**

Key sections:
- `[metadata]` — free-form dict; conventional keys: difficulty, category, tags, author info, time estimates
- `[agent]` — `timeout_sec` (float, default 600.0): wall-clock seconds for the agent
- `[verifier]` — `timeout_sec` (float, default 600.0) and `env` (dict): timeout and environment variables for test.sh
- `[environment]` — resource limits (cpus, memory_mb, storage_mb, gpus), pre-built images, network access, MCP servers
- `[solution]` — `env` (dict): environment variables for solve.sh
- `[[environment.mcp_servers]]` — MCP server configs (repeated section with `[[double brackets]]`)

**MCP server configuration** (for tasks that provide MCP tools to the agent):

```toml
[[environment.mcp_servers]]
name = "my-mcp-server"
transport = "streamable-http"       # "sse" (default), "streamable-http", or "stdio"
url = "http://mcp-server:8000/mcp"  # Required for sse/streamable-http

[[environment.mcp_servers]]
name = "local-tool"
transport = "stdio"
command = "/usr/local/bin/tool"      # Required for stdio
args = ["--verbose"]
```

A Pydantic validator (`validate_transport_fields`) enforces that `url` is set for HTTP transports and `command` is set for `stdio`.

## Writing environment/Dockerfile

The Dockerfile creates the sandboxed environment where the agent works.

**Base images:** Use `ubuntu:24.04` unless a specific runtime is needed (`python:3.12-slim`, `node:22`).

**Critical rules:**
- DO NOT copy `tests/` or `solution/` into the image — Harbor uploads these at runtime to `/tests/` and `/solution/` respectively
- DO NOT install test-only dependencies (pytest, etc.) — install them in `test.sh`
- Keep images minimal — only what the agent needs to work on the task
- Pin dependency versions for reproducibility (e.g., `pip install flask==3.1.0`, not `pip install flask`)

**Patterns:**

Basic (empty workspace):
```dockerfile
FROM ubuntu:24.04
WORKDIR /app
```

With dependencies:
```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip git curl \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
```

With seed data (files the agent should see/modify):
```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir flask==3.1.0
WORKDIR /app
COPY sort.py /app/sort.py
COPY data/ /app/data/
```

Place any files referenced by `COPY` in the `environment/` directory alongside the Dockerfile.

## Multi-Container Tasks (docker-compose.yaml)

For tasks requiring multiple services (databases, APIs, etc.), add `docker-compose.yaml` to the `environment/` directory. When Harbor detects this file, it switches to "compose mode" and uses `docker compose` to orchestrate all services.

**Critical rule: the primary service must be named `main`.**

Harbor hardcodes the service name `main` for all `exec`, `upload`, and `download` operations. If your primary service has a different name, Harbor cannot interact with it. This is not configurable.

```yaml
services:
  main:                            # MUST be named "main"
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/testdb

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**Other rules:**
- Services communicate by service name (Docker Compose DNS)
- Always include health checks for services the agent depends on — without them, the main container may start before dependencies are ready
- When `allow_internet = false` in task.toml, Harbor appends `network_mode: none` to the main service

## Writing tests/test.sh

The test script runs inside the container AFTER the agent finishes. It determines the task reward.

**Reward contract:**
- Write a float to `/logs/verifier/reward.txt` (e.g., `1` for pass, `0` for fail, `0.5` for partial)
- Alternatively, write JSON to `/logs/verifier/reward.json` (e.g., `{"reward": 1.0}`)
- `reward.txt` takes priority — if both exist, Harbor reads `reward.txt` and ignores `reward.json`
- `reward.json` supports multiple named rewards (e.g., `{"reward": 1.0, "quality": 0.8, "style": 0.6}`)
- MUST write the reward file in ALL code paths — if missing, the trial fails with `RewardFileNotFoundError`; if empty, `RewardFileEmptyError`

**Pattern 1: Pytest + CTRF (recommended for complex tasks)**

```bash
#!/bin/bash
set -uo pipefail

# Install test dependencies (NOT in the Dockerfile)
apt-get update && apt-get install -y curl
curl -LsSf https://astral.sh/uv/0.9.7/install.sh | sh
source $HOME/.local/bin/env

# Run tests with CTRF output for the Harbor Viewer
uvx --with pytest==8.4.1 --with pytest-json-ctrf==0.3.5 \
  pytest --ctrf /logs/verifier/ctrf.json /tests/test_outputs.py -rA || true

EXIT_CODE=${PIPESTATUS[0]}
if [ "$EXIT_CODE" -eq 0 ]; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

Use `set -uo pipefail` (without `-e`) so the script continues to write the reward file even if pytest fails. Use `|| true` after pytest and check `${PIPESTATUS[0]}` for the actual exit code.

CTRF (`ctrf.json`) is displayed in the Harbor Viewer UI for detailed per-test inspection. It is NOT used for reward calculation — the reward always comes from `reward.txt` or `reward.json`.

**Pattern 2: Simple file check (for straightforward tasks)**

```bash
#!/bin/bash
if [ -f /app/output.txt ] && grep -q "expected content" /app/output.txt; then
  echo 1 > /logs/verifier/reward.txt
else
  echo 0 > /logs/verifier/reward.txt
fi
```

**Pattern 3: Partial credit with multiple named rewards**

```bash
#!/bin/bash
SCORE=0
TOTAL=3

[ -f /app/part1.txt ] && grep -q "correct" /app/part1.txt && SCORE=$((SCORE + 1))
[ -f /app/part2.txt ] && grep -q "correct" /app/part2.txt && SCORE=$((SCORE + 1))
[ -f /app/part3.txt ] && grep -q "correct" /app/part3.txt && SCORE=$((SCORE + 1))

# Simple reward
echo "scale=2; $SCORE / $TOTAL" | bc > /logs/verifier/reward.txt

# Or use reward.json for named sub-scores (pick one approach, not both)
# echo "{\"reward\": $(echo "scale=2; $SCORE/$TOTAL" | bc), \"completeness\": $SCORE}" \
#   > /logs/verifier/reward.json
```

**Key rules:**
- Working directory when test.sh runs is `/tests/`
- Install test dependencies in test.sh, NOT in the Dockerfile
- Always write the reward file, even on error
- Use CTRF output (`--ctrf`) for pytest — Harbor displays it in the Viewer for human inspection
- Pin all dependency versions in test.sh (e.g., `pytest==8.4.1`)

## Writing solution/solve.sh

The reference solution proves the task is solvable and validates the tests. The Oracle Agent uploads the entire `solution/` directory to `/solution/` in the container, then runs `solve.sh`.

```bash
#!/bin/bash
# Reference solution — performs real computation, not hardcoded answers
cd /app
python3 sort.py --fix
```

**Rules:**
- Must perform real computation (not just `echo "42" > /app/answer.txt`)
- Should work with the same environment the agent sees
- Validate with: `harbor trials start --path ./my-task --agent oracle`
- Environment variables from `[solution].env` in task.toml are injected (plus `DEBIAN_FRONTEND=noninteractive`)

## Quality Checklist

Harbor's `QualityChecker` uses an 11-point rubric (defined in `default_rubric.toml`). Each criterion is evaluated by an LLM judge as pass/fail/not_applicable. Verify these before publishing:

1. **behavior_in_task_description** — All behavior verified by tests is explicitly described in instruction.md
2. **behavior_in_tests** — All behavior described in instruction.md is covered by tests
3. **informative_test_structure** — Tests are well-structured with clear sections and comments
4. **anti_cheating_measures** — No leaked answers in environment files; no mutable external resource dependencies
5. **structured_data_schema** — If the task requires structured output (JSON, CSV), the exact schema is documented
6. **pinned_dependencies** — External dependencies (especially Python packages) have pinned versions
7. **typos** — No typos in filenames, paths, commands, or variable names
8. **tests_or_solution_in_image** — `tests/` and `solution/` are NOT copied into the Dockerfile
9. **test_deps_in_image** — Test-only dependencies are installed in test.sh, not baked into the Dockerfile
10. **hardcoded_solution** — solve.sh demonstrates real steps to derive the answer, not just echoing the result
11. **file_reference_mentioned** — If the agent must produce specific output files, their filenames are explicitly mentioned in instruction.md

Run the checker: `harbor tasks check ./my-task`

## Common Task Patterns

**Programming tasks:** Fix bugs, implement algorithms, build features. Base image with language runtime, seed code in environment, pytest verification.

**DevOps tasks:** Configure services, write CI pipelines, fix infrastructure. Multi-container with docker-compose, service health checks, integration tests.

**Data analysis tasks:** Process datasets, generate reports, build visualizations. Python image with data libraries, seed data files, output format verification.

**MCP tasks:** Agent uses MCP tools to accomplish goals. Configure `[[environment.mcp_servers]]` in task.toml. The MCP server typically runs as a separate service in docker-compose.yaml, and the agent connects via its MCP client.

**GPU tasks:** For ML/CUDA workloads. Set `gpus` and `gpu_types` in `[environment]`. Only Modal and GKE backends support GPUs — `DockerEnvironment` does not. Use `--environment-type modal` when running.

**LLM-as-judge tasks:** Open-ended outputs evaluated by another LLM. Custom logic in test.sh calls an LLM API and writes a normalized score to reward.txt.

## Common Pitfalls

These are the mistakes that trip up task authors most often:

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| Missing reward file | Trial errors with `RewardFileNotFoundError` | Always write reward.txt in every code path, including error paths |
| Empty reward file | Trial errors with `RewardFileEmptyError` | Ensure echo writes a value, not empty string |
| Using `set -e` in test.sh | Script exits on pytest failure before writing reward | Use `set -uo pipefail` (no `-e`), see test.sh patterns above |
| Tests in Dockerfile | Agent can read test expectations and cheat | Never COPY tests/ into the image |
| Service not named `main` | Harbor can't exec/upload/download into container | Always name the primary compose service `main` |
| Unpinned dependencies | Builds break when packages update | Pin every version: `flask==3.1.0`, `pytest==8.4.1` |
| Using `rewards.json` | Harbor looks for `reward.json` (singular) | Use `reward.txt` or `reward.json` |
| Missing output file path in instruction | Agent doesn't know where to write results | Explicitly state every file path the tests will check |
| Requesting GPUs on Docker | Runtime error: environment doesn't support GPUs | Use `--environment-type modal` or `gke` for GPU tasks |

## Complete Example Walkthroughs

See [references/example-tasks.md](references/example-tasks.md) for 4 fully annotated task examples covering simple, medium, complex, and MCP patterns.

See [references/task-toml-reference.md](references/task-toml-reference.md) for the complete field reference with types and defaults.
