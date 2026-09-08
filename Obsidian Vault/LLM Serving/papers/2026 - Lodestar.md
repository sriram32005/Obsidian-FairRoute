---
title: "Lodestar: An Online-Learning LLM Inference Router"
authors:
  - Jiaxin Shan Le Xu Liguang Xie
  - Gangmuk Lim† Wanyu Zhao Brighten Godfrey
year: 31 May 2026
venue: ""
paper_type: research
status: unread
rating:
url: ""
pdf: ""
tags:
---


# 2026 - Lodestar

> [!abstract] One-Line Summary  
> Write the entire paper's contribution in **1–2 sentences**.

---

## 1. Paper Metadata

| Field                   | Notes                                                           |
| ----------------------- | --------------------------------------------------------------- |
| **Authors**             |                                                                 |
| **Year**                |                                                                 |
| **Venue**               |                                                                 |
| **Paper Type**          | System / Algorithm / Scheduling / Routing / Simulation / Theory |
| **Code Available?**     | Yes / No                                                        |
| **Artifact Available?** | Yes / No                                                        |
| **Related System**      | vLLM / Sarathi / Custom / Other                                 |

---

# 2. Problem

## Problem Statement

LLM inference workloads require fast response to queries as they are part of interactive applications, but they have drastically different characteristics than traditional latency-sensitive applications such as CPU-based user-facing cloud services and various supporting services like key-value stores and databases

 Even within the LLM inference workloads, different workloads exhibit highly variable computational requirements, ranging from short conversational exchanges to long document understanding  to complex reasoning tasks  involving highly dynamic input and output token ranges.

**What precise problem does this paper solve?**

How to map each incoming inference request to one of the many instances (or “pods” in Kubernetes) within a cluster, which we refer to as inference routing.


## Why Do Existing Approaches Fail?

Traditional load balancing techniques, like round robin or variants of Join the Shortest Queue (JSQ), seek to balance queue length but could result in significant inefficiency and under-utilization, because they do not take into account that there will be significant differences in the resources needed to run different requests and also to run an inference request on different instances at different moments in time.

---

# High Level Solution

Lodestar, an online learning-based, cloud-native distributed request routing system for LLM inference. It collects detailed cluster states and aims to leverage them with the power of deep learning to navigate the complex environment. 

*Lodestar learns to accurately predict latency that would result from routing a particular inference request to a particular instance, given the request information and cluster state at the time of routing. Then, it applies this predictor to route requests to the instance that maximizes predicted reward such as negative of time-to-first-token (TTFT).*

Two Key Challenges: 
1. **Accurate online latency prediction:** Modeling per-request reward in complex LLM inference system which depends on serving engine, model size, mode arch, network, hardware, workload etc... is hard. Circular dependency exists between routing policy, cluster state and request latency profile which is also hard to model. 
2. **Efficient and reliable cloud-scale routing system**: Lodestar introduces additonal overhead of collecting real-time cluster state, preparing model inputs and running complex routing logic for each request. 

**Main Reward: TTFT**
The reward predictor is a neural network that takes the request features and current cluster state as input and for each serving instance, predicts the reward. 
Lodestar learns more accurate reward predictor overtime by adapting to current cluster configuration. 


---

# Core Contribution

## Core Insight

> What is the **one central idea** that distinguishes this paper?
> Uses neural network to predict the reward for routing better instead of simple linear regression estimation of the cost in previous papers

## Main Contributions

-  First public per-request LLM routing dataset pairing cluster snapshots, request features, and latency collected from public-cloud clusters.
- Design of a new distributed inference routing system, Lodestar, that is adaptable to the dynamic environment via online learning. 

---

# 5. Algorithm Classification

## Algorithm Tags

- [ ]  Fairness
    
- [ ]  Locality
    
- [ ]  Prediction
    
- [ ]  Load Balancing
    
- [ ]  SLO-aware
    
- [ ]  Throughput Optimization
    
- [ ]  Latency Optimization
    
- [ ]  KV Cache Aware
    
- [ ]  Resource Allocation
    
- [ ]  Admission Control
    
- [ ]  Migration
    
- [ ]  Batching
    
- [ ]  Scheduling
    
- [ ]  Other:
    

## Combination

**Primary combination:**

`Fairness / Locality / Prediction / ...`

---

# 6. Input Signals

## What Does the Algorithm Observe?

|Signal|Source|Why Is It Needed?|
|---|---|---|
|Queue Length|||
|KV Cache Occupancy|||
|Prefix Similarity|||
|Token Count|||
|Tenant ID|||
|Historical Latency|||
|GPU Utilization|||
|Predicted Load|||
|Other|||

## Signal Collection Overhead

- [ ] Centralized or distributed?
    
- [ ] Push or pull telemetry?
    
- [ ] Update frequency:
    
- [ ] Metadata size:
    
- [ ] Potential bottleneck:
    

---

# 7. Decision

## Decision Variable

**What exactly does the system decide?**

- [ ]  Which replica receives a request
    
- [ ]  Which request executes next
    
- [ ]  Which requests form a batch
    
- [ ]  Whether to admit a request
    
- [ ]  Whether to migrate a request
    
- [ ]  Resource allocation
    
- [ ]  Other:
    

## Decision Granularity

- Per request:
    
- Per token:
    
- Per batch:
    
- Periodic:
    
- Event driven:
    

---

# 8. Algorithm

## High-Level Workflow

```text
Request / Event
        ↓
Collect Signals
        ↓
Decision Logic
        ↓
Select Action
        ↓
Update System State
```

Replace with the paper's actual workflow.

## Algorithm Steps

### Step 1

### Step 2

### Step 3

### Step 4

## Pseudocode / Formula

```text
Add simplified pseudocode or equations here.
```

## Time Complexity

- Decision complexity:
    
- State update complexity:
    
- Memory complexity:
    

---

# 9. State Maintained

|State|Scope|Update Rule|Purpose|
|---|---|---|---|
||Per-request / Per-tenant / Per-replica / Global|||

Examples to investigate:

- Queues
    
- Virtual time
    
- Deficit counters
    
- Token counters
    
- Prefix trees
    
- KV-cache metadata
    
- Prediction history
    
- Load estimates
    

---

# 10. Objective

## What Is Being Optimized?

- [ ]  Throughput
    
- [ ]  TTFT
    
- [ ]  TPOT
    
- [ ]  End-to-End Latency
    
- [ ]  Tail Latency
    
- [ ]  Fairness
    
- [ ]  Cache Hit Rate
    
- [ ]  GPU Utilization
    
- [ ]  SLO Attainment
    
- [ ]  Cost
    
- [ ]  Other:
    

## Formal Objective

```text
Write the paper's objective function here.
```

## Optimization Tradeoff

> What gets worse when the optimized metric improves?

---

# 11. Assumptions and Constraints

## Assumptions

- [ ]  Homogeneous GPUs
    
- [ ]  Heterogeneous GPUs
    
- [ ]  Centralized Controller
    
- [ ]  Distributed Controller
    
- [ ]  Known Prompt Length
    
- [ ]  Known Output Length
    
- [ ]  Prefix Information Available
    
- [ ]  Accurate Prediction Available
    
- [ ]  Stable Workload
    
- [ ]  Static Cluster
    
- [ ]  Other:
    

## Hidden Assumptions

> What assumptions are necessary for the algorithm to work but are not heavily emphasized?

## Constraints

---

# 12. Experimental Methodology

## Experimental Workloads

|Dataset / Trace|Characteristics|Why Used?|
|---|---|---|
||||

## Workload Generation

- Arrival process:
    
- Tenant distribution:
    
- Prompt distribution:
    
- Output distribution:
    
- Prefix overlap:
    
- Burst injection:
    
- Synthetic modifications:
    

---

# 13. Simulation Setup

## Simulator

- Simulator:
    
- Version:
    
- What does it model?
    
- What does it NOT model?
    

## Simulated Cluster

|Parameter|Value|
|---|---|
|Number of GPUs||
|GPU Type||
|Number of Replicas||
|Model||
|GPU Memory||
|Network||
|CPU||

## Simulation Assumptions

---

# 14. Real Hardware Setup

## Hardware

|Component|Configuration|
|---|---|
|GPU||
|GPU Memory||
|Number of GPUs||
|CPU||
|RAM||
|Storage||
|Network||

## Serving Framework

- Framework:
    
- Version:
    
- Model:
    
- Precision:
    
- Tensor Parallelism:
    
- Number of replicas:
    
- Scheduler configuration:
    
- KV cache configuration:
    

---

# 15. Metrics

|Metric|Definition|Why Important?|
|---|---|---|
|Throughput|||
|TTFT|||
|TPOT|||
|P50|||
|P95|||
|P99|||
|Jain's Fairness Index|||
|Cache Hit Rate|||
|GPU Utilization|||
|SLO Attainment|||

## Metric Formulas

```text
Add important formulas here.
```

---

# 16. Baselines

|Baseline|Why Compared?|Strong / Weak?|
|---|---|---|
||||

## Missing Baselines?

> What baseline should have been included but was not?

---

# 17. Key Results

## Main Quantitative Results

|Experiment|Result|Improvement|
|---|---|---|
||||

## Most Important Figure

**Figure:**

**What it shows:**

**Why it matters:**

## Main Takeaway

---

# 18. Failure Cases and Limitations

## Where Does It Fail?

## Limitations Stated by Authors

## Limitations I Identified

## Scalability Concerns

Consider:

- CPU overhead
    
- Memory overhead
    
- Network overhead
    
- Metadata overhead
    
- Centralized bottleneck
    
- Number of tenants
    
- Number of replicas
    

---

# 19. Challenges and Engineering Problems

|Challenge|Cause|Solution Used|
|---|---|---|
||||

## Failure / Edge Cases

---

# 20. Comparison With FairRoute

## Which Signal Does This Paper Optimize?

|Fairness|Locality|Prediction|
|---|---|---|
|Yes / No|Yes / No|Yes / No|

## What Can FairRoute Reuse?

## What Does FairRoute Need Beyond This?

## Threat to FairRoute

> Could this paper invalidate the FairRoute research gap or make the proposed contribution trivial?

## How Could This Paper Be Extended to Compete With FairRoute?

---

# 21. Baseline Decision

## Should We Implement This?

-  Yes, directly
    
-  Simplified version
    
-  Use as conceptual baseline only
    
-  Not needed
    

### Reason

## Comparison Priority

`High / Medium / Low`

---

# 22. Implementation Difficulty

## Difficulty

`Easy / Medium / Hard`

## Why?

## Dependencies

## Estimated Components Needed

```text
Component 1
Component 2
Component 3
```

---

# 23. Evidence

## Important Sections

|Claim / Observation|Page|Section / Figure / Table|
|---|---|---|
||||

## Important Quotes

> Keep only short quotes necessary for later verification.

---

# 24. Final Research Notes

## What I Learned

## Open Questions

## Research Opportunities

## Connection to Other Papers

- [[Paper Name]]
    
- [[Paper Name]]
    

---

# 25. Paper Verdict

## One-Sentence Verdict

## Importance to FairRoute

⭐ / ⭐⭐ / ⭐⭐⭐ / ⭐⭐⭐⭐ / ⭐⭐⭐⭐⭐

## Deep Dive Required?

- [ ]  Yes
    
- [ ]  No
    

## Read Again Before Implementation?

- [ ]  Yes
    
- [ ]  No