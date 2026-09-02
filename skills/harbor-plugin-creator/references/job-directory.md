# Job Directory Layout

What a finished job leaves on disk, and how to read it. Verified against Harbor 0.22.0.

## Layout

```
<jobs_dir>/<job_name>/
├── config.json                 # JobConfig
├── lock.json                   # JobLock, resolved inputs
├── job.log
├── result.json                 # JobResult, WITHOUT trial results
└── <trial_name>/
    ├── config.json             # TrialConfig
    ├── lock.json               # TrialLock
    ├── result.json             # TrialResult for this trial
    ├── trial.log
    ├── exception.txt           # only when the trial raised
    ├── agent/                  # agent logs, trajectory.json
    ├── verifier/               # verifier output, reward file
    └── artifacts/
        └── manifest.json
```

Multi-step trials nest their phases under `steps/<step_name>/` inside the trial
directory, each with its own `agent/`, `verifier/` and `artifacts/`.

## The Filenames That Trip People Up

Three names differ from what nearby documentation suggests. All three fail
quietly, producing plausible output rather than an error.

**`reward.json`, not `rewards.json`.** The verifier reads
`/logs/verifier/reward.txt` (one float, stored under the key `reward`) or
`/logs/verifier/reward.json` (an object of named rewards). Harbor's own task
template comment instructs you to write `rewards.json`, and a task that follows
it fails with `RewardFileNotFoundError` despite the verifier having produced
correct output.

**Per-trial `result.json`, not `results.json`.** The `TrialPaths` docstring
describes `results.json`. Harbor writes `result.json`. Accept both if you want to
tolerate other versions.

**The job `result.json` has no trial results.** Harbor calls
`_write_job_result(exclude_trial_results=True)` at five sites, including the
final write when a job completes. The persisted file therefore contains only
`id`, `started_at`, `finished_at`, `updated_at`, `n_total_trials` and `stats`.

There is no `trial_results` key. Code that loads that file and iterates
`job_result.trial_results` sees an empty list, summarizes zero trials, and
reports success.

## Reading a Finished Job

Per-trial files are the source of truth. The job result is a fallback for runs
that did persist them inline.

```python
from pathlib import Path

from harbor.models.job.result import JobResult
from harbor.models.trial.result import TrialResult


def load_job(job_dir: str | Path) -> tuple[JobResult, list[TrialResult]]:
    """Load a finished job, reading trials from their own directories."""
    path = Path(job_dir)
    job_result = JobResult.model_validate_json(
        (path / "result.json").read_text(encoding="utf-8")
    )

    trials: list[TrialResult] = []
    seen: set[Path] = set()
    # "result.json" is current; "results.json" is accepted so a directory
    # written by another version still loads.
    for pattern in ("*/result.json", "*/results.json"):
        for candidate in sorted(path.glob(pattern)):
            if candidate.parent in seen:
                continue
            try:
                trials.append(
                    TrialResult.model_validate_json(
                        candidate.read_text(encoding="utf-8")
                    )
                )
                seen.add(candidate.parent)
            except Exception:
                continue  # skip a truncated or unreadable trial

    if not trials:
        trials = list(job_result.trial_results)
    return job_result, trials
```

Note the glob excludes the job's own `result.json`, which sits at the top level
rather than one directory down.

## Moved Directories

A job directory can be copied or moved after the run. `TrialConfig.trials_dir`
still points at the original location, so resolving trial files through it breaks.
Resolve relative to the directory you are actually reading, and reassign
`trial_result.config.trials_dir` if downstream code depends on it.

## Verifying a Plugin Against Real Output

Two built-in agents exercise both paths in real containers with no model spend:

```bash
harbor run --path task --agent oracle --plugin myplugin   # applies the reference solution, passes
harbor run --path task --agent nop    --plugin myplugin   # does nothing, fails
```

`oracle` copies `solution/solve.sh` into the container, so it exercises the full
environment build, agent phase and verifier without any model calls. Use both
before trusting a plugin against a paid run.
