# Models a Plugin Receives

Field reference for the objects passed to plugin callbacks. Verified against
Harbor 0.22.0.

## Job

Passed to `on_job_start`.

| Member | Meaning |
|--------|---------|
| `job.id` | UUID, stable across the run and recovered on resume |
| `job.config` | `JobConfig`. `job.config.job_name` is the run's name |
| `job.job_dir` | `jobs_dir / job_name`, where output belongs |
| `len(job)` | Number of planned trials |
| `job.on_trial_started(cb)` | Subscribe to trial start |
| `job.on_environment_started(cb)` | Container ready |
| `job.on_agent_started(cb)` | Agent phase begins |
| `job.on_agent_ended(cb)` | Agent phase ends |
| `job.on_verification_started(cb)` | Verifier begins |
| `job.on_trial_ended(cb)` | Trial finished, result populated |
| `job.on_trial_cancelled(cb)` | Trial cancelled |

Each subscription takes an async callable receiving one `TrialHookEvent` and
returns the job, so calls chain.

## TrialHookEvent

| Field | Meaning |
|-------|---------|
| `event` | A `TrialEvent` enum member |
| `task_name` | The task this trial ran |
| `config` | `TrialConfig` |
| `result` | `TrialResult`, fully populated at `END` |
| `lock` | `TrialLock`, resolved inputs |
| `timestamp` | When the event fired |
| `trial_name` | Derived from `config` |
| `trial_id` | Derived from `result` |

## JobResult

Passed to `on_job_end`.

| Field | Meaning |
|-------|---------|
| `id` | Job UUID |
| `started_at`, `finished_at` | Run window |
| `n_total_trials` | Planned count |
| `stats` | `JobStats`, counts and per-eval aggregates |
| `trial_results` | Populated in memory; **absent from the persisted file** |

## TrialResult

| Field | Meaning |
|-------|---------|
| `id` | UUID. **Stable across retries of the same trial** |
| `trial_name` | Unique per attempt, includes a random suffix |
| `task_name` | The logical task |
| `task_checksum` | Hash of the task directory |
| `task_id` | `LocalTaskId`, `GitTaskId` or `PackageTaskId` |
| `source` | Dataset name, `None` for ad hoc runs |
| `config` | `TrialConfig` |
| `agent_info` | `.name`, `.version`, `.model_info` |
| `agent_result` | `AgentContext`, `None` on multi-step trials |
| `verifier_result` | `VerifierResult`, **optional** |
| `exception_info` | `.exception_type`, `.exception_message`, traceback |
| `started_at`, `finished_at` | Trial window |
| `environment_setup` | `TimingInfo` |
| `agent_setup` | `TimingInfo` |
| `agent_execution` | `TimingInfo` |
| `verifier` | `TimingInfo` |
| `step_results` | `list[StepResult]` on multi-step trials, else `None` |

`TimingInfo` has `started_at` and `finished_at`, either of which can be `None`.
Durations are derived, not stored.

## Rewards Are Optional

`verifier_result` may be `None`, and `verifier_result.rewards` may be `None` even
when the result exists. A trial can therefore finish cleanly with no reward
attached: the verifier failed to write one, the task does not score, or a
streaming consumer observed the trial before its verdict landed.

That is a **missing verdict, not a failure**. Treating it as a failure reports an
unscored run as a failed one, which is a different and much worse claim. A trial
that raised is a separate case and does count against a pass rate, because an
exception is a known negative outcome.

Three states are worth keeping distinct:

```python
graded = result.verifier_result is not None and result.verifier_result.rewards
errored = result.exception_info is not None
ungraded = not graded and not errored
```

## Tokens and Cost

`compute_token_cost_totals()` returns
`(n_input_tokens, n_cache_tokens, n_output_tokens, cost_usd)`, aggregating across
steps on a multi-step trial. Every element can be `None` when the agent reported
nothing.

**`n_input_tokens` is total input including cache.** Cache tokens are a subset,
not an additional bucket, so summing input and cache double counts. Harbor's own
field description states this.

## Multi-Step Trials

A multi-step trial leaves `agent_result` and `verifier_result` unset at the top
level and populates `step_results` instead, each `StepResult` carrying its own
`agent_result`, `verifier_result`, `exception_info` and timings. Handle both
shapes, or single-step assumptions will silently drop multi-step data.

## Reward Semantics

`VerifierResult.rewards` is `dict[str, float | int] | None`. Keys are chosen by
whoever wrote the task, so treat them as data rather than schema.

Convention is a single key named `reward`, which is what a `reward.txt` file
produces. Tasks writing `reward.json` choose their own keys, and a plugin
reducing them to one number should say which key it treated as the headline
rather than assuming.
