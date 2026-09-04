---
title: Fairness in Serving Large Language Models
authors:
  - Ying Sheng, Shiyi Cao, Dacheng Li, Banghua Zhu, Zhuohan Li, Danyang Zhuo, Joseph E. Gonzalez, Ion Stoica
year: "2024"
venue: OSDI 2024 (the 18th USENIX Symposium on Operating Systems Design and Implementation)
paper_type: research
status: reading
rating:
url: "[https://arxiv.org/abs/2401.00588](https://arxiv.org/abs/2401.00588)"
pdf: "[[2024 - Fairness in Serving Large Language Models.pdf]]"
tags:
  - fairness
---


# 2024 - Fairness in Serving Large Language Models

> [!abstract] One-Line Summary  
> This paper presents the first formal definition of fairness in multi-tenant large language model (LLM) serving and introduces the **Virtual Token Counter (VTC)** scheduler to handle unpredictable request lengths and asymmetric token costs. Operating at token-level granularity, VTC dynamically prioritizes clients with the least service received to guarantee robust performance isolation and a tight theoretical fairness bound while ensuring high, work-conserving GPU utilization.

---

## 1. Paper Metadata

| Field                   | Notes                                                                                                    |
| ----------------------- | -------------------------------------------------------------------------------------------------------- |
| **Authors**             | Ying Sheng, Shiyi Cao, Dacheng Li, Banghua Zhu, Zhuohan Li, Danyang Zhuo, Joseph E. Gonzalez, Ion Stoica |
| **Year**                | 2024                                                                                                     |
| **Venue**               | **OSDI 2024** (the **18th USENIX Symposium on Operating Systems Design and Implementation**)             |
| **Paper Type**          | System / Algorithm / Scheduling / Routing / Simulation / Theory                                          |
| **Code Available?**     | Yes  https://github.com/Ying1123/VTC-artifact                                                            |
| **Artifact Available?** | Yes                                                                                                      |
| **Related System**      | S-LoRA, \[LightLLM backbone - (Continuous Batching +  PagedAttention\]                                   |

---

# 2. Problem

## Problem Statement

## What precise problem does this paper solve?
The paper addresses the challenge of **fairness and performance isolation in multi-tenant Large Language Model (LLM) serving systems**. In shared inference environments, clients submit highly heterogeneous workloads (ranging from short chatbot interactions to massive document-analysis tasks). Due to this heterogeneity, existing schedulers fail to prevent high-demand or "ill-behaved" clients from monopolizing computational resources, which starves other clients and leads to extreme latency spikes or timeouts.
## Why Does This Problem Matter?
- **Resource Isolation**: A single high-demand tenant can easily monopolise shared servers, causing massive latency spikes or timeouts for other clients.
- **GPU Utilization vs. Cost**: Simple rate limiting solves isolation but prevents work-conservation, causing expensive and scarce GPUs to sit idle during periods of low activity.
- **Multi-Tenant Viability**: Modern frameworks (e.g., S-LoRA, Punica) serve thousands of personalized adapters on a shared server, making fair resource allocation crucial to SLA guarantees.

## Why Do Existing Approaches Fail?

Traditional fair schedulers cannot be applied to LLMs due to **three unique characteristics**:

1. **Unknown output lengths** before scheduling.
2. **Asymmetric costs** (sequential output decodes are far more expensive than parallelized input prefills).
3. **Fluctuating server capacity** (tokens/sec) depending on memory constraints and batch composition.

| Existing Approach                          | Limitation                                                       |
| ------------------------------------------ | ---------------------------------------------------------------- |
| First-Come-First-Serve (FCFS)              | No client isolation or fairness guarantees                       |
| Request-Per-Minute (RPM) Limits            | Non-work-conserving and causes resource under-utilization        |
| Traditional Fair Queueing (e.g., WFQ, SFQ) | Cannot handle unknown request lengths                            |
| Deficit Round Robin (DRR)                  | Cannot determine how many requests can fit in a scheduling round |
| Completely Fair Scheduler (CFS - CPU)      | Does not account for GPU batch concurrency                       |

---

# 3. System Context

## System Layer

- [ ]  Cluster Routing
    
- [x]  Request Scheduling (vtc scheduler)
    
- [x]  Continuous Batching (iteration level)
    
- [x]  Prefill Scheduling
    
- [x]  Decode Scheduling (based on counters)
    
- [ ]  KV Cache Management
    
- [x]  GPU Scheduling
    
- [ ]  Distributed Inference
    
- [ ]  Simulation / Evaluation Infrastructure
    
- [ ]  Other:
    

## Target Workload

- [x]  Multi-tenant
    
- [ ]  Prefix Sharing
    
- [x]  Bursty Traffic
    
- [ ]  SLO-aware
    
- [ ]  Long-context
    
- [x]  Mixed Prefill / Decode
    
- [x]  Heterogeneous Requests
    
- [x]  Interactive Serving
    
- [ ]  Batch / Offline Serving
    

### Workload Characteristics

- [ ] Number of tenants: 2-8
    
- [ ] Arrival pattern: idle and active bursts
    
- [ ] Prompt length:  2-1021
    
- [ ] Output length: 2-977
    
- [ ] Prefix overlap: None/Minimal
    
- [ ] Burstiness: 120-180 req/min
    
- [ ] SLO assumptions:
    

---

# 4. Core Contribution
_Notations_
![[Pasted image 20260904113836.png]]

## Core Insight

> **VTC Scheduler** To achieve true fairness and performance isolation in multi-tenant LLM serving, scheduling must operate at **token-level rather than request-level granularity** to handle extreme workload heterogeneity (such as short chats vs. long document analyses). Because LLM generation is auto-regressive and output lengths are unknown in advance, the scheduler must **track services dynamically on-the-fly** and employ a **counter-lift mechanism** to prevent idle clients from hoarding service credits and starving active users when they rejoin the queue.

## Main Contributions
- **First Formulation of LLM Serving Fairness**: Defined a formal max-min fairness framework specifically tailored for LLM serving, featuring a customizable cost function that differentiates between the asymmetric costs of input prefilling (parallelized) and output decoding (highly sequential).
- **The Virtual Token Counter (VTC) Algorithm**: Proposed a practical, work-conserving scheduling algorithm that integrates seamlessly with continuous batching and PagedAttention to prioritize clients based on their actual resource consumption.
- **Rigorous Theoretical Bounds**: Mathematically proved a tight $2\times$ upper bound on the service difference between two backlogged clients, while guaranteeing strict performance isolation and bounded dispatch latency for well-behaved tenants.
- **Comprehensive Real-World Evaluation**: Demonstrated that VTC outperforms traditional First-Come-First-Serve (FCFS) and Request-Per-Minute (RPM) methods under both synthetic and real-world traces (re-scaled from the ==LMSYS Chatbot== Arena logs), preserving high GPU throughput and isolation.

Server Capacity: ![[Pasted image 20260904105636.png]]

Fairness Guarantee: ![[Pasted image 20260904110000.png]]
## What Is Actually Novel?
- **Iteration-Level Token Scheduling (No Pre-Dispatch Requirements)**: Traditional fair queueing algorithms (e.g., WFQ, SFQ) calculate virtual start/finish tags using packet lengths before scheduling. VTC schedules requests step-by-step and updates counters on-the-fly as each token is decoded, cleanly bypassing the problem of **unknown output lengths**.
- **Asymmetric Resource Costing (**$w_p$ **vs.** $w_q$**)**: While traditional operating systems or network schedulers assume uniform resource costs for data chunks/cycles, VTC introduces weighted tracking to accurately capture the stark difference in GPU cost between prefill and decode phases5more_horiz.
	- ![[Pasted image 20260904113736.png]]
- **Credit Saturation Protection via Counter-Lift**: Simple Least-Counter-First (LCF) policies suffer from credit accumulation: a client that has been idle accumulates a massive deficit and can monopolize the GPU upon rejoining. VTC solves this by dynamically lifting the rejoined tenant's counter to match the current active pool.
- **Decoupling Fairness from Fluctuating Capacity**: Unlike network links or CPUs, an LLM server’s processing capacity (tokens/sec) constantly changes based on dynamic batch composition and memory limits. VTC relies strictly on comparative client counters, meaning scheduling logic remains robust regardless of transient server throughput shifts.

---

# 5. Algorithm Classification

## Algorithm Tags

- [x]  Fairness
    
- [ ]  Locality
    
- [x]  Prediction
    
- [ ]  Load Balancing
    
- [ ]  SLO-aware
    
- [x]  Throughput Optimization
    
- [ ]  Latency Optimization
    
- [ ]  KV Cache Aware
    
- [x]  Resource Allocation
    
- [ ]  Admission Control
    
- [ ]  Migration
    
- [x]  Batching
    
- [x]  Scheduling
    
- [ ]  Other:
    

## Combination

**Primary combination:**

`Fairness / Scheduling / Prediction`

---

# 6. Input Signals

## What Does the Algorithm Observe?
request trace
input token weight wp, 
output token weight wq, 
upper bound from ![[Pasted image 20260904105430.png]] denoted as U.



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
    
- [x]  Which request executes next
    
- [x]  Which requests form a batch
    
- [ ]  Whether to admit a request
    
- [ ]  Whether to migrate a request
    
- [ ]  Resource allocation
    
- [ ]  Other:
    

## Decision Granularity

- Per request: request from a backloged clients are priroitized
    
- Per token: token level fairness is implemented
    
- Per batch: NA
    
- Periodic: NA
    
- Event driven: NA
    

---

# 8. Algorithm

## High-Level Workflow

![[Pasted image 20260904102805.png]]

## Algorithm Steps

![[Pasted image 20260904000310.png]]

- Can improve the step where the loop is broken when the high priority task doesn't fit into the gpu memory

## Few important variations:
1. Wighted VTC
![[Pasted image 20260904114917.png]]
Replate Line 22 with this:
![[Pasted image 20260904114128.png]]
2. VTC with Length Prediction
		when a request r is selected, the cost associated with the predicted number of output tokens is immediately added to the virtual counter of the client sending the request. During the actual decoding process, adjustments are made to the virtual counter based on the actual number of output tokens produced.
	![[Pasted image 20260904114535.png]]
## Time Complexity

- Decision complexity: O(1)
    
- State update complexity: O(1)
    
- Memory complexity: O(1)
    

---
%%
# 9. State Maintained

| State | Scope      | Update Rule                                         | Purpose         |
| ----- | ---------- | --------------------------------------------------- | --------------- |
|       | Per-tenant | Serve the client with least number of tokens served | Ensure Fairness |

Examples to investigate:

- Queues
    
- Virtual time
    
- Deficit counters
    
- Token counters
    
- Prefix trees
    
- KV-cache metadata
    
- Prediction history
    
- Load estimates
    
%%
---

# 10. Objective

## What Is Being Optimized?

- [ ]  Throughput
    
- [ ]  TTFT
    
- [ ]  TPOT
    
- [x]  End-to-End Latency
    
- [ ]  Tail Latency
    
- [x]  Fairness
    
- [ ]  Cache Hit Rate
    
- [x]  GPU Utilization
    
- [ ]  SLO Attainment
    
- [ ]  Cost
    
- [ ]  Other:
    

## Formal Objective

```text
Write the paper's objective function here.
```

## Optimization Tradeoff

> **Work-Conservation (GPU Throughput)** and **Performance Locality (Cache Hits)** get worse as you improve (tighten) the fairness metric.
>
>	- **Work-Conservation / Throughput**: To guarantee a tighter fairness bound, the scheduler must restrict the batch memory allocation allowed for any single client. This restriction prevents the system from fully packing continuous batches, causing the GPU to sit underutilized and lowering overall token throughput.
>	- **Cache Locality**: Prioritizing clients strictly by their virtual counters conflicts directly with cache-aware scheduling (e.g., prefix sharing), which requires scheduling requests out of fair order to maximize prompt prefix reuse in the KV cache.

---

# 11. Assumptions and Constraints

## Assumptions

- [x]  Homogeneous GPUs
    
- [ ]  Heterogeneous GPUs
    
- [x]  Centralized Controller
    
- [ ]  Distributed Controller
    
- [ ]  Known Prompt Length
    
- [ ]  Known Output Length
    
- [ ]  Prefix Information Available
    
- [ ]  Accurate Prediction Available
    
- [ ]  Stable Workload
    
- [ ]  Static Cluster
    
- [ ]  Other:
    

%%## Hidden Assumptions

> What assumptions are necessary for the algorithm to work but are not heavily emphasized?

## Constraints%%

---

# 12. Experimental Methodology

## Experimental Workloads

| Dataset / Trace         | Characteristics                                                                                                                                                                                         | Why Used?                                                                                                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Synthetic Workload**  | • Deterministic or stochastic arrivals.<br>• Fixed (256/256) or highly asymmetric prompt and output lengths (e.g., 64 vs. 512 tokens).<br>• Evaluated with 2 to 8 clients.                              | To cleanly isolate and visualize specific scheduler behaviours under controlled environments (such as constant overloading, ON/OFF traffic, and extreme length heterogeneity). |
| **Real Workload Trace** | • Constructed from real-world LMSYS Chatbot Arena logs.<br>• Consists of **27 clients** (each representing a unique LLM served on the platform).<br>• Highly dynamic, skewed request rate distribution. | To rigorously validate the performance, fairness, and isolation guarantees of VTC under highly dynamic, unpredictable, and complex real-world workloads.                       |
## Workload Generation

- **Arrival process**:
    - _Synthetic_: Deterministic uniform arrivals (evenly spaced) or stochastic arrivals generated via a **Poisson process** (coefficient of variance = 1).
    - _Real_: Timestamp entries from the LMSYS Chatbot Arena logs are **re-scaled** to map a compressed 10-minute duration with a target aggregate arrival rate of **210 requests per minute**.
- **Tenant distribution**:
    - _Synthetic_: 2 to 8 concurrent clients sharing the server.
    - _Real_: 27 concurrent clients with a highly skewed distribution (a few highly popular "models" send the vast majority of requests, while others send very little).
- **Prompt distribution**:
    - _Synthetic_: Fixed at **256 tokens** or systematically varied (64 or 512 tokens).
    - _Real_: Highly variable input lengths ranging from **2 to 1,021 tokens** (with an average input length of **136 tokens**).
- **Output distribution**:
    - _Synthetic_: Fixed at **256 tokens** or systematically varied (64 or 512 tokens).
    - _Real_: Highly variable output lengths ranging from **2 to 977 tokens** (with an average output length of **256 tokens**).
- **Prefix overlap**:
    - **None / Orthogonal**. SGLang-style prefix-sharing optimization is treated as an orthogonal scheduler priority that is in direct conflict with fair queueing, and is thus omitted from the baseline tests.
- **Burst injection**:
    - Handled via a dynamic **"ON/OFF" request pattern**. Selected clients transition between active phases (ON) where they submit requests over their capacity share, and silent phases (OFF) where they submit zero traffic.
- **Synthetic modifications**:
    - Real-world timestamps from the Chatbot Arena database are re-scaled so that a 10-minute slice of traffic properly saturates the target NVIDIA A10G serving instance.

---

# 13. Simulation Setup

None

---

# 14. Real Hardware Setup

## Hardware

| Component      | Configuration                                                                       |
| -------------- | ----------------------------------------------------------------------------------- |
| GPU            | NVIDIA A10G _(for synthetic/real workloads)_ <br>NVIDIA A100 (for ablation studies) |
| GPU Memory     | 24 GB (for A10G)<br>80 GB (for A100)                                                |
| Number of GPUs | 1                                                                                   |
| CPU            | -                                                                                   |
| RAM            | -                                                                                   |
| Storage        | -                                                                                   |
| Network        | -                                                                                   |

## Serving Framework

- Framework:
    S-LoRA (built on top of LightLLM)
- Version:
    
- Model:
    Llama-2-7B _(primary tests)_ , Llama-2-13B _(ablation studies)_
- Precision:
    -
- Tensor Parallelism:
    -
- Number of replicas:
    1
- Scheduler configuration:
    Virtual Token Counter (VTC) integrated into S-LoRA's scheduler loop
- KV cache configuration:
    Memory pool size of **10,000 tokens** _(A10G)_  or **35,000 / 65,000 tokens** _(A100)_, running PagedAttention with a **block size of 1**

---

# 15. Metrics

| Metric                               | Definition                                                                                                                                                                                      | Why Important?                                                                                                               |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Throughput                           | The **total number of tokens** (including both input and output tokens) processed divided by the total execution time (tokens/sec)                                                              | Measures system efficiency and verifies that a scheduler can achieve fairness without sacrificing overall hardware capacity. |
| Response Time or First Token Latency | **"Response Time"** or **"First Token Latency"**, which is the average time a client's requests wait in queue before their first token is processed (measured over a 30-second sliding window). | Directly dictates the user-perceived responsiveness of interactive applications like chatbots.                               |
| **Service Difference**               | **service difference** is the quantitative metric used to assess how much a scheduling algorithm deviates from "ideal fairness"                                                                 | **Service Difference** quantifies the deviation from ideal fairness to prove how closely the scheduler isolates clients.     |
| Work-Conservation                    | meaning the scheduler never leaves the GPU idle if there are requests waiting in the queue.                                                                                                     | Ensures expensive, scarce GPU resources are fully utilized (a key weakness of static RPM rate limits)                        |
| SLO Attainment                       |                                                                                                                                                                                                 |                                                                                                                              |

## Metric Formulas

```text
Add important formulas here.
```

---

# 16. Baselines

| Baseline                                            | Why Compared?                                                                                                                                                         | Strong / Weak?                                                                                                                                                                                    |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **First-Come-First-Serve (FCFS)**                   | It is the default scheduling policy in modern systems (including vLLM and Hugging Face TGI).                                                                          | **Weak**: Completely lacks client isolation; a single client sending massive bursts can starve other users and trigger system-wide latency spikes or timeouts.                                    |
| **Request-Per-Minute (RPM)**                        | It represents the standard industry practice for API management and access control to provide client-level isolation.                                                 | **Weak**: Non-work-conserving; it statically throttles active users even when the server has spare capacity, leading to severe throughput drops and underutilized GPUs.                           |
| **Least Counter First (LCF)**                       | A direct algorithmic variant of VTC evaluated without the counter-lift mechanism. It acts as a baseline to prove the necessity of VTC's credit saturation protection. | **Weak/Flawed**: Fails under dynamic "ON/OFF" traffic; rejoining clients inherit old deficits, allowing them to temporarily monopolise the queue and starve other users.                          |
| **VTC with Length Prediction** _(oracle & predict)_ | Design variants of VTC introduced to evaluate how resolving the "unknown request output length" issue impacts service discrepancy.                                    | **Strong**: VTC (oracle) defines the theoretical upper bound of prediction-driven token scheduling, while VTC (predict) is a highly practical variant that uses rolling request history averages. |
## Missing Baselines?

> - **Cache-Aware Schedulers (e.g., SGLang / RadixAttention)**: SGLang prioritizes scheduling requests based on prefix cache sharing to optimize GPU throughput. VTC notes that cache locality and fair sharing are orthogonal and in conflict, but the paper lacks a quantitative comparison demonstrating how much throughput must be sacrificed to achieve VTC’s fairness guarantees.
>- **Preemptive Schedulers (e.g., FastServe)**: FastServe utilizes preemption to minimize Job Completion Time (JCT) in LLM serving. Since VTC assumes a non-preemptive continuous batching scheme, comparing against a preemptive scheduling baseline would have highlighted the direct latency trade-offs between fairness-oriented queues and completion-optimized queues.



---

# 17. Key Results

## Main Quantitative Results

![[Pasted image 20260904100337.png]]


![[Pasted image 20260904122943.png]]

![[Pasted image 20260904123140.png]]

![[Pasted image 20260904124850.png]]
---

# 18. Failure Cases and Limitations

## Where Does It Fail?
- **High Discrepancy under No-Preemption**: In the default no-preemption setup, if a client has many requests active in the batch, it can generate a massive stream of output tokens that cannot be interrupted1. This causes temporary starvation for other clients and pushes the service difference to the theoretical maximum bound of $2\max(w_p \cdot L_{\text{input}}, w_q \cdot M)$.
- **Conflict with Prefix-Sharing Cache (e.g., SGLang)**: SGLang-style schedulers prioritize requests that share KV cache prefixes to maximize throughput. VTC’s strict counter-based ordering directly conflicts with this, meaning you must sacrifice either cache locality or fairness.

## Limitations Stated by Authors
- **Unpredictable Output Lengths**: Standard VTC cannot prevent over-compensating clients at selection time because it doesn't know output lengths in advance. While VTC with length prediction helps, its effectiveness relies entirely on workload predictability.
- **Memory vs. Fairness Trade-off**: Tightening the fairness bound requires restricting the running batch memory allocated to individual clients, which directly compromises work-conservation and lowers overall GPU throughput.
- **Distributed System Constraints**: The paper evaluates a single-GPU server. Scaling to multi-replica setups requires a central request dispatcher and introduces counter synchronization challenges.

## Limitations I Identified
- **Vulnerability to Cold-Start Bursting**: Although the counter-lift mechanism lifts newly joined tenants to the minimum active counter, a client that has been silent can still burst up to its limit instantly, causing short-term latency degradation for active users.
- **SLA/Deadline Insensitivity**: VTC is a max-min fair share scheduler, not an SLO-aware scheduler. It cannot prioritize requests with strict latency deadlines (TTFT/TPOT) if the client has already exhausted its fair token share.

## Scalability Concerns

Consider:

- CPU overhead: **Very Low.** VTC only adds about 100 lines of code to the serving engine. The scheduling decision simply performs an `argmin` operation over active clients to select the next request.
    
- Memory overhead: The scheduler only needs to store a single floating-point virtual counter per client, requiring virtually no extra GPU/CPU memory.
    
- Network overhead: **Low on single-node, potentially high on distributed clusters.** In a multi-replica setup, updating and synchronizing counters concurrently across serving engines will require continuous network message exchanges.
    
- Metadata overhead: Consists solely of a lightweight map tracking client IDs and their active virtual counter values.
    
- Centralized bottleneck: In distributed deployments, the central request dispatcher responsible for tracking counters and scheduling requests becomes a potential single point of failure and a throughput bottleneck.
    
- Number of tenants: Scales well. However, if serving tens of thousands of active LoRA adapters, linear search (`argmin`) over the active queues can become a CPU bottleneck unless implemented using a min-priority queue (min-heap) structure.
    
- Number of replicas: High synchronization complexity. As replica count increases, the delay in synchronizing virtual counters across serving instances can lead to outdated counter states and fairness violations.
    

---

# 19. Challenges and Engineering Problems

|Challenge|Cause|Solution Used|
|---|---|---|
||||

## Failure / Edge Cases

---

# 20. Comparison With FairRoute

## Which Signal Does This Paper Optimize?

| Fairness | Locality | Prediction |
| -------- | -------- | ---------- |
| Yes      | No       | No         |

## What Can FairRoute Reuse?
VTC scheduling algorithim

## What Does FairRoute Need Beyond This?
Implement the algorithm for multi-replica systems with cache awareness and workload prediction

## Threat to FairRoute

> Never


---

# 21. Baseline Decision

## Should We Implement This?

-  Yes, directly

### Reason
This paper has the proved fairness guarantee

## Comparison Priority

Medium

---
%%
# 22. Implementation Difficulty

## Difficulty

Easy 

## Why?

## Dependencies

## Estimated Components Needed

```text
Component 1
Component 2
Component 3
```
%%
---

%%# 23. Evidence

## Important Sections

|Claim / Observation|Page|Section / Figure / Table|
|---|---|---|
||||

## Important Quotes

> Keep only short quotes necessary for later verification.%%

---

%%# 24. Final Research Notes

## What I Learned

## Open Questions

## Research Opportunities

## Connection to Other Papers

- [[Paper Name]]
    
- [[Paper Name]]
    %%

---

# 25. Paper Verdict

## One-Sentence Verdict

## Importance to FairRoute

⭐⭐⭐⭐

## Deep Dive Required?

- [ ]  Yes
    
- [x]  No
    

## Read Again Before Implementation?

- [ ]  Yes
    
- [x]  No