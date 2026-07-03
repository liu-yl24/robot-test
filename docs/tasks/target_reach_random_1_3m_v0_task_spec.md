# target_reach_random_1_3m_v0 Task Spec

Task gate: D1 for `target_reach_random_1_3m_v0`

Owner: A

Status: draft task specification, no motion approval

Created: 2026-07-03

## 1. Task Identity

| Field | Value |
| --- | --- |
| Task name | `target_reach_random_1_3m_v0` |
| Task type | A-only motion task |
| Platform | Kuavo 5-W simulation platform |
| Execution model | episode runner only |
| First phase | low-speed, single-target, observable, safe-stop-required |
| Web execution | forbidden |
| B/C execution | forbidden |
| Benchmark execution | forbidden until A approves prior gates |

This task is not a read-only check, Web health check, or general demo script.
It is a motion task and must be handled as an A-controlled safety gate.

## 2. Task Goal

For one episode:

```text
Sample one target point in the world frame around the robot start position.
Move the robot toward that target.
Stop the robot safely when the task ends or fails.
Record enough artifacts for A safety review, B metrics analysis, and C read-only display.
```

The intended behavior is:

```text
start_pose -> sampled target_pose within 1-3 m -> final_pose near target_pose -> safe_stop
```

## 3. Sampling Specification

The v0 target is sampled in a planar annulus around the episode start position:

```text
r     in [1.0, 3.0]
theta in [-pi, pi]
frame = world
target_x = start_x + r * cos(theta)
target_y = start_y + r * sin(theta)
target_yaw = unchanged or null for v0
```

Rules:

```text
targets_per_episode = 1
seed must be recorded
sampled radius must be recorded
sampled theta must be recorded
target_pose must be recorded
target_out_of_bounds must fail before motion
```

The first motion stage must not start from this random sampler. The required
progression is:

```text
target_reach_random_1_3m_dry_run_v0
target_reach_fixed_1m_v0
target_reach_fixed_distance_v0
target_reach_random_1_3m_v0 single run
target_reach_random_1_3m_5
target_reach_random_1_3m_20
```

## 4. Initial Parameters

| Parameter | Initial value |
| --- | ---: |
| `radius_min_m` | 1.0 |
| `radius_max_m` | 3.0 |
| `reach_threshold_m` | 0.15 |
| `max_speed_mps` | 0.20-0.25 |
| `max_duration_sec` | 45-60 |
| `targets_per_episode` | 1 |
| `safe_stop_on_exit` | true |
| `operator` | A |
| `permission_level` | `motion_a_only` |
| `frame` | `world` |

If fixed 1 m is unstable, random 1-3 m is not allowed.

If one random target is unstable, 5-run and 20-run benchmarks are not allowed.

If safe_stop is unstable, all motion benchmark progression stops.

## 5. Required Config Semantics

The eventual episode config must express these constraints:

```yaml
schema_version: "0.2"
task_name: target_reach_random_1_3m_v0
entry_type: scenario
operator: A
permission_level: motion_a_only
timeout_sec: 60

safety:
  allow_motion: true
  safe_stop_on_exit: true
  max_speed_mps: 0.25
  max_duration_sec: 60

target_reach:
  mode: random_annulus
  frame: world
  radius_min_m: 1.0
  radius_max_m: 3.0
  reach_threshold_m: 0.15
  targets_per_episode: 1
  seed: required
```

This block is a specification, not approval to add or run the config.

## 6. Execution Constraints

Mandatory:

```text
Must run through scripts/run_episode.py or the equivalent episode runner API.
Must generate run_id.
Must write outputs/episodes/<run_id>/.
Must write metrics.json, result.json, status.json, latency_breakdown.json, stdout.log, stderr.log.
Must write task_metrics.json or an explicitly documented equivalent.
Must write safe_stop.json.
Must record start_pose, target_pose, final_pose.
Must record seed, radius_m, theta_rad.
Must record timeout and failure_reason.
Must safe_stop on success, failure, timeout, or exception.
```

Forbidden:

```text
No direct Web call to /cmd_vel.
No Web endpoint that accepts target_pose and executes motion.
No Web "Run random target" button.
No arbitrary Python execution path for this task.
No benchmark runner bypassing the episode runner.
No continuous random cruising.
No missing artifact success claim.
No B/C motion run.
No direct dialogue-generated control code.
```

## 7. Required Artifacts

At minimum, a completed or failed target reach episode must contain:

| Artifact | Required content |
| --- | --- |
| `manifest.json` | run_id, task_name, operator, permission_level, artifact list, git info |
| `config.yaml` | original episode config |
| `resolved_config.yaml` | resolved config used by the runner |
| `status.json` | final status and stage |
| `result.json` | run summary and ok flag |
| `metrics.json` | generic episode metrics and safe_stop fields |
| `latency_breakdown.json` | runner and executor timing |
| `stdout.log` | executor stdout |
| `stderr.log` | executor stderr |
| `events.jsonl` | lifecycle events |
| `safe_stop.json` | attempted, ok, exit_code, failure_reason |
| `task_metrics.json` | target reach task metrics |

`task_metrics.json` must include:

```text
schema_version
run_id
task_name
seed
frame
start_pose
target_pose
final_pose
radius_m
theta_rad
position_error_m
reach_threshold_m
duration_sec
max_duration_sec
timeout
max_speed_mps
final_velocity_norm
stop_threshold
safe_stop_ok
mode_restored
run_success
task_success
failure_reason
```

## 8. Success Semantics

`run_success` and `task_success` must stay separate.

`run_success` means the episode runner and executor completed according to their
generic run contract.

`task_success` means the robot satisfied the target reach task contract:

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

## 9. Failure Reasons

Initial `failure_reason` values:

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

Failures must be safe by default. A failed task still needs complete artifacts
and safe_stop evidence.

## 10. A/B/C Responsibility Boundary

### A

A owns:

```text
motion approval
target range safety
max speed and max duration limits
fixed target to random target promotion
single run to benchmark promotion
safe_stop acceptance
failure-path acceptance
motion PR review
```

Only A can approve the first real motion run.

### B

B owns:

```text
metrics schema review
task_metrics.json analysis
summary.csv / summary.md design
latency and benchmark reporting
fixed distance comparison
random benchmark statistics after A approval
```

B cannot run this motion task unless A explicitly approves the exact stage and
operator path.

### C

C owns:

```text
read-only Web display
artifact viewer validation
operator guide updates
illegal artifact / illegal run_id checks
ensuring Web has no motion trigger for this task
```

C cannot add run buttons, target input execution, restore execution, or `/cmd_vel`
paths for this task.

## 11. Stage Gates

This task must advance in order:

| Gate | Required outcome |
| --- | --- |
| D0 baseline | v0.2 baseline approved |
| D1 task spec | this task spec reviewed and frozen |
| D2A safety | A safety and acceptance doc complete |
| D3 dry-run | target generation and artifacts pass with no motion |
| D4 fixed_1m | A-approved 1 m fixed target motion passes |
| D5 fixed distances | 1/2/3 m results are documented |
| D6 random single | A-approved single random target passes |
| D7 random benchmark | A-approved 5-run then 20-run benchmark passes |
| D8 Web validation | read-only Web display works and no motion entry exists |

Skipping gates is not allowed.

## 12. D1 Exit Rule

D1 is complete only when A confirms:

```text
Task name and type are frozen.
Sampling range is frozen for v0.
Initial speed, duration, and reach threshold are frozen for first implementation.
Episode runner is mandatory.
safe_stop_on_exit is mandatory.
Web execution is forbidden.
B/C execution is forbidden.
run_success and task_success are separate.
task_metrics.json required fields are accepted for B analysis and C display.
```

Current D1 decision:

```text
Not approved yet. This document is the proposed baseline specification.
```
