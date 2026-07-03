# target_reach_random_1_3m_v0 — B Benchmark & Metrics Doc

Task gate: D2B for `target_reach_random_1_3m_v0`

Owner: B

Status: draft B metrics & benchmark specification, no motion approval

Created: 2026-07-03

Builds on:

```text
D0: docs/tasks/v0.2-stable-baseline-checklist.md
D1: docs/tasks/target_reach_random_1_3m_v0_task_spec.md
```

## 1. Document Purpose

This document is owned by B. It defines the **metrics, schema, and benchmark
reporting** for `target_reach_random_1_3m_v0` so that a motion task can answer
more than "it ran":

```text
Did the robot actually reach the target?
What is the error?
Did it time out?
Was safe_stop reliable?
Why did it fail?
Is the result reproducible?
Is there evidence to advance to the next stage?
```

B's core job is **not** to control the robot. B's core job is to make the
quality of a run measurable, comparable, and auditable.

This is an A-only motion task. B may prepare the dry-run design, the
`task_metrics.json` schema, the benchmark summary fields, the summary
templates, and the analysis/reporting logic. B may **not** run any real motion
task and may **not** create or approve a motion experiment config.

All field names, the success formula, the failure-reason enum, and the
parameters below are copied from D1 (`target_reach_random_1_3m_v0_task_spec.md`).
B does not invent new task fields here; B specifies how those fields are
aggregated and reported.

## 2. Prerequisite State

D0 and D1 are merged but **not signed off** (both are `not approved yet`). Per
D0 §7, until D0 signoff:

```text
Do not run fixed_1m.
Do not run random_1_3m.
Do not create Web motion controls.
Do not let B/C run motion benchmark configs.
```

Per D1 §4 the frozen initial parameters are:

```text
radius_min_m        = 1.0
radius_max_m        = 3.0
reach_threshold_m   = 0.15
max_speed_mps       = 0.20-0.25
max_duration_sec    = 45-60
targets_per_episode = 1
safe_stop_on_exit   = true
operator            = A
permission_level    = motion_a_only
frame               = world
```

Per D1 §3 the sampler is a planar annulus in the world frame:

```text
r     in [1.0, 3.0]
theta in [-pi, pi]
target_x = start_x + r * cos(theta)
target_y = start_y + r * sin(theta)
target_yaw = unchanged or null for v0
```

Everything in this document must stay inside these boundaries.

## 3. B's Responsibility Scope

B owns:

```text
1. task_metrics.json schema (field-by-field spec)
2. run_success vs task_success distinction
3. task_success decision formula (quoted from D1)
4. failure_reason enumeration (quoted from D1)
5. dry-run benchmark design (generation-only, no motion)
6. motion benchmark summary fields
7. benchmark config maintenance (dry-run template only)
8. summary.json / summary.csv / summary.md aggregation design
9. analysis of position_error / timeout / safe_stop / duration
10. data-driven recommendation to A on stage advancement
```

B does **not** own:

```text
robot motion control
first motion acceptance
safe_stop implementation
restore implementation
Web motion entry
KuavoSim control layer
PowerShell launch chain
```

B must never substitute "the program exited 0" for "the task succeeded".

## 4. B's Permission Boundary

### 4.1 B may run

```text
fast_health_check
read_only_interfaces
full_interfaces_check
health_check_20 benchmark
interfaces_check_10 benchmark
target_reach_random_1_3m_dry_run benchmark   (after A provides the dry-run experiment config)
analyze_episode_latency.py
summary generation scripts
```

### 4.2 B must NOT run (unless A explicitly approves the exact stage)

```text
base_probe
move_for
scenario motion
/cmd_vel
restore
target_reach_random_1_3m_v0 real motion
target_reach_random_1_3m_5 motion benchmark
target_reach_random_1_3m_20 motion benchmark
```

### 4.3 B may modify

```text
docs/tasks/target_reach_random_1_3m_v0_B_benchmark_and_metrics.md   (this file)
configs/benchmarks/*dry_run*.yaml.template
non-motion summary/aggregation logic in scripts/run_benchmark.py
read-only analysis logic in scripts/analyze_episode_latency.py
benchmark summary documents
CSV / Markdown summary scripts
```

### 4.4 B must send to A for review

```text
any experiment config with allow_motion = true
any motion benchmark config
any code that could trigger target_reach / move_for / /cmd_vel
any change to safe_stop, restore, or the KuavoSim control layer
```

### 4.5 D0 policy caveat (flagged for A)

D0 §5 records that `configs/policies/v0.2_policy.yaml` currently gives B/C
`allow_motion: true` and lists `base_probe` as an allowed task. D0 states this
is **too broad** for `target_reach_random_1_3m_v0`. Before any target-reach
motion config lands, A must either:

```text
1. narrow B/C motion policy for this task, or
2. add a task-specific permission level (motion_a_only) that only A can run.
```

B will treat `motion_a_only` as A-only regardless of the policy file and will
not author any config that relies on the current broad B/C motion permission.

## 5. Core Concept: run_success vs task_success

These are separate and must not be collapsed (D1 §8).

| Field | Meaning |
| --- | --- |
| `run_success` | the episode runner and executor completed per their generic run contract |
| `task_success` | the robot satisfied the target-reach task contract |
| `safe_stop_ok` | the robot stopped safely after the task ended or failed |
| `benchmark_success` | a batch of runs met the statistical gate (§13) |

The dangerous case B must always flag:

```text
run_success  = true
task_success = false
```

This means the episode did not crash, but the robot did **not** reach the
target per the task contract. This must be counted as a task failure in every
report; it must never be written up as a success.

| run_success | task_success | Interpretation |
| --- | --- | --- |
| true | true | genuine success |
| true | false | ran cleanly but did not reach target — task failure |
| false | false | run itself failed — task failure, investigate run path |
| false | true | impossible by definition (task_success requires run_success) |

## 6. task_metrics.json Schema

Each target-reach episode writes `outputs/episodes/<run_id>/task_metrics.json`.
The required fields are exactly those in D1 §7. B adds no new fields; B only
specifies types, semantics, and null handling.

### 6.1 Example payload

```json
{
  "schema_version": "0.2",
  "run_id": "20260703_120000_target_reach_random_1_3m_v0_a1b2",
  "task_name": "target_reach_random_1_3m_v0",
  "seed": 12345,
  "frame": "world",
  "start_pose": { "x": 0.0, "y": 0.0, "yaw": 0.0 },
  "target_pose": { "x": 1.82, "y": -0.74, "yaw": null },
  "final_pose": { "x": 1.71, "y": -0.65, "yaw": 0.03 },
  "radius_m": 1.96,
  "theta_rad": -0.386,
  "position_error_m": 0.142,
  "reach_threshold_m": 0.15,
  "duration_sec": 18.2,
  "max_duration_sec": 60.0,
  "timeout": false,
  "max_speed_mps": 0.25,
  "final_velocity_norm": 0.01,
  "stop_threshold": 0.03,
  "safe_stop_ok": true,
  "mode_restored": true,
  "run_success": true,
  "task_success": true,
  "failure_reason": "none"
}
```

### 6.2 Field specification

| Field | Type | Notes |
| --- | --- | --- |
| `schema_version` | string | `"0.2"` |
| `run_id` | string | matches the episode `run_id` in `metrics.json` |
| `task_name` | string | `target_reach_random_1_3m_v0` |
| `seed` | int | the seed used for sampling; must be reproducible |
| `frame` | string | `world` for v0 |
| `start_pose` | object | `{x, y, yaw}`; the robot pose at episode start |
| `target_pose` | object | `{x, y, yaw}`; sampled target in `frame` |
| `final_pose` | object | `{x, y, yaw}`; observed robot pose at task end |
| `radius_m` | float | sampled radius `r in [1.0, 3.0]` |
| `theta_rad` | float | sampled angle `theta in [-pi, pi]` |
| `position_error_m` | float | euclidean distance `final_pose -> target_pose` (xy) |
| `reach_threshold_m` | float | success threshold, initial `0.15` |
| `duration_sec` | float | actual task duration |
| `max_duration_sec` | float | initial `45-60`, frozen per-run by A |
| `timeout` | bool | true if `duration_sec > max_duration_sec` |
| `max_speed_mps` | float | initial `0.20-0.25`, frozen per-run by A |
| `final_velocity_norm` | float | robot speed norm at task end |
| `stop_threshold` | float | velocity stop threshold, initial `0.03` |
| `safe_stop_ok` | bool | from `safe_stop.json` |
| `mode_restored` | bool | control mode restored after task |
| `run_success` | bool | generic run/executor contract result |
| `task_success` | bool | computed by the formula in §7 |
| `failure_reason` | string | primary reason from the enum in §8 (`none` on success) |

### 6.3 Null / missing-observation rules

```text
yaw is not a target in v0 -> target_pose.yaw must be null (do not fake yaw success).
final_pose missing or unobserved -> task_success = false, failure_reason = pose_observation_missing (or the D1 enum equivalent; see §8 note).
safe_stop_ok = false -> task_success = false even if position_error_m is within threshold.
position_error_m cannot be computed -> task_success = false.
```

If `task_metrics.json` itself is missing from an episode, the summary must
count that run as `artifact_incomplete` (see §8) and exclude it from
position-error percentiles, not silently treat it as success.

## 7. task_success Decision Formula

Quoted verbatim from D1 §8:

```text
task_success =
  run_success == true
  AND timeout == false
  AND position_error_m <= reach_threshold_m
  AND safe_stop_ok == true
  AND mode_restored == true
  AND final_velocity_norm <= stop_threshold
```

Exit code 0 alone is not sufficient.

### 7.1 Default thresholds (D1 §4)

| Metric | Initial threshold |
| --- | ---: |
| `reach_threshold_m` | 0.15 |
| `stop_threshold` | 0.03 |
| `max_duration_sec` | 45-60 |
| `safe_stop_ok` | must be true |
| `mode_restored` | must be true |

### 7.2 Tightening policy

Thresholds may be tightened only after benchmark data supports it, never by
feel:

```text
reach_threshold_m:      0.15 -> 0.10
p90_position_error_m:   0.30 -> 0.20
timeout_rate:           10%  -> 5%
```

Any threshold change must cite the benchmark summary it is based on.

## 8. failure_reason Enumeration

B maintains a single enum so failures aggregate cleanly. The values are those
in D1 §9. The summary in §12 uses `primary_failure_reason` for rollup and
`failure_reasons` (list) for post-mortem.

### 8.1 Enum (from D1 §9)

```text
target_out_of_bounds
policy_denied
preflight_failed
executor_failed
timeout
position_error_exceeded
safe_stop_failed
mode_restore_failed
final_velocity_nonzero
artifact_incomplete
operator_not_A
web_motion_forbidden
benchmark_gate_not_approved
unknown
```

On success the value is `none`.

### 8.2 Multi-reason recording

A run may have several contributing reasons. Record both:

```json
{
  "primary_failure_reason": "timeout",
  "failure_reasons": ["timeout", "position_error_exceeded"]
}
```

`primary_failure_reason` is what the benchmark summary counts.
`failure_reasons` is the full list for复盘.

### 8.3 Mapping note for D2B field names

D1 §7 lists a single `failure_reason` field on `task_metrics.json`. D2B
recommends the writer populate `failure_reason` with the **primary** reason and
also emit `failure_reasons` (list) when more than one applies, so the summary
aggregation in §12 has a stable single key (`primary_failure_reason`, falling
back to `failure_reason`). If the writer can only emit one, use `failure_reason`.

A D1 §9 alias worth noting: D1 also references `pose_observation_missing` in
prose; treat it as `artifact_incomplete` for v0 unless A adds it to the enum.
B will not add enum values unilaterally.

## 9. Dry-run Benchmark Design

Real motion is gated behind D3 (dry-run). B prepares the dry-run design now;
it must not move the robot.

### 9.1 Names

```text
task name:      target_reach_random_1_3m_dry_run_v0
benchmark name: target_reach_random_1_3m_dry_run_20
```

### 9.2 What the dry-run verifies (generation only)

```text
target generation (r, theta sampling)
seed reproducibility (same seed -> same target)
r / theta distribution sanity
target_x / target_y computation
target_in_bounds check
artifact completeness
summary generation
```

### 9.3 What the dry-run must NOT do

```text
publish /cmd_vel
call move_for
run scenario motion
call restore
trigger Web motion
change robot state
```

The dry-run is a `check`-type experiment (not `scenario`), so it does not
trigger the benchmark runner's motion gate (`MOTION_TASK_KEYWORDS` =
`base_probe`, `base_forward`, `move_for`, `scenario`). The experiment config
itself is provided by A at the D3 gate; B does not author it.

### 9.4 Dry-run summary fields

```text
total_runs
target_generation_success_rate
invalid_target_rate
seed_reproducibility_ok
artifact_complete_rate
mean_radius_m
min_radius_m
max_radius_m
mean_theta_rad
target_in_bounds_rate
```

### 9.5 Dry-run pass criteria

```text
target_generation_success_rate = 100%
invalid_target_rate            = 0%
artifact_complete_rate         = 100%
seed_reproducibility_ok        = true
no motion commands emitted
```

The dry-run benchmark config template is provided as
`configs/benchmarks/target_reach_random_1_3m_dry_run_20.yaml.template`
(disabled by the `.template` suffix; see that file).

## 10. Fixed-distance Stage Metrics

Per D1 §3 the required progression before random single-run is:

```text
target_reach_random_1_3m_dry_run_v0
target_reach_fixed_1m_v0
target_reach_fixed_distance_v0
target_reach_random_1_3m_v0 single run
target_reach_random_1_3m_5
target_reach_random_1_3m_20
```

For the fixed-distance stage B's report table must include at least:

| distance_m | run_id | run_success | task_success | position_error_m | duration_sec | timeout | safe_stop_ok | failure_reason |
| ---: | --- | --- | --- | ---: | ---: | --- | --- | --- |
| 1.0 | pending | pending | pending | pending | pending | pending | pending | pending |
| 2.0 | pending | pending | pending | pending | pending | pending | pending | pending |
| 3.0 | pending | pending | pending | pending | pending | pending | pending | pending |

B must answer from the fixed-distance data:

```text
Does error grow with distance?
Are there timeouts?
Is safe_stop stable?
Is there obvious directional drift?
Is it safe to advance to random directions?
```

These motion runs are A-only; B fills the table from A-provided run_ids.

## 11. Random Single-run Report

After A completes the first approved random 1-3 m run, B produces a single-run
analysis from the `run_id`.

Save as:

```text
docs/tasks/target_reach_random_1_3m_v0_single_run_report.md
```

Content must include:

```text
run_id
seed
start_pose
target_pose
final_pose
radius_m
theta_rad
position_error_m
duration_sec
timeout
safe_stop_ok
mode_restored
final_velocity_norm
task_success
failure_reason
latency summary
A acceptance conclusion (quoted)
B data conclusion
```

B's conclusion must state explicitly:

```text
whether the run meets the bar to start the 5-run benchmark
whether reach_threshold_m needs adjustment
whether pose observation is missing
whether there is a safe_stop risk
```

## 12. Motion Benchmark Summary Fields

Motion benchmarks are A-approved only. Recommended order:

```text
target_reach_random_1_3m_5  ->  target_reach_random_1_3m_20
```

Do not start at 20.

The motion benchmark `summary.json` must include at least:

```text
benchmark_id
approved_by_A
operator
total_runs
run_ids
run_success_rate
task_success_rate
safe_stop_success_rate
timeout_rate
artifact_complete_rate
mean_position_error_m
p50_position_error_m
p90_position_error_m
max_position_error_m
mean_duration_sec
p50_duration_sec
p90_duration_sec
failure_reason_counts
external_timing_available_rate
```

If trajectory data is available, additionally:

```text
mean_path_length_m
mean_path_efficiency
p90_path_efficiency
max_overshoot_m
mean_final_velocity_norm
```

`failure_reason_counts` is a mapping of enum value -> count, e.g.:

```json
{ "timeout": 2, "position_error_exceeded": 1, "none": 17 }
```

Note (implementation, not part of this doc PR): the current
`scripts/run_benchmark.py` `summarize_benchmark()` only computes generic
latency/throughput aggregates from `metrics.json["ok"]`. Adding the
task-level fields above requires extending that function to read
`task_metrics.json` per run. That code change is allowed under §4.3
("non-motion summary/aggregation logic") but is intentionally **out of scope**
for this doc PR and will be a follow-up once a real `task_metrics.json` exists
to aggregate.

## 13. Benchmark Pass Criteria

### 13.1 Initial 5-run benchmark

```text
total_runs              = 5
task_success_rate       >= 80%
safe_stop_success_rate  = 100%
timeout_rate            <= 20%
artifact_complete_rate  = 100%
no safe_stop_failed
```

### 13.2 Initial 20-run benchmark

```text
total_runs              = 20
task_success_rate       >= 80%
safe_stop_success_rate  = 100%
timeout_rate            <= 10%
p90_position_error_m    <= 0.30
artifact_complete_rate  = 100%
```

### 13.3 Stable target

```text
task_success_rate       >= 90%
safe_stop_success_rate  = 100%
timeout_rate            <= 5%
p90_position_error_m    <= 0.20
artifact_complete_rate  = 100%
```

If `safe_stop_success_rate < 100%`, advancing to the next stage is forbidden.

## 14. Summary File Format

Benchmark output stays under the existing layout:

```text
outputs/benchmarks/<benchmark_id>/
  benchmark_config.yaml
  run_ids.txt
  summary.json
  summary.csv
  summary.md
```

### 14.1 summary.md template

```text
# Benchmark Summary: target_reach_random_1_3m_20

## Metadata

- benchmark_id:
- date:
- operator:
- approved_by_A:
- config:
- total_runs:

## Core Results

- run_success_rate:
- task_success_rate:
- safe_stop_success_rate:
- timeout_rate:
- artifact_complete_rate:

## Error Metrics

- mean_position_error_m:
- p50_position_error_m:
- p90_position_error_m:
- max_position_error_m:

## Timing Metrics

- mean_duration_sec:
- p50_duration_sec:
- p90_duration_sec:

## Failure Reasons

- timeout:
- position_error_exceeded:
- safe_stop_failed:
- artifact_incomplete:
- unknown:

## Conclusion

- meets pass criteria:
- recommend next stage:
- primary failure reason:
- recommended adjustments:
```

### 14.2 summary.csv

One header row + one data row, field order:

```text
benchmark_id,operator,total_runs,run_success_rate,task_success_rate,
safe_stop_success_rate,timeout_rate,artifact_complete_rate,
mean_position_error_m,p50_position_error_m,p90_position_error_m,
max_position_error_m,mean_duration_sec,p50_duration_sec,p90_duration_sec,
external_timing_available_rate
```

## 15. B's Stage-judgement Format For A

Every B report ends with an explicit judgement for A:

```text
Stage judgement: pass / fail / conditional pass

Evidence:
1. task_success_rate          =
2. safe_stop_success_rate     =
3. timeout_rate               =
4. p90_position_error_m       =
5. primary failure reason     =

Recommendation:
- advance to next stage:
- parameter adjustment needed:
- observation gap to close:
- pause motion benchmark:
```

"测试完成。" alone is not an acceptable conclusion.

## 16. Completion Checklist (D2B exit rule)

```text
[ ] task_metrics.json schema complete (D1 §7 fields, no invented fields)
[ ] run_success / task_success distinction explicit
[ ] task_success formula explicit (quoted from D1 §8)
[ ] failure_reason enum explicit (quoted from D1 §9)
[ ] dry-run benchmark design explicit (generation-only)
[ ] motion benchmark summary fields explicit
[ ] summary.md / summary.csv templates explicit
[ ] pass criteria and stage gates explicit
[ ] B's no-motion boundary explicit
[ ] items requiring A review explicit
```

## 17. Final Principle

```text
B is responsible for proving whether the system moves accurately,
stably, and quickly, and where it fails.
```

B does not move the robot. B proves how well it moved.
