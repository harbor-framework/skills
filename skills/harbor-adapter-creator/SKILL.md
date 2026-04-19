---
name: harbor-adapter-creator
description: "Create Harbor benchmark adapters that convert external benchmark datasets into Harbor task format. Use when porting an existing benchmark to Harbor, running parity experiments, registering a dataset to the Harbor registry, or debugging adapter validation failures. Covers: adapter class interface (generate_task, make_local_task_id), directory layout including YAML job configs, oracle verification, parity planning and experiments, dataset registration, and the full post-implementation workflow."
---

# Creating Harbor Benchmark Adapters

Adapters convert external benchmarks (SimpleQA, GAIA, AiderPolyglot, CodePDE, spider2-dbt, etc.) into Harbor's standardized task directory format. Each adapter reads source benchmark data and generates many individual task directories, one per benchmark instance.

## When to Use an Adapter vs. Creating Tasks Directly

**Use an adapter when:**
- You have an existing benchmark dataset with many instances
- Tasks share the same structure but differ in data (questions, code, etc.)
- You want to track evaluation parity with the original benchmark

**Create tasks directly when:**
- You're authoring original evaluation challenges
- Each task has unique structure and environment
- There are fewer than ~10 tasks

## Adapter Directory Structure

Adapters now use a `src` layout (Python package under `src/<adapter_name>/`):

```
adapters/<adapter-name>/
├── .python-version            # Python version (created by uv init)
├── pyproject.toml             # Python package config
├── README.md                  # Benchmark docs, license, parity, citation
├── adapter_metadata.json      # Adapter metadata (JSON array)
├── parity_experiment.json     # Parity tracking results (JSON array)
├── run_<adapter-name>.yaml    # Job config for oracle and parity experiment runs
└── src/
    └── <adapter_name>/         # adapter-name with dashes → underscores
        ├── __init__.py
        ├── adapter.py          # Core conversion logic (adapter class)
        ├── main.py             # CLI entry point
        └── task-template/      # Template files copied into each task
            ├── task.toml
            ├── instruction.md
            ├── environment/
            │   └── Dockerfile
            ├── tests/
            │   └── test.sh
            └── solution/
                └── solve.sh
```

All template files, parity_experiment.json, adapter_metadata.json, and README.md are required. The validator (`harbor adapters validate`) checks for all of them. The YAML job config is not validated but is expected for parity experiments.

## Scaffolding with harbor adapters init

```bash
harbor adapters init my-benchmark
```

The interactive `AdapterWizard` prompts for:
- **Benchmark name** -- Vanilla name (e.g., "SWE-bench")
- **Adapter ID** -- Lowercase, no spaces (e.g., `swebench`)
- **Class name** -- Python class name (auto-derived or overridden, e.g., `SwebenchAdapter`)
- **Description** -- One-line description for the README
- **Source URL** -- Link to original benchmark paper or repo
- **License** -- Dataset license for the README

If all fields are provided as CLI arguments, the wizard runs non-interactively. The wizard renders Jinja2 templates from `src/harbor/cli/template-adapter/` and places the result under `adapters/<adapter-id>/` using the src layout above.

After scaffolding, validate:

```bash
harbor adapters validate adapters/my-benchmark
```

## Writing adapter.py

The adapter class has no formal base class, but every adapter must implement the same interface: a `NAME` attribute, `make_local_task_id` static method, and `generate_task` method.

### Core Class Interface

```python
import shutil
from pathlib import Path
from loguru import logger


class MyBenchmarkAdapter:
    """Converts MyBenchmark instances into Harbor task directories."""

    NAME = "mybenchmark"

    @staticmethod
    def make_local_task_id(source_id: str) -> str:
        """Convert a source benchmark ID to a Harbor task directory name.

        Convention: lowercase, hyphens as separators, prefixed with
        the adapter ID to prevent collisions across benchmarks.
        """
        normalized = source_id.lower().replace("_", "-")
        return f"mybenchmark-{normalized}"

    def __init__(self, task_dir: Path, **kwargs: object):
        """Initialize the adapter.

        Args:
            task_dir: Root directory where generated tasks will be written.
                      Each task gets a subdirectory: task_dir / local_task_id
            **kwargs: Adapter-specific configuration (data paths, splits, etc.)
        """
        self.task_dir = Path(task_dir)
        self._config = kwargs

        # Locate template files relative to this file (inside src/<adapter_name>/)
        self.template_dir = Path(__file__).parent / "task-template"
        if not self.template_dir.exists():
            raise FileNotFoundError(f"Template directory not found: {self.template_dir}")

        # Load your benchmark data here.
        # Examples from real adapters:
        #   SimpleQA: pandas.read_csv() from a URL
        #   GAIA: HuggingFace datasets or local JSONL
        #   AiderPolyglot: walk language dirs, parse .meta/config.json
        #   CodePDE: clone repo, load PDE descriptions
        logger.info(f"Initializing MyBenchmarkAdapter -> {self.task_dir}")

    def _copy_template(self, output_dir: Path) -> None:
        """Copy baseline template files into the output directory."""
        shutil.copytree(self.template_dir, output_dir, dirs_exist_ok=True)

    def generate_task(self, source_id: str, local_task_id: str) -> None:
        """Generate a single Harbor task directory.

        This is the core method. It must create a complete, valid task directory
        with all required files: task.toml, instruction.md, environment/Dockerfile,
        tests/test.sh, and optionally solution/solve.sh.

        Args:
            source_id: Identifier from the original benchmark
            local_task_id: Harbor task directory name (from make_local_task_id)
        """
        output_dir = self.task_dir / local_task_id
        output_dir.mkdir(parents=True, exist_ok=True)

        # 1. Copy baseline template
        self._copy_template(output_dir)

        # 2. Look up the source instance data
        # instance = self.data[source_id]

        # 3. Write task-specific files:
        #    - instruction.md  (the prompt the agent sees)
        #    - task.toml       (metadata, timeouts, resource limits)
        #    - environment/Dockerfile
        #    - tests/test.sh   (MUST write to /logs/verifier/reward.txt)
        #    - solution/solve.sh (reference solution for OracleAgent)
        #    - Copy attachments into environment/ if needed

        logger.info(f"Generated task {local_task_id} in {output_dir.resolve()}")
```

### The generate_task Method in Detail

The `generate_task` method receives `source_id` (the original benchmark's identifier) and `local_task_id` (the Harbor directory name produced by `make_local_task_id`). It builds the output path from `self.task_dir / local_task_id`.

Typical steps inside `generate_task`:

1. **Create directory and copy template** -- `shutil.copytree` with `dirs_exist_ok=True`
2. **Load instance data** -- look up the specific benchmark record
3. **Write instruction.md** -- replace placeholders or write dynamically
4. **Write task.toml** -- set metadata, tags, timeouts
5. **Write or modify Dockerfile** -- add `COPY` commands for attachments
6. **Write tests/test.sh** -- the verification script (see test.sh requirements below)
7. **Write solution/solve.sh** -- reference solution for oracle agent
8. **Copy attachments** -- instance-specific files into `environment/workspace/` or `environment/`

### Data Loading Patterns (from real adapters)

| Adapter | Data Source | Loading Pattern |
|---------|-----------|----------------|
| SimpleQA | OpenAI public CSV | `pandas.read_csv(url)` in `__init__` |
| GAIA | HuggingFace or local JSONL | `datasets` library or line-by-line JSON |
| AiderPolyglot | Exercism repo | Walk language dirs, parse `.meta/config.json` |
| CodePDE | CodePDE repo | Clone repo, load PDE description files |
| spider2-dbt | Spider2-DBT repo | Clone repo, read dbt project structures |

### Template Filling Patterns

- **Simple string replacement**: `content.replace("{KEY}", value)` -- used by GAIA
- **Placeholder replacement**: `content.replace("{{PLACEHOLDER}}", value)` -- common pattern
- **Direct file writes**: Build content programmatically and write -- used by AiderPolyglot
- **Jinja2**: For complex templates with conditionals (not used by existing adapters but supported)

### Handling Attachments

When source benchmarks include per-instance files (images, CSVs, databases):

1. Copy files into `environment/workspace/` in the task directory
2. Add `COPY workspace/ /app/files/` to the Dockerfile (GAIA pattern)
3. Reference files from `instruction.md` using the container path (e.g., `/app/files/data.csv`)
4. Mention the attachment in the instruction so the agent knows about it

## Writing main.py

The CLI entry point lives at `src/<adapter_name>/main.py` and follows a standard pattern. Run it via `uv run python -m <adapter_name>.main`. The default output directory is `datasets/<adapter-id>`.

```python
#!/usr/bin/env python3
import argparse
import sys
from pathlib import Path
from loguru import logger

from mybenchmark.adapter import MyBenchmarkAdapter


def _parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Generate Harbor tasks from MyBenchmark",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    )
    parser.add_argument(
        "--output-dir", type=Path,
        default=Path("datasets/mybenchmark"),
        help="Directory to write generated Harbor task directories",
    )
    parser.add_argument(
        "--task-ids", nargs="*",
        help="Specific source task IDs to generate (space-separated)",
    )
    parser.add_argument(
        "--ids-file", type=Path,
        help="Path to a file containing source task IDs, one per line",
    )
    parser.add_argument(
        "--limit", type=int,
        help="Limit the number of tasks to generate",
    )
    parser.add_argument(
        "--overwrite", action="store_true",
        help="Overwrite existing task directories",
    )
    return parser.parse_args()


def _collect_ids(task_ids: list[str] | None, ids_file: Path | None) -> list[str]:
    """Collect task IDs from command line args or a file."""
    if task_ids:
        return task_ids
    if ids_file:
        return [line.strip() for line in ids_file.read_text().splitlines() if line.strip()]
    return []


def _process_benchmark(output_dir: Path, task_ids: list[str]) -> None:
    """Initialize adapter and generate all tasks."""
    output_dir.mkdir(parents=True, exist_ok=True)
    adapter = MyBenchmarkAdapter(task_dir=output_dir)

    if not task_ids:
        # Load all available IDs from the benchmark data
        task_ids = adapter.get_all_source_ids()

    errors = []
    for i, source_id in enumerate(task_ids, 1):
        local_task_id = MyBenchmarkAdapter.make_local_task_id(source_id)
        try:
            adapter.generate_task(source_id, local_task_id)
            logger.info(f"[{i}/{len(task_ids)}] {source_id} -> {local_task_id}")
        except Exception as e:
            errors.append((source_id, str(e)))
            logger.error(f"[{i}/{len(task_ids)}] {source_id} FAILED: {e}")

    logger.info(f"Generated {len(task_ids) - len(errors)}/{len(task_ids)} tasks")
    if errors:
        sys.exit(1)


def main() -> None:
    args = _parse_args()
    task_ids = _collect_ids(args.task_ids, args.ids_file)
    if args.limit and task_ids:
        task_ids = task_ids[:args.limit]
    _process_benchmark(args.output_dir, task_ids)


if __name__ == "__main__":
    main()
```

## test.sh Requirements

This is the most common source of adapter validation failures. The verifier uploads the entire `tests/` directory to `/tests` inside the container, makes `test.sh` executable, and runs it.

**test.sh MUST write a reward value to `/logs/verifier/reward.txt`** -- a single float (e.g., `0` or `1` or `0.75`). If this file is missing, the verifier raises `RewardFileNotFoundError` and the trial fails.

Optionally, test.sh can also write `/logs/verifier/reward.json` for structured reward data (e.g., `{"reward": 1.0, "quality": 0.8}`). The verifier checks `reward.txt` first, then falls back to `reward.json`.

### Verification Patterns

**String matching** (GAIA pattern):
```bash
#!/bin/bash
mkdir -p /logs/verifier
EXPECTED=$(cat /tests/expected_answer.txt)
if [ ! -f /app/answer.txt ]; then
    echo 0 > /logs/verifier/reward.txt
    exit 0
fi
ACTUAL=$(cat /app/answer.txt | tr '[:upper:]' '[:lower:]' | xargs)
EXPECTED_NORM=$(echo "$EXPECTED" | tr '[:upper:]' '[:lower:]' | xargs)
if [ "$ACTUAL" = "$EXPECTED_NORM" ]; then
    echo 1 > /logs/verifier/reward.txt
else
    echo 0 > /logs/verifier/reward.txt
fi
```

**Pytest-based** (CodePDE, hello-world pattern):
```bash
#!/bin/bash
# Install test dependencies inside test.sh, not in the Dockerfile
apt-get update && apt-get install -y curl
curl -LsSf https://astral.sh/uv/0.9.7/install.sh | sh
source $HOME/.local/bin/env
uvx --with pytest==8.4.1 pytest /tests/test_solution.py -rA
if [ $? -eq 0 ]; then
    echo 1 > /logs/verifier/reward.txt
else
    echo 0 > /logs/verifier/reward.txt
fi
```

**LLM judge** (SimpleQA pattern):
```bash
#!/bin/bash
mkdir -p /logs/verifier
cd /tests
python3 llm_judge.py  # reads ground_truth.json, writes reward.txt
```

## task.toml Configuration

Every generated `task.toml` **must** contain a `name` field under the `[task]` section. Harbor uses this to identify the task when it's added to a dataset; tasks without a `name` cannot be registered.

```toml
version = "1.0"

[task]
name = "mybenchmark/task-001"    # required: <org>/<task-id>, must be stable across runs

[metadata]
author_name = "Your Name"
difficulty = "medium"           # easy, medium, hard
category = "qa"
tags = ["factual", "reasoning"]

[agent]
timeout_sec = 600.0             # max wall-clock seconds for the agent

[verifier]
timeout_sec = 600.0             # max seconds for test.sh
# env = { API_KEY = "..." }    # env vars injected into test.sh

[environment]
build_timeout_sec = 600.0       # max seconds for docker build
cpus = 1                        # CPU cores (default: 1)
memory_mb = 2048                # RAM in MB (default: 2048)
storage_mb = 10240              # disk in MB (default: 10240)
gpus = 0                        # GPUs (default: 0)
allow_internet = true           # network access (default: true)

[solution]
# env = { SECRET = "..." }     # env vars for solve.sh (OracleAgent)
```

**Task naming requirements:**
- `name` must be unique within the dataset and **stable across adapter runs** (unstable names churn registry digests on republish)
- Format: `<org>/<task-id>`, e.g., `mybenchmark/task-001`
- Sanitize upstream identifiers: lowercase, replace spaces/slashes/special characters with hyphens
- If the upstream lacks stable identifiers, mint a deterministic scheme (e.g., `{dataset}-1`, `{dataset}-2`) from a reproducible sort
- `version = "1.0"` is the schema version — leave it alone

## parity_experiment.json

A JSON **array** of experiment objects tracking how Harbor results compare to the original benchmark. Required fields per entry:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `adapter_name` | string | Yes | Adapter name (e.g., `"mybenchmark"`) |
| `agent` | string | Yes | Agent with version (e.g., `"codex@1.0"`) |
| `model` | string | Yes | Full model identifier |
| `date` | string | Yes | Experiment date |
| `adapted_benchmark_size` | integer | Yes | Total tasks converted by adapter |
| `parity_benchmark_size` | integer | Yes | Tasks used for parity |
| `number_of_runs` | integer | Yes | Runs per side (must match on both sides) |
| `notes` | string | No | Additional context |
| `original_parity_repo` | string | Yes | Fork URL for reproducing parity on original |
| `adapter_pr` | string[] | Yes | All adapter PR links in harbor repo |
| `dataset_pr` | string[] | Yes | All PR links in harbor-datasets repo |
| `parity_pr` | string[] | Yes | All PR links to HuggingFace parity dataset |
| `metrics` | object[] | Yes | Metric comparison objects |

Each `metrics` entry needs `benchmark_name`, `metric`, `original` (mean ± stderr string), `harbor` (mean ± stderr string), `original_runs` (array of per-run scores), and `harbor_runs` (array of per-run scores).

```json
[
  {
    "adapter_name": "mybenchmark",
    "agent": "codex@1.0",
    "model": "openai/gpt-5-2025-06-01",
    "date": "2026-01-15",
    "adapted_benchmark_size": 500,
    "parity_benchmark_size": 50,
    "number_of_runs": 3,
    "notes": "50-task sample; averaged over 3 runs",
    "original_parity_repo": "https://github.com/example/mybenchmark/tree/harbor-parity",
    "adapter_pr": [
      "https://github.com/harbor-framework/harbor/pull/123"
    ],
    "dataset_pr": [
      "https://github.com/harbor-framework/harbor-datasets/pull/45"
    ],
    "parity_pr": [
      "https://huggingface.co/datasets/harborframework/parity-experiments/discussions/10"
    ],
    "metrics": [
      {
        "benchmark_name": "MyBenchmark",
        "metric": "Accuracy",
        "original": "72.0 +/- 1.5",
        "harbor": "71.3 +/- 1.2",
        "original_runs": [73.0, 69.0, 74.0],
        "harbor_runs": [72.0, 70.0, 71.9]
      }
    ]
  }
]
```

**Important:** the per-run arrays are named `original_runs` and `harbor_runs` (not `_trials`). The validator enforces `*_runs` naming.

## adapter_metadata.json

A JSON **array** describing the adapter and its relationship to the original benchmark.

```json
[
  {
    "adapter_name": "mybenchmark",
    "adapter_builders": [
      "Your Name (you@example.com)"
    ],
    "original_benchmark": [
      {
        "split": "test",
        "size": 500,
        "harness": "agent",
        "supported_agents": ["openhands"],
        "adaptable": true,
        "notes": "Only test split has ground-truth answers."
      }
    ],
    "harbor_adapter": [
      {
        "split": "test",
        "adapted_benchmark_size": 500,
        "parity_benchmark_size": 50,
        "parity_sampling_rate": 0.1,
        "registry_benchmark_size": 500,
        "added_agents": ["None"],
        "parity_matching_agents": ["codex@1.0+openai/gpt-5-2025-06-01"],
        "parity_unmatching_agents": ["None"],
        "parity_costs": "$150",
        "notes": "Full benchmark adapted. Parity on 10% sample."
      }
    ]
  }
]
```

**New required fields in `harbor_adapter`:**
- `added_agents`: custom agents added for this adapter (`["None"]` if none)
- `parity_unmatching_agents`: agents that were tested but did not achieve parity (`["None"]` if all matched)

## Validation

The `harbor adapters validate` command (backed by `scripts/validate_adapter.py`) checks:

1. **Required files exist**: `adapter_metadata.json`, `parity_experiment.json`, `README.md`, `run_<adapter-name>.yaml`
2. **src layout**: `src/<adapter_name>/adapter.py`, `src/<adapter_name>/main.py`, `src/<adapter_name>/task-template/`
3. **Template structure**: `task-template/task.toml`, `task-template/instruction.md`, `task-template/environment/Dockerfile`, `task-template/tests/test.sh`, `task-template/solution/solve.sh`
4. **test.sh writes reward**: Template `tests/test.sh` must contain a write to `/logs/verifier/reward.txt`
5. **parity_experiment.json schema**: Valid JSON array with required fields
6. **adapter_metadata.json schema**: Valid JSON array with required fields
7. **README.md compliance**: Checks for "Overview", "Parity", "Citation" sections

```bash
# Validate adapter structure
harbor adapters validate adapters/mybenchmark

# Generate a single task to test
uv run python -m mybenchmark.main \
  --output-dir datasets/mybenchmark \
  --task-ids instance-001

# Validate generated task
harbor tasks check datasets/mybenchmark/mybenchmark-instance-001

# Run a single trial with oracle agent to verify the solution works
harbor trials start -p datasets/mybenchmark/mybenchmark-instance-001 --agent oracle

# Run all tasks as a batch job
harbor jobs start -p datasets/mybenchmark --agent oracle
```

## GPU Tasks

For adapters with GPU tasks, add a `docker-compose.yaml` in the task's `environment/` directory with nvidia device reservations. For cloud/Modal runs, also set `gpus` in `task.toml`. See the [featurebench adapter](https://github.com/harbor-framework/harbor/tree/main/adapters/featurebench) for a comprehensive example — it handles 44 GPU tasks across multiple repos with separate `_docker_cpu.yaml`, `_docker_gpu.yaml`, and `_modal.yaml` config files.

## Reference Adapters by Scenario

| Scenario | Example adapter | When to use |
|----------|----------------|-------------|
| Compatible agent already exists | `adapters/adebench/` | Upstream already supports Claude-Code / Codex / OpenHands / Gemini-CLI |
| Fork upstream + add LLM agent | `adapters/evoeval/` | LLM-based benchmark with no Harbor-compatible agent |
| Custom agent, separate dataset | `adapters/bixbench/`, `adapters/financeagent/` | Custom interaction semantics; financeagent also demos LLM-as-a-Judge |
| Custom agent in-place | `adapters/medagentbench/` | Custom HTTPAgent, no separate dataset |
| Multi-agent workflow | `adapters/cooperbench/` | Multiple agents coordinate via messaging / sidecars |
| GPU tasks | `adapters/featurebench/` | Comprehensive Docker + Modal GPU example |

## Post-Implementation Workflow

After generating valid tasks, complete these steps before submitting:

1. **Oracle Verification** — Run oracle agent across all tasks; must reach 100% pass rate. Create a WIP PR with a screenshot of results.
2. **Parity Planning** — Contact the team before running parity. They determine agents, models, number of runs, and API key provisioning.
3. **Parity Experiments** — Run target agent+model against both original harness and Harbor, using `run_<adapter-id>.yaml`. Record all run scores.
4. **Results Documentation** — Fill `parity_experiment.json`; upload results to the HuggingFace parity dataset via the `upload-parity-experiments` skill.
5. **Dataset Registration** — Fork `harbor-datasets`, place tasks under `datasets/<adapter-id>/`, run `harbor init` to create `dataset.toml`, open a PR.
6. **Adapter PR** — Change title from `[WIP]` to `[Ready for Review] Adapter: {adapter_name}`, request review from `@Slimshilin`.
7. **README Thoroughness** — Ensure README covers Overview, Parity results, License, and Citation. The README is parsed by automation — do not add, rename, reorder, or remove template sections.

See [references/adapter-anatomy.md](references/adapter-anatomy.md#6-post-implementation-workflow) for commands and YAML job config format.

## Gotchas

- **Missing reward.txt write**: The most common failure. test.sh must always write to `/logs/verifier/reward.txt`, even on error paths.
- **Wrong parity_experiment.json format**: It is a JSON array `[{...}]`, not a plain object `{...}`. Using the singular filename `parity_experiment.json`, not plural. Per-run score arrays are named `original_runs` / `harbor_runs` (not `_trials`).
- **Forgetting adapter_metadata.json**: This file is required and checked by the validator. Include `added_agents` and `parity_unmatching_agents` fields.
- **Baking test dependencies into Dockerfile**: Test-only dependencies (pytest, evaluation scripts) should be installed inside `test.sh`, not in the Docker image. The `tests/` directory is uploaded separately at verification time.
- **Including tests/ or solution/ in Docker image**: These directories should not be COPYed in the Dockerfile. They are injected by Harbor at runtime.
- **Missing `[task]` section in task.toml**: Every generated task.toml must include a `[task]` block with a `name` field. Without it, tasks cannot be registered. The `name` field does not belong at the top level — it must be under `[task]`.
- **Unstable task names**: Task names must be deterministic across adapter runs. If upstream IDs change or aren't stable, mint a reproducible scheme from a consistent sort.
- **Using /app vs /workspace inconsistently**: Pick one working directory and be consistent between instruction.md, test.sh, and Dockerfile WORKDIR.
- **Not escaping shell-unsafe characters**: Answers containing single quotes, backticks, or dollar signs can break test.sh or solve.sh. Escape them when embedding in shell scripts.
- **Missing `mkdir -p /logs/verifier`**: Some base images may not have this directory. Create it in test.sh before writing reward.txt.

## Existing Adapter Patterns

See [references/adapter-anatomy.md](references/adapter-anatomy.md) for annotated walkthroughs of all existing adapters (SimpleQA, GAIA, AiderPolyglot, CodePDE, spider2-dbt, StrongReject) and a full comparison table.
