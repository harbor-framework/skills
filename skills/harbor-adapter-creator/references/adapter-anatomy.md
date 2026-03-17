# Adapter Anatomy: Annotated Walkthroughs

Real adapter patterns from the Harbor ecosystem, showing the class interface, data loading, task generation, and verification approaches.

## Table of Contents

- [1. QA Adapter (SimpleQA pattern)](#1-qa-adapter-simpleqa-pattern) -- LLM-judge grading, remote CSV
- [2. Attachment Adapter (GAIA pattern)](#2-attachment-adapter-gaia-pattern) -- file attachments, string matching
- [3. Code Evaluation Adapter (AiderPolyglot pattern)](#3-code-evaluation-adapter-aiderpolyglot-pattern) -- language-specific setup, encrypted oracle
- [4. Scientific Computing Adapter (CodePDE pattern)](#4-scientific-computing-adapter-codepde-pattern) -- pytest + nRMSE, HuggingFace data download
- [5. Common Patterns Across All Adapters](#5-common-patterns-across-all-adapters)
- [6. Post-Implementation Workflow](#6-post-implementation-workflow) -- oracle, parity, dataset registration
- [7. Existing Adapter Patterns](#7-existing-adapter-patterns) -- comparison table

---

## 1. QA Adapter (SimpleQA pattern)

SimpleQA converts short factual questions from OpenAI's public dataset into Harbor tasks. Its key distinguishing feature is using an LLM judge for grading rather than exact string matching.

### Directory Structure

```
adapters/simpleqa/
├── adapter.py
├── run_adapter.py
├── README.md
├── parity_experiment.json
├── adapter_metadata.json
└── template/
    ├── task.toml
    ├── instruction.md
    ├── environment/
    │   └── Dockerfile
    ├── tests/
    │   ├── test.sh
    │   └── llm_judge.py          # LLM-based grading script
    └── solution/
        └── solve.sh
```

### adapter.py

```python
"""SimpleQA Adapter -- converts OpenAI SimpleQA dataset into Harbor tasks."""

import shutil
from dataclasses import dataclass
from pathlib import Path

import pandas
from loguru import logger


@dataclass
class SimpleQATask:
    source_id: str
    question: str
    answer: str
    topic: str
    answer_type: str


class SimpleQAAdapter:
    """Converts SimpleQA instances into Harbor task directories."""

    NAME = "simpleqa"

    # Data source: OpenAI public CSV
    DATA_URL = "https://openaipublic.blob.core.windows.net/simple-evals/simple_qa_test_set.csv"

    @staticmethod
    def make_local_task_id(source_id: str) -> str:
        normalized = source_id.lower().replace("_", "-")
        return f"simpleqa-{normalized}"

    def __init__(self, task_dir: Path, **kwargs: object):
        self.task_dir = Path(task_dir)
        self.template_dir = Path(__file__).parent / "template"

        # Load data directly from URL
        df = pandas.read_csv(self.DATA_URL)
        self.tasks: dict[str, SimpleQATask] = {}
        for idx, row in df.iterrows():
            sid = str(idx)
            self.tasks[sid] = SimpleQATask(
                source_id=sid,
                question=row["problem"],
                answer=row["answer"],
                topic=row.get("metadata", {}).get("topic", "general"),
                answer_type=row.get("metadata", {}).get("answer_type", "short"),
            )
        logger.info(f"Loaded {len(self.tasks)} SimpleQA tasks")

    def generate_task(self, source_id: str, local_task_id: str) -> None:
        task = self.tasks[source_id]
        output_dir = self.task_dir / local_task_id
        output_dir.mkdir(parents=True, exist_ok=True)

        # Copy template (includes llm_judge.py)
        shutil.copytree(self.template_dir, output_dir, dirs_exist_ok=True)

        self._prepare_task(task, output_dir)
        logger.info(f"Generated {local_task_id}")

    def _prepare_task(self, task: SimpleQATask, output_dir: Path) -> None:
        """Populate all task-specific files."""

        # ground_truth.json -- consumed by llm_judge.py during verification
        import json
        gt = {"question": task.question, "answer": task.answer,
              "topic": task.topic, "answer_type": task.answer_type}
        (output_dir / "tests" / "ground_truth.json").write_text(
            json.dumps(gt, indent=2)
        )

        # instruction.md
        (output_dir / "instruction.md").write_text(
            f"# Question\n\n{task.question}\n\n"
            "Write your answer to `/workspace/answer.txt`. Write only the answer.\n"
        )

        # task.toml -- set tags from metadata
        toml_content = (output_dir / "task.toml").read_text()
        toml_content = toml_content.replace("{{TOPIC}}", task.topic)
        (output_dir / "task.toml").write_text(toml_content)

        # solution/solve.sh -- escape for shell safety
        safe_answer = task.answer.replace("'", "'\\''")
        (output_dir / "solution" / "solve.sh").write_text(
            f"#!/bin/bash\necho '{safe_answer}' > /workspace/answer.txt\n"
        )
```

### Key Insight: LLM Judge

Instead of exact string matching, SimpleQA uses an `llm_judge.py` script in `tests/` that:
1. Reads `ground_truth.json` (question + expected answer)
2. Reads the agent's answer from `/workspace/answer.txt`
3. Calls an LLM (e.g., `gpt-4o-mini`) to grade: CORRECT, INCORRECT, or NOT_ATTEMPTED
4. Writes `1.0` or `0.0` to `/logs/verifier/reward.txt`
5. Saves grading details to `/logs/verifier/grading_details.json`

This approach handles paraphrased answers, different units, and other variations that string matching would miss.

---

## 2. Attachment Adapter (GAIA pattern)

GAIA tasks involve multi-step reasoning and often come with file attachments (PDFs, images, spreadsheets). The adapter must copy these into the task environment.

### adapter.py (key methods)

```python
"""GAIA Adapter -- converts GAIA validation tasks into Harbor tasks."""

from dataclasses import dataclass
from pathlib import Path
import shutil
from loguru import logger


@dataclass
class GAIATask:
    task_id: str
    question: str
    answer: str
    level: int              # 1, 2, or 3
    has_attachment: bool
    attachment_name: str | None


class GAIAAdapter:
    NAME = "gaia"

    @staticmethod
    def make_local_task_id(source_id: str) -> str:
        normalized = source_id.lower().replace("_", "-")
        return f"gaia-{normalized}"

    def __init__(self, task_dir: Path, data_dir: Path | None = None, **kwargs):
        self.task_dir = Path(task_dir)
        self.data_dir = data_dir  # directory containing attachment files
        self.template_dir = Path(__file__).parent / "template"

    def _render_template(self, template_path: Path, context: dict) -> str:
        """Simple {KEY} placeholder replacement."""
        content = template_path.read_text()
        for key, value in context.items():
            content = content.replace(f"{{{key}}}", str(value))
        return content

    def generate_task(self, source_id: str, local_task_id: str) -> None:
        task = self.tasks[source_id]
        task_dir = self.task_dir / local_task_id

        # Create directory structure
        (task_dir / "environment" / "workspace").mkdir(parents=True, exist_ok=True)
        (task_dir / "solution").mkdir(parents=True, exist_ok=True)
        (task_dir / "tests").mkdir(parents=True, exist_ok=True)

        # Handle attachments
        attachment_info = ""
        has_attachment = False
        if task.has_attachment and self.data_dir:
            src_file = self.data_dir / task.attachment_name
            if src_file.exists():
                dst = task_dir / "environment" / "workspace" / task.attachment_name
                shutil.copy2(src_file, dst)
                has_attachment = True
                attachment_info = (
                    f"\n\nAn attached file `{task.attachment_name}` "
                    f"is available at `/app/files/{task.attachment_name}`."
                )

        # Generate Dockerfile (conditionally add COPY for attachments)
        dockerfile_content = self._render_template(
            self.template_dir / "environment" / "Dockerfile",
            {"WORKDIR": "/app"}
        )
        if has_attachment:
            dockerfile_content += "\n# Copy attached files\nCOPY workspace/ /app/files/\n"
        (task_dir / "environment" / "Dockerfile").write_text(dockerfile_content)

        # Instruction with attachment note
        (task_dir / "instruction.md").write_text(
            self._render_template(
                self.template_dir / "instruction.md",
                {"QUESTION": task.question, "ATTACHMENT_INFO": attachment_info}
            )
        )

        # Map GAIA levels to difficulty and tags
        difficulty = {1: "easy", 2: "medium", 3: "hard"}.get(task.level, "medium")
        (task_dir / "task.toml").write_text(
            self._render_template(
                self.template_dir / "task.toml",
                {"DIFFICULTY": difficulty, "TAGS": f'["gaia", "level-{task.level}"]'}
            )
        )

        # test.sh + expected_answer.txt
        shutil.copy2(self.template_dir / "tests" / "test.sh", task_dir / "tests" / "test.sh")
        (task_dir / "tests" / "expected_answer.txt").write_text(task.answer)

        # solve.sh
        safe_answer = task.answer.replace("'", "'\\''")
        (task_dir / "solution" / "solve.sh").write_text(
            self._render_template(
                self.template_dir / "solution" / "solve.sh",
                {"ANSWER": safe_answer}
            )
        )

        logger.info(f"Generated {local_task_id} (level {task.level}, attachment={has_attachment})")
```

### Key Insight: Conditional Dockerfile

GAIA dynamically modifies the Dockerfile based on whether the task has attachments. Tasks with files get a `COPY workspace/ /app/files/` line appended. This pattern is useful whenever your benchmark has optional per-instance resources.

### test.sh (string matching with normalization)

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

The expected answer is stored in a separate file (`expected_answer.txt`) rather than embedded in the shell script. This avoids shell escaping issues entirely.

---

## 3. Code Evaluation Adapter (AiderPolyglot pattern)

AiderPolyglot converts Exercism exercises across many programming languages into Harbor tasks. It has the most complex `generate_task` of any adapter due to language-specific handling.

### Key Design Elements

**Exercise discovery**: Walks language directories in the Polyglot repo, reads `.meta/config.json` for each exercise to find solution files, test files, and editor files.

**Language-specific workspace setup** (`_setup_workspace`):
- **C++**: Copies `CMakeLists.txt`, removes `-Werror`
- **Java**: Copies `gradlew`, `build.gradle`, `settings.gradle`, `gradle/wrapper`
- **Go**: Copies `go.mod`, `go.sum`
- **Rust**: Copies `Cargo.toml` (prefers `Cargo-example.toml`), `Cargo.lock`, `.cargo`
- **JavaScript**: Copies `package.json`, `yarn.lock`, `tsconfig.json`, `jest.config.js`

**Encrypted oracle payload**: The reference solution is encrypted with `openssl enc -aes-256-cbc` and stored as `solution.enc`. The `solve.sh` decrypts it at runtime. This prevents agents from accidentally reading the solution during task execution.

### generate_task structure (simplified)

```python
def generate_task(self, source_id: str, local_task_id: str) -> None:
    task = self.tasks[source_id]
    output_dir = self.task_dir / local_task_id

    # Create all directories
    environment_dir = output_dir / "environment"
    workspace_dir = environment_dir / "workspace"
    solution_dir = output_dir / "solution"
    tests_dir = output_dir / "tests"
    for d in [workspace_dir, solution_dir, tests_dir]:
        d.mkdir(parents=True, exist_ok=True)

    # Each step is a separate private method
    self._setup_workspace(task, workspace_dir)
    self._setup_test_files(task, tests_dir)
    self._create_instruction(task, output_dir)
    self._create_task_toml(task, output_dir)
    self._create_test_script(task, tests_dir)      # renders language-specific test.sh
    self._create_solution_script(task, solution_dir, workspace_dir, tests_dir)
    self._create_dockerfile(task, environment_dir)  # renders language-specific Dockerfile
```

### Key Insight: Decompose Complex Adapters

When `generate_task` is complex, break it into private methods. AiderPolyglot has seven helper methods, each handling one aspect of task generation. This makes the adapter testable and maintainable.

---

## 4. Scientific Computing Adapter (CodePDE pattern)

CodePDE evaluates AI-generated solvers for Partial Differential Equations. Tasks require downloading data from HuggingFace at test time and running pytest with numerical accuracy checks.

### Key Design Elements

**PDE-specific configuration**: A dictionary mapping PDE types (advection, burgers, reacdiff1d, etc.) to parameters, difficulty, timeouts, and data file paths.

**test.sh downloads data at test time**:
```bash
#!/bin/bash
mkdir -p /logs/verifier /app/data

# Download PDE data from HuggingFace if not cached
DATA_FILE="advection_data.hdf5"
if [ ! -f "/app/data/$DATA_FILE" ]; then
    curl -L "https://huggingface.co/datasets/LDA1020/codepde-data/resolve/main/$DATA_FILE" \
         -o "/app/data/$DATA_FILE"
fi

# Run pytest to evaluate solver accuracy
cd /app
python -m pytest /tests/test_outputs.py -rA
_EXIT_CODE=$?

if [ $_EXIT_CODE -eq 0 ]; then
    echo 1 > /logs/verifier/reward.txt
else
    echo 0 > /logs/verifier/reward.txt
fi
exit $_EXIT_CODE
```

**Pytest test file** (`test_outputs.py`) calls `nRMSE_evaluator.py` to compute the normalized Root Mean Square Error, then asserts `nRMSE < 0.05`.

### Key Insight: Heavy Test Dependencies Belong in test.sh

CodePDE needs `pytest`, HDF5 libraries, and large data files. These are installed/downloaded inside `test.sh`, not baked into the Dockerfile. This keeps the agent's Docker image lean and focused on what the agent actually needs.

---

## 5. Common Patterns Across All Adapters

### The Adapter Class Contract

Every adapter follows this interface (no formal base class, but enforced by convention and the init wizard template):

```python
class XxxAdapter:
    NAME = "xxx"                    # lowercase adapter identifier

    @staticmethod
    def make_local_task_id(source_id: str) -> str:
        """source_id -> harbor task directory name"""
        ...

    def __init__(self, task_dir: Path, **kwargs):
        """Load benchmark data, set up template paths"""
        ...

    def generate_task(self, source_id: str, local_task_id: str) -> None:
        """Create one complete Harbor task directory"""
        ...
```

### Container Path Conventions

| Path | Purpose |
|------|---------|
| `/app` or `/workspace` | Agent working directory (set by Dockerfile WORKDIR) |
| `/logs/verifier/reward.txt` | Scalar reward output (required) |
| `/logs/verifier/reward.json` | Structured reward output (optional) |
| `/logs/verifier/test.stdout` | test.sh stdout/stderr (written by Harbor) |
| `/tests/` | Where Harbor uploads the tests/ directory contents |
| `/app/files/` | Common location for attachment files (GAIA convention) |

### Output Directory Convention

The default output directory for generated tasks is `datasets/<adapter-id>`, matching the Harbor Dataset Registry pattern. After generating tasks, they can be registered in `registry.json` (in the `harbor-datasets` repo) for discovery via `harbor datasets list` and `harbor datasets download`.

### Error Handling in generate_task

Good practice: wrap `generate_task` in a try/except in `run_adapter.py` and clean up partial directories on failure (as AiderPolyglot does):

```python
try:
    adapter.generate_task(source_id, local_task_id)
except Exception as e:
    # Remove partial task directory
    partial_dir = output_dir / local_task_id
    if partial_dir.exists():
        shutil.rmtree(partial_dir)
    logger.error(f"Failed to generate {source_id}: {e}")
```

### Adapting Benchmark Types

| Benchmark Type | Template Differences | test.sh Approach |
|---------------|---------------------|------------------|
| **QA / Factual** (SimpleQA) | Minimal Dockerfile, simple instruction | LLM judge or string matching |
| **Reasoning + Files** (GAIA) | Conditional Dockerfile COPY, workspace dir | String match with normalization |
| **Code Editing** (AiderPolyglot) | Language runtimes in Dockerfile, workspace with source code | Run language-specific test suites |
| **Scientific Computing** (CodePDE) | Python scientific stack in Dockerfile | pytest + numerical accuracy (nRMSE) |
| **Data Transformation** (spider2-dbt) | dbt + DuckDB in Dockerfile, project files | Compare against gold-standard database |
| **Safety / Refusal** (StrongReject) | Minimal environment | Custom scoring of defense behavior |
| **Graded Scoring** (any) | Same as above | Write float 0.0-1.0 for partial credit |

---

## 6. Post-Implementation Workflow

After the adapter generates valid tasks and oracle verification passes, there are 6 more steps before the adapter is ready to merge.

### Step 1: Oracle Verification

Run the oracle agent against all generated tasks. Every task must pass with reward `1.0`.

```bash
# Run oracle against all generated tasks
harbor jobs start -p datasets/mybenchmark --agent oracle

# Check results
harbor view jobs
```

Fix any tasks that fail before proceeding. Common oracle failures: shell escaping bugs in `solve.sh`, wrong working directory, missing `mkdir -p` for output directory.

### Step 2: Parity Planning

Select a representative sample of tasks for the parity experiment — typically 10% or 50 tasks, whichever is larger. Identify which agent+model combination matches the original benchmark's reported results.

Create a `run_<adapter-id>.yaml` job config in the adapter directory:

```yaml
# run_mybenchmark.yaml
# Reference config for running parity experiment
dataset: datasets/mybenchmark
agent: codex
model: openai/gpt-4o-2025-03-01
num_trials: 1
task_ids:
  - mybenchmark-instance-001
  - mybenchmark-instance-002
  # ... parity sample
```

The naming convention varies across adapters (e.g., `gaia.yaml`, `simpleqa_oracle.yaml`, `simpleqa_parity_claude_opus4_6.yaml`). Use a name that makes the config's purpose clear.

### Step 3: Parity Experiments

Run the same agent+model on both the original benchmark harness and on Harbor. Compare results.

```bash
# Run Harbor parity experiment using the YAML config
harbor jobs start -c adapters/mybenchmark/run_mybenchmark.yaml

# View results
harbor view jobs
```

Record all individual trial scores in addition to the mean. The `parity_experiment.json` requires `original_trials` and `harbor_trials` arrays.

### Step 4: Results Documentation and HuggingFace Upload

1. Fill in `parity_experiment.json` with the experiment results (see schema in SKILL.md).
2. Upload results to the [HuggingFace Harbor Parity Experiments dataset](https://huggingface.co/datasets/harborframework/parity-experiments) by opening a PR there following its contribution guide.

### Step 5: Dataset Registration

Fork the `harbor-datasets` repository. Place all generated tasks under `datasets/<adapter-id>/`. Update `registry.json` with a new dataset entry:

```json
{
  "name": "mybenchmark",
  "version": "1.0",
  "description": "MyBenchmark tasks adapted for Harbor",
  "tasks": [
    {
      "name": "mybenchmark-instance-001",
      "git_url": "https://github.com/laude-institute/harbor-datasets.git",
      "git_commit_id": "<commit-hash>",
      "path": "datasets/mybenchmark/mybenchmark-instance-001"
    }
  ]
}
```

Add a second entry with `"version": "parity"` and only the parity sample tasks — this enables `harbor jobs start -d mybenchmark@parity`. Open a PR to `harbor-datasets`.

### Step 6: Adapter PR

Open a PR to the main `harbor` repository with:
- The full `adapters/<adapter-id>/` directory
- `adapter.py`, `run_adapter.py`, `run_<adapter-id>.yaml`
- `parity_experiment.json`, `adapter_metadata.json`
- `README.md` with Overview, Parity results, License, and Citation sections
- `template/` with all required files

Link the dataset PR and HuggingFace parity experiment in the PR description.

---

## 7. Existing Adapter Patterns

| Adapter | Benchmark | Key Pattern | Data Source |
|---------|-----------|-------------|-------------|
| simpleqa | OpenAI SimpleQA | LLM-judge grading via `llm_judge.py` + `ground_truth.json` | CSV from OpenAI public URL |
| gaia | GAIA | File attachments, string matching, `_render_template` helper | HuggingFace or local JSONL |
| aider_polyglot | Aider Polyglot | Language-specific workspace setup, encrypted oracle payload | Exercism repo directories |
| codepde | CodePDE | Pytest + nRMSE evaluation, HuggingFace data download in test.sh | CodePDE repo clone |
| spider2-dbt | Spider2-DBT | dbt project setup, DuckDB gold-standard DB comparison | Spider2-DBT repo clone |
| strongreject | StrongReject | Safety/refusal evaluation, defense score metric | StrongReject dataset |
