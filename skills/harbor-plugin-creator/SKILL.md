---
name: harbor-plugin-creator
description: "Create Harbor job plugins that observe a job and export its results. Use when building an integration that ships Harbor results to an external platform, writing per-trial records or reports to disk, streaming trial outcomes as they finish, or debugging why a plugin is not discovered by --plugin. Covers: the harbor.plugins entry point group, the JobPlugin protocol and BaseJobPlugin base class, the on_job_start and on_job_end lifecycle, subscribing to trial hooks for streaming, passing configuration with --plugin-kwarg, fail-open discipline, correct handling of retried and resumed trials, and the on-disk filenames a plugin must read."
---

# Creating Harbor Job Plugins

A job plugin observes a Harbor job from the host process. It receives the job
before trials start and the results after they finish, and it can subscribe to
per-trial events in between. Use one to export results somewhere, write reports,
or ship telemetry, without changing how the job runs.

A plugin never runs inside a task container. It runs in the orchestrator, so
credentials it holds are not reachable by the agent under evaluation.

## When to Use a Plugin

**Use a job plugin when:**
- Results need to reach an external system (a dashboard, a tracing backend, a database)
- You want reports or flat records written alongside the job directory
- You need per-trial reactions while the job is still running

**Do not use a plugin when:**
- You are converting a benchmark dataset into tasks. Use an adapter instead.
- You are grading a task. Use a verifier.
- You only need to read a finished job. Read the job directory directly, no plugin required.

## The Contract

Two async methods. That is the whole interface.

```python
from harbor.models.job.plugin import BaseJobPlugin


class MyPlugin(BaseJobPlugin):
    def __init__(self, *, out_dir: str = "out", **_ignored: object) -> None:
        self._out_dir = out_dir

    async def on_job_start(self, job) -> None:
        ...

    async def on_job_end(self, job_result) -> None:
        ...
```

`BaseJobPlugin` is an ABC in `harbor.models.job.plugin`. `JobPlugin` is the
runtime-checkable Protocol Harbor validates against, so any object with both
methods is accepted; subclassing the base class is the clearer route.

Harbor rejects an object failing that check at attach time, before any trial runs.

## Registration

Harbor discovers plugins through the `harbor.plugins` entry point group. Declare
it in your package metadata:

```toml
[project.entry-points."harbor.plugins"]
myplugin = "my_package.plugin:MyPlugin"
```

Installing the package registers the plugin. There is nothing to import or call.

```bash
harbor plugins list          # confirm it resolved
harbor run --path task --agent oracle --plugin myplugin
```

`--plugin` also accepts a literal `module:Class` path, which is useful before
publishing. Both forms go through `resolve_plugin_import_path`.

## Configuration

Pass options with `--plugin-kwarg key=value`. Values are parsed as JSON, so
numbers, booleans, lists and objects all work:

```bash
harbor run ... --plugin myplugin \
  --plugin-kwarg out_dir=results \
  --plugin-kwarg attempts=[1,5] \
  --plugin-kwarg verbose=true
```

With several `--plugin` flags, prefix the key with the plugin name:
`--plugin-kwarg myplugin.out_dir=results`.

Kwargs reach `__init__` directly. **Accept `**_ignored`** so an unrecognised key
does not abort a job that has already spent money on agent calls. A construction
failure raises before any trial runs, which is the most expensive moment to fail.

## Streaming With Trial Hooks

`on_job_end` fires once, at the end. To react as trials finish, subscribe during
`on_job_start`:

```python
async def on_job_start(self, job) -> None:
    job.on_trial_ended(self._trial_finished)

async def _trial_finished(self, event) -> None:
    result = event.result          # a TrialResult
    name = event.result.trial_name
```

`Job` exposes `on_trial_started`, `on_environment_started`, `on_agent_started`,
`on_agent_ended`, `on_verification_started`, `on_trial_ended` and
`on_trial_cancelled`. Each takes an async callback receiving a `TrialHookEvent`.

Streaming is what lets a cancelled or crashed job leave usable output behind.

## Two Correctness Traps

**Retries reuse the trial id.** Harbor retries a failed trial under the same
`TrialResult.id`. Appending each hook call to a list double counts the trial and
inflates your denominator. Key by `result.id` in a dict so a retry replaces its
earlier attempt.

**A resumed job only reports what it replayed.** `JobResult.trial_results` is
authoritative for the current run, but a resume does not re-run trials that
already completed. If you streamed records earlier, union them with the final
results rather than replacing, or the suite silently shrinks.

## Fail-Open

A plugin failure must never take down a job. Harbor guards `on_job_end`, but not
`on_job_start` and not your own hook callbacks, and a raise inside a trial hook
propagates into the trial.

Wrap every callback body, log, and degrade to a no-op. By the time a plugin runs,
the job has already paid for agent calls; losing that to a reporting bug is the
worst possible trade.

## Reading the Job Directory

Plugins that read persisted output need the real filenames. These differ from
what some docstrings and template comments say, verified against Harbor 0.22.0:

| What | Actual path | Note |
|------|-------------|------|
| Verifier reward, single value | `/logs/verifier/reward.txt` | Parsed as one float under key `reward` |
| Verifier reward, named keys | `/logs/verifier/reward.json` | **Singular.** The task template comment says `rewards.json`, which is wrong |
| Per-trial result | `<job_dir>/<trial_name>/result.json` | **Singular.** `TrialPaths` documents `results.json` |
| Job result | `<job_dir>/result.json` | Written **without** trial results |

The last row matters most. Harbor writes the final job `result.json` with
`exclude_trial_results=True`, so the file has no `trial_results` key at all.
Code that backfills from it alone reads zero trials and reports success. Read the
per-trial files and treat the job result as a fallback.

See `references/job-directory.md` for the full layout and a working backfill
example.

## What You Receive

`on_job_start` gets a `Job`: `job.id`, `job.config.job_name`, `job.job_dir`,
`len(job)` for planned trials, and the hook subscriptions above.

`on_job_end` gets a `JobResult`: `id`, `started_at`, `finished_at`,
`n_total_trials`, `stats`, and `trial_results`.

Each `TrialResult` carries `id`, `trial_name`, `task_name`, `task_checksum`,
`source` (the dataset, absent for ad hoc runs), `agent_info`, `verifier_result`,
`exception_info`, four `TimingInfo` spans, and `compute_token_cost_totals()`.

Note that `verifier_result` and its `rewards` are both optional. A trial can
finish cleanly with no reward attached, which is a missing verdict rather than a
failure. See `references/models.md` for the full field reference.

## Checklist

- [ ] Subclass `BaseJobPlugin`, implement both methods
- [ ] Declare the `harbor.plugins` entry point
- [ ] Accept `**_ignored` in `__init__`
- [ ] Guard every callback so failures degrade to no-ops
- [ ] Key streamed trials by `result.id` to survive retries
- [ ] Union streamed records with final results to survive resumes
- [ ] Read `reward.json` and per-trial `result.json`, both singular
- [ ] Verify with `harbor plugins list`, then a run with `--agent oracle`

Running against the `oracle` agent exercises the full pass path in real
containers with no model spend. `nop` gives you the fail path.
