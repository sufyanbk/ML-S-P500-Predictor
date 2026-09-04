# DSE Capacity vs Model Generation Workload

## Overview

The workload running within the Data Science Environment, DSE, has changed as our fraud models have evolved.

Each new model generation brings a different processing requirement. The number of features can change. The type of features can change. Historical data requirements can increase. Data volumes can grow. Feature generation can become more complex.

DSE capacity does not change automatically when these workloads change.

DSE operated with **50 workers until February 2026**, when capacity increased to **100 workers**.

That was a significant uplift. However, model development has continued to evolve since the environment was originally sized and since the capacity increase was made.

The key question is:

> **Is DSE capacity keeping pace with the workload being introduced by each new model generation?**

---

## The Problem

The DSE is being asked to support changing workloads while the underlying environment changes much less frequently.

Model generations continue to evolve:

**OB4 → FE8 → FV9 → FV10 → Future Generations**

DSE capacity changes separately:

**50 Workers → 100 Workers → Current Capacity**

A new model generation does not automatically trigger a corresponding change in DSE capacity.

This means the environment is expected to absorb new workload until processing or capacity constraints become visible.

The issue is therefore not one Feature Plan, one failed run or one model generation.

> **The issue is that model-generation workload continues to evolve, while DSE capacity is upgraded at separate points in time.**

Without a clearer link between the two, the same capacity problem is likely to return.

---

## Feature Plan and Weekly Model Plan

| Process | What it does | Type of workload |
|---|---|---|
| **Feature Plan** | Generates feature values for a future model generation using historical data | Model development workload |
| **Weekly Model Plan** | Refreshes feature values already required by established models using new weekly data and required history | Recurring workload |

A Feature Plan supports development of the next model generation.

The Weekly Model Plan supports the established model estate.

Both use DSE resources.

---

## Why Workload Changes Between Model Generations

Each model generation has different requirements.

The workload therefore cannot be assumed to be the same from one generation to the next.

| Area | What can change | Impact on DSE |
|---|---|---|
| **Feature volume** | More features are generated or evaluated | More processing |
| **Feature complexity** | Features use more complex calculations, joins or aggregations | More compute and memory |
| **Historical data** | Longer periods of historical data are required | More data to process |
| **Data volume** | More transactions or customers are included | Larger processing footprint |
| **Execution approach** | Work needs to be split into several runs | Longer processing and more operational effort |
| **Reruns** | Failed or incomplete runs need to be repeated | Additional resource usage |
| **Weekly workload** | More established features need to be refreshed | Higher recurring demand |

This is why final feature count alone does not explain DSE workload.

Two models with a similar final feature count can create very different workloads during development.

---

## Example: FV9

FV9 provides an example of how workload can change.

The FV9 Feature Plan used approximately **three years of historical data**.

Processing the full dataset together resulted in crashes.

The workload therefore had to be split into smaller parts and the outputs later combined.

Existing FV8 feature data also had to be joined with the generated FV9 data.

Conceptually:

```text
3 Years of Historical Data
        ↓
FV9 Feature Generation
        ↓
Full Processing Too Large
        ↓
Split into Smaller Runs
        ↓
Generate Outputs
        ↓
Combine Outputs
        ↓
Join Required FV8 Features
        ↓
Final Development Dataset
```

This is not simply a question of how many FV9 features existed.

The key question is what DSE had to do to generate them.

---

# 1. DSE Capacity vs Model Generation

This should become the main stakeholder view.

| Model / Period | DSE Workers | Feature Workload | Historical Requirement | Processing Approach | Main Observation |
|---|---:|---|---|---|---|
| **OB4** | TBD | TBD | TBD | TBD | TBD |
| **FE8** | TBD | TBD | TBD | TBD | TBD |
| **FV9** | 50 / transition to confirm | TBD | ~3 years | Partitioned processing | Full-volume crashes observed |
| **February 2026** | **100** | Capacity uplift | N/A | N/A | Workers increased from 50 to 100 |
| **FV10** | **100** | TBD | TBD | TBD | Current workload to establish |
| **Future Model** | Current capacity | TBD | TBD | TBD | Future requirement |

### What this should show

The comparison is not simply whether capacity increased.

It should show whether the increase in capacity has kept pace with the change in workload.

---

# 2. Feature Plan Workload Evolution

| Measure | OB4 | FE8 | FV9 | FV10 |
|---|---:|---:|---:|---:|
| Features submitted for generation | TBD | TBD | TBD | TBD |
| Historical period processed | TBD | TBD | ~3 years | TBD |
| Approximate data volume | TBD | TBD | TBD | TBD |
| Existing feature datasets required | TBD | TBD | FV8 | TBD |
| Number of processing parts | TBD | TBD | Multiple | TBD |
| Failed runs / reruns | TBD | TBD | Observed | TBD |
| End-to-end generation time | TBD | TBD | TBD | TBD |

The question this view answers is:

> **What did DSE have to process to generate the feature data required for each model generation?**

---

# 3. Weekly Model Plan Evolution

| Measure | Model Plan V4 | V5 | V6 | V7 | V8 |
|---|---:|---:|---:|---:|---:|
| Feature estate supported | TBD | TBD | TBD | TBD | TBD |
| Features calculated | TBD | TBD | TBD | TBD | TBD |
| Historical data requirement | TBD | TBD | TBD | TBD | TBD |
| Typical execution time | TBD | TBD | TBD | TBD | TBD |
| Peak execution time | TBD | TBD | TBD | TBD | TBD |
| Constraints observed | TBD | TBD | TBD | TBD | TBD |

The question this view answers is:

> **Is the Weekly Model Plan running today comparable with the workload that was running when earlier DSE capacity decisions were made?**

---

# 4. Final Model Evolution

| Measure | OB4 | FE8 | FV9 | FV10 |
|---|---:|---:|---:|---:|
| Final feature count | TBD | TBD | TBD | TBD |
| Stable VDR | TBD | TBD | TBD | TBD |
| Stable FPR | TBD | TBD | TBD | TBD |
| Features considered during development | TBD | TBD | TBD | TBD |

This view gives useful context, but final feature count should not be used as the main measure of DSE workload.

---

## Why the Problem Repeats

```text
DSE Capacity Is Set
        ↓
A New Model Generation Is Developed
        ↓
The Workload Changes
        ↓
DSE Absorbs the New Workload
        ↓
Processing or Capacity Constraints Appear
        ↓
The Environment or Process Is Adjusted
        ↓
The Next Model Generation Arrives
        ↓
The Workload Changes Again
```

This means we risk treating each issue as a separate problem when the underlying cause is the same.

The model-development roadmap and DSE capacity are not being planned as one connected capacity problem.

---

## What Needs to Change

| Lever | Question |
|---|---|
| **DSE Capacity** | Does the environment have enough resources and headroom for the current and next model generation? |
| **Workload Optimisation** | Are Feature Plans and Weekly Model Plans being executed in the most efficient way? |
| **Model Generation Cadence** | Are we introducing model generations at a pace the environment can support? |

The answer is not automatically more DSE capacity.

It may be more capacity, better processing, a different model-generation cadence, or a combination of all three.

---

## Target Position

Before a new model generation begins, we should understand:

- What workload already exists in DSE
- What new workload the generation introduces
- How much historical data will be processed
- How many features need to be generated
- Whether complex or expensive processing is being introduced
- Whether the current environment has enough headroom
- Whether the workload needs to be optimised first
- Whether the model-generation timeline is realistic against available capacity

This creates a direct link between model-generation planning and DSE capacity planning.

---

## Key Message

> **DSE capacity has increased, but model-generation workload continues to evolve. Each generation changes the amount and type of processing the environment needs to support. If capacity planning does not move with model-generation planning, DSE will continue to reach the same constraints as workloads become larger or more complex.**
