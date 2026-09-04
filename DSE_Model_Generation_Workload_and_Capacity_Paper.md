# Keeping DSE Capacity Aligned with Model Generation Workload

## The Challenge

The Data Science Environment, DSE, supports a workload that changes with each new model generation.

The environment itself does not change at the same rate.

DSE operated with **50 workers until February 2026**, when capacity increased to **100 workers**. While this doubled worker capacity, the workloads introduced through model development have continued to change.

The problem is therefore not simply whether DSE has more capacity than before.

The problem is whether the capacity available today remains appropriate for the workload being introduced today.

---

## Model Generations Do Not Create the Same Workload

OB4, FE8, FV9 and FV10 should not be treated as equivalent processing workloads.

Each generation can differ in several ways.

### Feature volume

A new generation may introduce a larger set of candidate features.

More features mean more calculations and larger intermediate outputs.

### Feature complexity

Not every feature has the same processing cost.

Some features use simple calculations. Others rely on larger aggregations, joins, mappings or behavioural logic.

Feature count therefore does not describe the full workload.

### Historical depth

A Feature Plan may need to process a longer period of historical data.

This increases the amount of data DSE must read, transform and calculate.

FV9, for example, used approximately **three years of historical data**.

### Data volume

Even when the feature logic remains similar, larger customer and transaction populations increase the processing footprint.

### Execution pattern

Larger workloads may no longer fit into one execution.

For FV9, full-volume processing encountered crashes. The workload had to be divided into smaller runs and the outputs later combined.

This adds processing steps, elapsed time and operational effort.

### Recurring demand

Once feature generations become part of the established model estate, they also affect the recurring feature processing carried out through the Weekly Model Plan.

DSE therefore supports both new model development and the recurring processing of the existing model estate.

---

## The Capacity Gap

The core issue is the timing of change.

Model workload changes with each generation.

DSE capacity changes at separate points in time.

```text
50 Workers
    ↓
New model-development workloads introduced
    ↓
More complex processing requirements
    ↓
February 2026
    ↓
100 Workers
    ↓
Further model generations introduced
    ↓
Workload changes again
```

Capacity is increased based on one point in the model lifecycle, while the modelling workload continues to evolve afterwards.

This creates a recurring gap.

---

## Why This Is Likely to Continue

If the next model generation introduces:

- More features
- More expensive features
- Longer historical data
- Higher transaction volumes
- More dataset joins
- More processing stages
- More recurring feature calculations

then the workload changes again.

If DSE remains unchanged, the environment is once again being asked to absorb a workload that differs from the one against which its capacity was previously assessed.

This is why the issue is not specific to FV9 or FV10.

It is a consequence of model development evolving independently from DSE capacity.

---

## What We Need to Establish

| Area | Comparison |
|---|---|
| **DSE capacity** | Workers and available resource at the time |
| **Feature generation** | Number and type of features processed |
| **Historical requirement** | Period of data processed |
| **Data scale** | Volume of records processed |
| **Execution approach** | Single run, partitioned runs, reruns |
| **Weekly processing** | Recurring workload after features become established |
| **Final model** | Final feature count, VDR and FPR |

This will allow us to compare the workload DSE was supporting previously with the workload it is supporting now.

---

## The Decision

The response should focus on three areas.

### 1. Capacity

Does DSE have enough resources and headroom to support the current and next model generation?

### 2. Optimisation

Are there parts of the Feature Plan or Weekly Model Plan that should be redesigned to reduce unnecessary processing?

### 3. Model Generation Cadence

Should future model generations be sequenced differently where the available environment cannot sustainably support the planned workload?

The answer may involve all three.

---

## Target Position

Model-generation planning and DSE capacity planning should operate together.

Before introducing a new model generation, the expected processing requirement should be understood and compared with the capacity already in use.

This gives us an informed decision before the workload reaches the environment.

---

## Core Message

> **The DSE is not supporting a fixed workload. Every model generation can change the scale, complexity and historical processing required. Capacity has increased from 50 to 100 workers, but the workload has continued to evolve. Unless DSE capacity, workload optimisation and model-generation planning move together, the same constraints will continue to return.**
