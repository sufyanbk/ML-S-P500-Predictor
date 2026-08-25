# Weekly Model Plan Spark Optimisation Experiment Register

## 1. Purpose

This document records the controlled Spark optimisation experiments performed against the Weekly Model Plan.

The objectives are to:

- Reduce end-to-end runtime.
- Reduce unnecessary spill and repeated task execution.
- Improve runtime stability.
- Understand behaviour under concurrent execution.
- Establish an evidence-based Spark profile for future Model Plan runs.
- Separate Spark tuning issues from infrastructure/platform issues.

**Spark version:** 3.5.7  
**Primary benchmark workload:** 50% data volume, full Model Plan  
**Primary production-scale reference:** 100% data volume, full Model Plan

---

## 2. Problem Statement

The Weekly Model Plan has shown three operational problems:

1. High runtime.
2. Random failures, particularly when other workloads run in parallel.
3. High resource consumption and cost.

The optimisation work therefore focuses on two distinct areas:

- **Performance:** reduce runtime and computational inefficiency.
- **Stability:** reduce failed work, executor loss, shuffle failures, and sensitivity to contention.

---

## 3. Baseline Configuration

| Property | Value |
|---|---:|
| Job profile | SK-MIG12 |
| Data volume | 50% |
| Plan scope | Full Model Plan |
| Manual partitions | 20,000 |
| `spark.executor.instances` | 150 |
| `spark.executor.cores` | 6 |
| `spark.executor.memory` | 45G |
| `spark.executor.memoryOverhead` | 20G |
| `spark.memory.fraction` | 0.3 |
| `spark.memory.storageFraction` | 0.1 |
| `spark.dynamicAllocation.enabled` | false |
| `spark.reducer.maxBlocksInFlightPerAddress` | 5 |
| `spark.shuffle.service.enabled` | false |
| `spark.network.timeout` | 3600s |

### Baseline runtime references

| Workload | Runtime |
|---|---:|
| 10% full Model Plan | ~2h |
| 25% full Model Plan | ~3h |
| 50% full Model Plan | 5h37 |
| 100% full Model Plan | 12h23 |

The runtime increase is non-linear as data volume grows.

---

## 4. Experiment Summary

| Experiment | Research Question | Controlled Change | Result | Decision | Main Conclusion |
|---|---|---|---|---|---|
| Baseline | What is the current performance profile? | None | 5h37 at 50% | Reference | Establishes the control configuration |
| Experiment 1 | Does more Spark execution memory reduce spill and runtime? | `spark.memory.fraction` 0.3 → 0.6 | Better | KEEP | More execution memory materially reduced spill and runtime |
| Experiment 2 | Does increasing partition count reduce the extreme task/shuffle tail? | 20K → 30K partitions | Worse | REJECT | Normal partitions became smaller, but the pathological partition remained |
| Experiment 3 | Does limiting remote shuffle requests improve stability? | Added `spark.reducer.maxReqsInFlight=20` | Worse | REJECT | Shuffle throttling reduced throughput without improving the underlying issue |
| Experiment 4 | What happens when two full-strength Model Plans run concurrently? | Two winning-profile jobs run in parallel | In progress | Pending | Establishes the contention baseline for parallel execution |

---

# 5. Metric Reference

| Metric | Definition | Why It Matters |
|---|---|---|
| Full runtime | Wall-clock time from application start to completion | Primary operational and cost measure |
| Stage duration | Wall-clock duration of a Spark stage | Identifies which processing phase dominates runtime |
| Total task time | Sum of runtime across all tasks | Measures total compute effort, even when tasks run concurrently |
| Median task duration | Runtime of the middle task | Represents normal task behaviour |
| P75 task duration | 75% of tasks complete within this time | Shows whether slowdown affects a broad portion of tasks |
| Max task duration | Runtime of the longest task | Measures long-tail/straggler behaviour |
| Shuffle | Redistribution of intermediate data between Spark workers | Required by grouping and aggregation, and creates network/memory/disk pressure |
| Shuffle read | Intermediate data read by downstream tasks | Measures receiving-stage data movement |
| Shuffle write | Intermediate data produced for downstream tasks | Measures upstream shuffle output |
| Median shuffle read per task | Typical shuffle data processed by one task | Represents normal partition size |
| Max shuffle read per task | Largest shuffle data processed by one task | Key indicator of skew or extreme partitions |
| Memory spill | Execution data Spark had to spill because it could not remain in execution memory | High values indicate execution-memory pressure |
| Disk spill | Spill physically written to local disk | Adds slower disk I/O to the workload |
| Peak execution memory | Highest task execution-memory requirement | Shows working-memory demand |
| JVM heap usage | Executor Java heap used | Used to assess executor memory sizing |
| GC time | JVM garbage-collection time | High values indicate memory-management overhead |
| Failed task attempts | Tasks that failed and had to retry | Represents wasted compute and instability |
| Failed stage attempt | Whole stage attempt rerun after failure | High-cost failure mode |
| Dead executor | Executor process terminated or disappeared | Relevant to stability and lost shuffle output |
| Lost node | YARN worker disappeared from the cluster | Can cause shuffle-output loss |
| `INTERNAL_ERROR_NETWORK` | Network/remote communication failure | Relevant to shuffle instability |
| `MetadataFetchFailedException` | Spark could not retrieve required shuffle output metadata | Often associated with lost shuffle output |
| Running containers | Active YARN containers | Indicates cluster resource consumption |
| Allocated vCores | CPU capacity allocated by YARN | Shows reserved compute capacity |
| Queue percentage | Share of YARN queue resources allocated | Indicates contention risk, not CPU utilisation |
| Active nodes | Worker nodes participating in the cluster | Useful for capacity and node-loss analysis |

---

# 6. Experiment 1 — Increase Spark Execution Memory

## Hypothesis

Increasing `spark.memory.fraction` from 0.3 to 0.6 will provide more heap space for execution operations such as aggregation, sorting, and shuffle processing.

Expected effects:

- Lower spill.
- Lower task time.
- Lower stage runtime.
- Lower end-to-end runtime.

## Controlled Change

```text
spark.memory.fraction: 0.3 -> 0.6
```

All other benchmark settings remained unchanged.

## Results

The 50% full Model Plan completed twice at **5h07**, compared with the original **5h37** baseline.

This is a repeatable runtime reduction of approximately **30 minutes**, or **8.9%**.

### Stage 2 Comparison

| Metric | Baseline 0.3 | Experiment 1, 0.6 | Change | Assessment |
|---|---:|---:|---:|---|
| Full runtime | 5h37 | 5h07 | -30 min | Better |
| Stage 2 duration | ~2.8h | 2.7h | Lower | Better |
| Total Stage 2 task time | 1,559.8h | 1,445.5h | -7.3% | Better |
| Shuffle read | 1,060.6 GiB | 1,060.6 GiB | No change | Neutral |
| Memory spill | 106.1 GiB | 78.3 GiB | -26.2% | Much better |
| Disk spill | 8.8 GiB | 6.5 GiB | -26.1% | Much better |
| Median task | 4.6 min | 4.4 min | Lower | Better |
| P75 task | 5.2 min | 4.8 min | Lower | Better |
| Max task | 2.3h | 2.3h | No change | No improvement |
| Max shuffle read/task | ~2.6 GiB | ~2.6 GiB | No change | No improvement |

## Interpretation

The amount of shuffle data did not change. The workload still moved approximately 1.06 TiB in the expensive stage.

The improvement came from executing the same workload more efficiently.

More execution memory reduced memory and disk spill by approximately 26% and reduced aggregate Stage 2 task time.

The extreme long-running task remained, so memory pressure was only one part of the performance problem.

## Decision

**KEEP**

```text
spark.memory.fraction = 0.6
```

This becomes part of the new control profile.

---

# 7. Experiment 2 — Increase Manual Partitions

## Hypothesis

Increasing manual partitions from 20,000 to 30,000 will reduce the amount of data handled by each task.

Expected effects:

- Smaller shuffle partitions.
- Lower spill.
- Shorter long-tail tasks.
- Lower Stage 2 runtime.

## Controlled Change

```text
manual partitions: 20,000 -> 30,000
```

All other settings remained on the Experiment 1 winning profile.

## Results

| Metric | 20K Control | 30K | Change | Assessment |
|---|---:|---:|---:|---|
| Full runtime | 5h07 | ~6h | ~+53 min | Much worse |
| Stage 0 | 25 min | 27 min | Higher | Worse |
| Stage 1 | 1.9h | 2.3h | Higher | Worse |
| Stage 2 | 2.7h | 3.2h | Higher | Much worse |
| Stage 2 total task time | 1,445.5h | 1,861.6h | +28.8% | Much worse |
| Median task | 4.4 min | 3.7 min | Lower | Better |
| P75 task | 4.8 min | 4.2 min | Lower | Better |
| Max task | 2.3h | 2.1h | Slightly lower | Slightly better |
| Median shuffle read/task | 53.6 MiB | 38.8 MiB | -27.6% | Better |
| Max shuffle read/task | 2.6 GiB | 2.6 GiB | No change | No improvement |
| Memory spill | 78.3 GiB | 76.1 GiB | Small reduction | Slightly better |
| Disk spill | 6.5 GiB | 6.2 GiB | Small reduction | Slightly better |
| Stage 0 failed task attempts | 279 | 941 | Large increase | Much worse |
| Stage 1 failed task attempts | 641 | 950 | Increase | Worse |
| Stage 2 failed task attempts | 612 | 756 | Increase | Worse |

## Interpretation

Increasing partition count worked for normal partitions.

The median Stage 2 shuffle partition fell from 53.6 MiB to 38.8 MiB.

However, the largest partition remained approximately **2.6 GiB**.

This is the key result.

The experiment created 50% more tasks without breaking down the pathological partition.

The additional task overhead increased aggregate task time and total runtime.

The evidence indicates that the remaining tail is not primarily caused by an insufficient global partition count.

It is consistent with persistent data/grouping skew.

## Decision

**REJECT**

Return to:

```text
manual partitions = 20,000
```

Do not continue partition-count tuning without new evidence.

---

# 8. Experiment 3 — Limit Outstanding Shuffle Requests

## Hypothesis

The workload performs TiB-scale shuffle across a large cluster.

Unrestricted remote shuffle request concurrency might contribute to network pressure and worker instability.

Limiting outstanding remote requests might improve resilience.

## Controlled Change

Added:

```text
spark.reducer.maxReqsInFlight = 20
```

All other settings remained on the winning profile.

## Results

| Metric | Winning Control | Experiment 3 | Change | Assessment |
|---|---:|---:|---:|---|
| Stage 0 | 25 min | 26 min | Higher | Worse |
| Stage 1 | 1.9h | 2.0h | Higher | Worse |
| Stage 2 | 2.7h | 3.4h | ~+42 min | Much worse |
| Stage 2 total task time | 1,445.5h | 1,553.8h | +7.5% | Worse |
| Median Stage 2 task | 4.4 min | 4.7 min | Higher | Worse |
| P75 Stage 2 task | 4.8 min | 5.2 min | Higher | Worse |
| Max Stage 2 task | 2.3h | 2.9h | Higher | Much worse |
| Shuffle read | 1,060.6 GiB | 1,060.6 GiB | No change | Neutral |
| Median shuffle read/task | 53.6 MiB | 53.6 MiB | No change | Neutral |
| Max shuffle read/task | 2.6 GiB | 2.6 GiB | No change | No improvement |
| Memory spill | 78.3 GiB | 79.9 GiB | Higher | Worse |
| Disk spill | 6.5 GiB | 6.5 GiB | No change | No improvement |
| Stage 0 failed task attempts | 279 | ~304 | Higher | Worse |
| Stage 1 failed task attempts | 641 | ~709 | Higher | Worse |
| Stage 2 failed task attempts | 612 | 612 | No change | No improvement |

## Interpretation

The property throttled how aggressively reduce tasks fetched remote shuffle blocks.

The workload still had to move approximately 1.06 TiB.

The restriction reduced throughput without reducing the pathological 2.6 GiB partition.

It did not materially improve spill or failed-task behaviour.

## Decision

**REJECT**

Remove:

```text
spark.reducer.maxReqsInFlight = 20
```

Do not carry this property into future benchmark profiles.

---

# 9. Current Winning Standalone Profile

| Property | Winning Value |
|---|---:|
| Base profile | SK-MIG12 |
| Manual partitions | 20,000 |
| `spark.memory.fraction` | 0.6 |
| `spark.memory.storageFraction` | 0.1 |
| `spark.executor.instances` | 150 |
| `spark.executor.cores` | 6 |
| `spark.executor.memory` | 45G |
| `spark.executor.memoryOverhead` | 20G |
| `spark.dynamicAllocation.enabled` | false |
| `spark.reducer.maxBlocksInFlightPerAddress` | 5 |
| `spark.reducer.maxReqsInFlight` | Not set |
| `spark.shuffle.service.enabled` | false |
| `spark.network.timeout` | 3600s |

### Measured 50% Standalone Runtime

**5h07**

Observed twice.

---

# 10. Executor and Worker Capacity Findings

## Worker Capacity

| Resource | Per Worker |
|---|---:|
| YARN memory | ~204.8 GiB |
| vCores | 30 |

## Current Executor Shape

| Resource | Per Executor |
|---|---:|
| Heap | 45G |
| Memory overhead | 20G |
| Cores | 6 |

Three executor containers consume approximately:

```text
Memory: 3 x 65G = 195G
CPU:    3 x 6   = 18 cores
```

This initially suggested that CPU could be stranded because memory prevents a fourth executor from fitting on the worker.

However, executor history showed the busiest executors reaching approximately **39.6–39.9 GiB JVM heap usage**.

That represents approximately **88–89% of the configured 45G heap**.

## Conclusion

Reducing executor heap simply to pack more executors onto each worker is not justified by the observed utilisation.

The 20G memory overhead remains worth reviewing with the platform/vendor team because it materially affects YARN packing, but it should not be reduced based only on Spark UI off-heap statistics.

---

# 11. Infrastructure Stability Finding

A successful 5h07 application still showed significant executor churn.

Observed:

- 98 dead executors.
- Failed task attempts.
- Containers released on `lost` nodes.
- Exit status `-100`.
- Historical `INTERNAL_ERROR_NETWORK`.
- Historical `MetadataFetchFailedException`.

## Interpretation

The 5h07 run is application-successful, but it is not infrastructure-clean.

This is a separate issue from Spark performance tuning.

## Platform Team RCA Required

The platform workstream should determine:

1. Why YARN worker nodes are being marked as lost.
2. Why containers are being released with exit status `-100`.
3. Whether node loss is related to resource pressure, infrastructure lifecycle, networking, or another platform mechanism.
4. Whether shuffle-output loss is a consequence of those node removals.
5. Whether the configured 20G executor memory overhead is required.
6. Whether platform-level shuffle preservation/decommissioning mechanisms are available and appropriate.

---

# 12. Experiment 4 — Parallel Contention Baseline

## Status

**IN PROGRESS**

## Research Question

What happens when two Model Plans using the winning standalone profile execute concurrently?

## Configuration

Both jobs use:

| Property | Value |
|---|---:|
| `spark.memory.fraction` | 0.6 |
| Manual partitions | 20K |
| Executors | 150 |
| Executor cores | 6 |
| Executor memory | 45G |
| Memory overhead | 20G |
| `spark.reducer.maxReqsInFlight` | Not set |

## Hypothesis

A single Model Plan previously consumed approximately:

- 151 running containers.
- 901 allocated vCores.
- 89.8% of the YARN queue.
- ~52 active nodes.

Two full-strength Model Plans are therefore expected to create substantial queue contention.

The purpose of this experiment is to quantify:

1. Runtime degradation.
2. Stability degradation.
3. Total time required for both Model Plans to complete.
4. Queue behaviour.
5. Executor/node failure behaviour.
6. Whether contention causes disproportionate degradation.

## Metrics to Capture

| Metric | Standalone Reference | Parallel Result |
|---|---:|---:|
| Plan A runtime | 5h07 | Pending |
| Plan B runtime | 5h07 equivalent control | Pending |
| Time until both plans complete | ~10h14 sequential reference | Pending |
| Stage 0 duration | 25 min | Pending |
| Stage 1 duration | 1.9h | Pending |
| Stage 2 duration | 2.7h | Pending |
| Failed task attempts | Baseline recorded | Pending |
| Failed stage attempts | 0 in reference application | Pending |
| Dead executors | Baseline recorded | Pending |
| Queue allocation | ~89.8% with one plan | Pending |
| Running containers | ~151 with one plan | Pending |
| Allocated vCores | ~901 with one plan | Pending |
| Active nodes | ~52 observed | Pending |
| Network/fetch errors | Historical occurrence | Pending |

## Decision Logic

If two 150-executor applications become disproportionately slower or materially less stable, the next experiment will target **resource allocation per concurrent application**.

The optimisation objective then changes from:

> Fastest individual Model Plan

to:

> Lowest total completion time and highest stability for multiple concurrent Model Plans.

A reduced-executor parallel profile should only be tested after the current contention baseline is measured.

---

# 13. Current Research Position

## Layer 1 — Execution Memory

**Result:** Improved

`spark.memory.fraction=0.6` reduced spill and runtime.

## Layer 2 — Partition Count

**Result:** Rejected

30K partitions reduced normal partition size but did not reduce the pathological 2.6 GiB partition.

## Layer 3 — Shuffle Request Throttling

**Result:** Rejected

`maxReqsInFlight=20` reduced throughput without improving the underlying issue.

## Layer 4 — Parallel Resource Contention

**Status:** In progress

The current experiment measures the impact of two full-strength Model Plans sharing the same YARN queue.

---

# 14. Next Steps

1. Complete Experiment 4 and record both application results.
2. Determine whether parallel slowdown is proportional or disproportionate.
3. If required, design a reduced-executor parallel profile to improve aggregate throughput.
4. Run the platform RCA for lost worker nodes and executor loss.
5. Once the final profile is selected, validate it at 100% data volume.
6. Compare the final 100% result against the existing **12h23** production-scale baseline.

---

# 15. Experiment Documentation Standard

Every future experiment must record:

1. Research question.
2. Hypothesis.
3. Control configuration.
4. Single controlled variable.
5. Data volume.
6. Plan scope.
7. Runtime.
8. Stage-level runtime.
9. Shuffle read/write.
10. Median and max task duration.
11. Median and max shuffle partition.
12. Memory and disk spill.
13. Failed task attempts.
14. Failed stage attempts.
15. Dead executors.
16. Node/network errors.
17. Interpretation.
18. Decision.
19. Resulting next question.

Only one primary variable should be changed per controlled benchmark wherever possible.

---

## Change Log

| Version | Change |
|---|---|
| 0.1 | Initial baseline captured |
| 0.2 | Experiment 1 added, memory fraction 0.6 retained |
| 0.3 | Experiment 2 added, 30K partitions rejected |
| 0.4 | Experiment 3 added, `maxReqsInFlight=20` rejected |
| 0.5 | Executor capacity analysis added |
| 0.6 | Parallel contention experiment added |
