---
title: "PiLLM: Resource-efficient LLM Inference Using Workload Prediction"
authors:
  - Yunqian Fan, Shihao Bai, Ruihao Gong, Zaijun Wang, Rui Fan  
year: "2026"
venue: EuroSys '26
paper_type: research
status: reading
rating:
url: ""
pdf: "[[2026 - PiLLM.pdf]]"
tags:
  - predictive
---

# 2026 - PiLLM

> [!abstract] One-Line Summary  
> PiLLM introduces a resource-efficient LLM serving system that leverages lightweight statistical workload predictions via the Central Limit Theorem to dynamically manage inter-GPU instance scaling and optimize intra-GPU batch-level memory allocation, reducing GPU requirements by up to 3.1x with negligible request evictions under strict SLO constraints.

---

## 1. Paper Metadata

| Field                   | Notes                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| **Authors**             | Yunqian Fan, Shihao Bai, Ruihao Gong, Zaijun Wang, Rui Fan         |
| **Year**                | 2026                                                               |
| **Venue**               | EuroSys '26                                                        |
| **Paper Type**          | System / Algorithm / Scheduling                                    |
| **Code Available?**     | No                                                                 |
| **Artifact Available?** | No                                                                 |
| **Related System**      | LightLLM (Extended), vLLM, SGLang, PastFuture, Mooncake, DistServe |

---

# 2. Problem

## Problem Statement

The paper solves the chronic **underutilization of GPU resources** in online Large Language Model (LLM) inference clusters. Online inference is highly unpredictable because individual request input/output sequence lengths vary by orders of magnitude, causing high variability in both compute demand and Key-Value (KV) cache memory. To meet strict Service Level Objectives (SLOs), traditional cloud systems are forced to statically overprovision by allocating excess GPU instances for peak traffic and reserving conservative, worst-case memory per request.

## Why Does This Problem Matter?
Online LLM serving demands substantial, highly expensive GPU infrastructure. Overprovisioning leads to massive economic inefficiencies and resource waste. As modern generative workflows evolve towards compound AI applications, multi-turn dialogues, reasoning agents, and long-context processing, the computational demand fluctuates wildly and unpredictably. Bridging the gap between peak capacity requirements and average resource efficiency directly lowers the cost and environmental footprint of serving advanced LLMs.

## Why Do Existing Approaches Fail?

| Existing Approach                                                  | Limitation                                                                                                                                                                                                             |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Metrics-based scaling (e.g., KServe)**                           | Relies on generic hardware utilization metrics that fail to detect workload complexity changes, leading to massive lag in auto-scaling response during sudden traffic spikes, violating SLOs.                          |
| **Aggressive per-request scheduling (e.g., vLLM)**                 | Maximizes immediate memory utilization via PagedAttention but suffers from high request eviction (preemption) rates (up to 68% under high-variability chat workloads) due to a complete lack of length predictability. |
| **Conservative per-request scheduling (e.g., SGLang, PastFuture)** | Limits or eliminates evictions by allocating memory based on max-sequence length predictions or expensive sampling, but severely underutilizes physical GPU resources (reserving empty memory that remains unused).    |
| **Neural length-prediction models**                                | Running specialized Transformer-based predictors for individual requests introduces significant computational overhead, which requires extra GPU resources and degrades real-time execution speeds.                    |



---

# 3. System Context

## System Layer

- [ ]  Cluster Routing
    
- [x]  Request Scheduling
    
- [x]  Continuous Batching
    
- [x]  Prefill Scheduling
    
- [x]  Decode Scheduling
    
- [x]  KV Cache Management
    
- [x]  GPU Scheduling
    
- [x]  Distributed Inference
    
- [x]  Simulation / Evaluation Infrastructure
    
- [ ]  Other:
    

## Target Workload

- [ ]  Multi-tenant
    
- [ ]  Prefix Sharing
    
- [x]  Bursty Traffic
    
- [x]  SLO-aware
    
- [x]  Long-context
    
- [x]  Mixed Prefill / Decode
    
- [x]  Heterogeneous Requests
    
- [x]  Interactive Serving
    
- [ ]  Batch / Offline Serving
    

### Workload Characteristics

- [x] Arrival pattern: Dynamic online request arrival pattern with extreme, non-linear load fluctuations and sudden spikes.

- [x] Prompt length: Extremely heterogeneous, ranging from very short queries (average 32 tokens in assistant tasks) to ultra-long contexts (up to 120,000+ tokens in document analysis).

- [x] Output length: Highly variable, ranging from ~30 tokens to over 25,000 tokens.

- [ ] Prefix overlap: None

- [x] Burstiness: Highly bursty; computational demand spikes (e.g. sum of squares of tokens) have little or no linear correlation with request frequency.

- [x] SLO assumptions: Tailored, request-specific SLO deadlines set at 20% higher than baseline execution time under maximum resource allocation.

---

# 4. Core Contribution

## Core Insight

> **While the sequence length and resource demand of any single LLM request are highly variable and unpredictable, the aggregated resource and memory requirements of a *batch* of requests are highly predictable.** By leveraging the Law of Large Numbers and the Central Limit Theorem (CLT), we can model and manage LLM workloads at batch granularity to optimize both cluster-level GPU scaling and device-level memory allocation.

![[Pasted image 20260905135514.png]]
## Main Contributions
1. **Lightweight Statistical Length Predi:** A lightweight prediction algorithm that tracks the mean and variance of request input/output lengths using a sliding window. Using the Central Limit Theorem, it estimates the batch's total requirements with mathematically sound, tunable confidence intervals ($\epsilon$), eliminating expensive neural network forecasting models.

2. **Elastic Inter-GPU Resource Manager & Dispatche:** A dual-window cluster scheduling system. It periodically scales the number of active prefill ($n_p$) and decode ($n_d$) instances based on FLOPs-to-instance conversions, and executes an SLO-aware routing algorithm with an on-demand "Spike Reaction" safety net to instantly boot instances during traffic surges.

3. **Batch-Level Intra-GPU Memory Scheduler:** A memory manager that replaces conservative, per-request worst-case memory reservations with batch-granular collective memory sharing. By extending LightLLM's linked-list token-slot design, requests within a batch share a unified KV cache pool to dynamically offset each other's deviations.

4. **Fine-Grained Prefill-Decode Disaggregation:** Implementation of these prediction-aware controllers on top of a disaggregated execution architecture, allowing independent optimization of compute-bound prefill and memory-bound decode hardware pools to maximize total GPU savings.

## What Is Actually Novel?
- **CLT-Based Batch Resource Predictor:** The first system to exploit the statistical properties of continuous batching to shrink error bounds on memory and compute estimates, turning an intractable per-request guessing game into a stable, semi-deterministic scheduling problem.

- **Workload-Driven Elastic Auto-Scaling:** Shifting from reactive cloud scaling based on generic hardware utilization metrics (which suffer from lag and cannot distinguish prompt complexity to proactive scaling driven by expected FLOPs derived from statistical length distributions.

- **Statistical Overcommitment of KV Cache:** Moving away from the classic "efficiency-reliability dilemma" (vLLM's high utilization with massive evictions vs. SGLang's zero evictions with low utilization) by statistically overcommitting physical memory under controlled, negligible eviction bounds (<0.5%).

---

# 5. Algorithm Classification

## Algorithm Tags

- [ ]  Fairness
    
- [ ]  Locality
    
- [x]  Prediction
    
- [x]  Load Balancing
    
- [x]  SLO-aware
    
- [x]  Throughput Optimization
    
- [x]  Latency Optimization
    
- [ ]  KV Cache Aware
    
- [x]  Resource Allocation
    
- [ ]  Admission Control
    
- [ ]  Migration
    
- [x]  Batching
    
- [x]  Scheduling
    
- [ ]  Other:
    

## Combination

**Primary combination:**

`Prediction / SLO-aware / Resource Allocation / Batching / Scheduling`

---

# 6. Input Signals

## What Does the Algorithm Observe?

| Signal                                                               | Source                         | Why Is It Needed?                                                                                                                             |
| -------------------------------------------------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Input Sequence Length ($l_{p,i}$)**                                | API Layer / Request Metadata   | Used to compute the quadratic computational workload (FLOPs) of the prefill phase and the initial prefill memory requirement.                 |
| **Current Output Length ($l_{d,i}$)**                                | Decode Instances               | Monitors progress of active requests and updates the current KV cache occupancy.                                                              |
| **Max Output Length ($l_{m,i}$)**                                    | Request Configuration          | Bounds maximum generation limits and serves as a reference for baseline comparison.                                                           |
| **Historical Sequence Lengths**                                      | Sliding Window Log             | Used to dynamically compute the rolling mean ($\mu_d, \mu_p$) and standard deviation ($\sigma_d, \sigma_p$) of sequence length distributions. |
| **Request Queue ($\mathcal{Q}$)**                                    | API Gateway / Global Scheduler | Monitors arrival rates, determines current traffic load, and flags SLO-jeopardized requests to trigger Spike Reaction.                        |
| **Instance Pool State ($\mathcal{I}_{idle}, \mathcal{I}_{active}$)** | Data Plane Monitor             | Identifies available GPU resources and calculates expected queue-wait times based on active execution progress.                               |
| **Offline Calibration Coefficients**                                 | Pre-computed Profiling         | Hardware-specific constants ($\alpha, \beta, \gamma, \lambda, \mu$) mapped to calibrate FLOPs and model execution timings.                    |

## Signal Collection Overhead

- [x] Centralized or distributed?

**Distributed collection with centralized decision-making.** Input lengths are gathered at the front-end API layer, token outputs are logged locally at decode GPU instances, and metrics are sent to the centralized Inter-GPU manager.

- [x] Push or pull telemetry?

**Push telemetry.** Decode instances periodically push compressed execution statistics to the global controller rather than transmitting every single token event, preventing networking overhead.

- [x] Update frequency:

**Hybrid.** Input signal collection is event-driven (instantly upon request arrival at the API layer), whereas decode output statistics are sampled and updated at configurable periodic time intervals.

- [x] Metadata size:

**Negligible.** Telemetry consists entirely of raw scalar numbers (sequence lengths, token counts, instance IDs, and queue statuses) requiring virtually zero network bandwidth.

- [x] Potential bottleneck:

**Centralized Controller Scaling.** The centralized control plane manages dispatching, scaling, and scheduling globally. The authors note that scaling to massive clusters (thousands of GPUs) may introduce communication and latency bottlenecks at this central coordinator [

---

# 7. Decision

## Decision Variable

**What exactly does the system decide?**

- [x] Which replica receives a request

*(The global dispatcher assigns incoming requests to specific prefill and decode instances based on expected completion times and SLO deadlines).*

- [ ] Which request executes next

*(Not primary; continuous batching schedules requests together, although requests in the global queue are processed in sorted priority based on SLO deadlines).*

- [x] Which requests form a batch

*(The intra-GPU scheduler bundles new requests into the continuous execution batch, and Spike Reaction builds optimal, utilization-compliant batch sizes ($\mathcal{B}_{spike}$) to run on newly activated GPUs).*

- [ ] Whether to admit a request

*(All incoming requests are admitted, but if resources are saturated and scaling is economically inefficient, they are queued and temporarily forfeit SLO guarantees).*

- [ ] Whether to migrate a request

*(The disaggregated architecture streams KV cache states progressively from prefill instances to decode instances, but this is a fixed pipeline stage rather than dynamic execution migration).*

- [x] Resource allocation

*(The Inter-GPU manager dynamically decides how many physical GPU instances ($n_p, n_d$) to allocate to the prefill and decode instance pools, and Spike Reaction decides whether to boot additional instances during surges).*

- [ ] Other:
    

## Decision Granularity

- **Per request:**

**Dispatching and Routing.** The global dispatcher routes requests individually to active instances or queues them upon arrival at the API layer.

- **Per token:**

**Memory Allocation.** The intra-GPU execution layer dynamically updates token-level linked-list slots during autoregressive token generation.

- **Per batch:**

**Memory Capacity Updates.** The intra-GPU scheduler updates its predicted batch-level memory allocation ($c(\mathcal{B})$) every time a request is enqueued or completed (popped).

- **Periodic:**

**Inter-GPU Pool Scaling.** Instance allocations ($n_p, n_d$) are re-evaluated and dynamically scaled at the boundary of a larger, periodic GPU management time window.

- **Event driven:**

**Spike Reaction Scaling.** The activation of additional physical GPU instances is triggered immediately when a queue surge threatens to violate request SLO deadlines.

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

Average output length  for a batch
![[Pasted image 20260905222842.png]]

FLOPs and Memory prediction:
![[Pasted image 20260905222819.png]]

Memory required for a batch (Intra gpu)
![[Pasted image 20260905222910.png]]

No of nodes (Inter gpu)
![[Pasted image 20260905223204.png]]
## Algorithm Steps

![[Pasted image 20260905135607.png]]

![[Pasted image 20260905135637.png]]
## Time Complexity

- Decision complexity:
    
- State update complexity:
    
- Memory complexity:
    

---

# 9. State Maintained

| State                                                            | Scope                        | Update Rule                                                                                                                                                                       | Purpose                                                                                                                                       |
| ---------------------------------------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Prediction History**                                           | Global / Scenario-specific   | Updated dynamically via a sliding window containing a rolling history of hundreds of completed requests.                                                                          | Calculates rolling mean ($\mu_d, \mu_p$) and standard deviation ($\sigma_d, \sigma_p$) of sequence lengths to feed the CLT prediction models. |
| **Batch Memory Requirements ($c(\mathcal{B})$)**                 | Per-replica (Model Instance) | Updated when requests are enqueued ($1_i = 1$) or completed ($1_i = 0$), adding/subtracting predicted remaining tokens ($c_{kv} \cdot \hat{l}_{d, \mathcal{B}_{\mathcal{W}_i}}$). | Tracks running collective memory commitments to allow safe batch-level memory sharing and prevent out-of-memory evictions.                    |
| **Instance States ($\mathcal{I}_{idle}, \mathcal{I}_{active}$)** | Global                       | Updated dynamically in real-time as instances finish execution, switch roles, or are activated/suspended.                                                                         | Enables the global dispatcher to monitor cluster resources and compute expected execution start times.                                        |
| **Request Queues ($\mathcal{Q}$, $\mathcal{Q}_{spike}$)**        | Global                       | Appends incoming requests upon arrival; transfers requests to $\mathcal{Q}_{spike}$ if deadlines are violated.                                                                    | Holds pending requests, determines current queue wait times, and triggers on-demand Spike Reaction scaling.                                   |
| **Token Counters ($l_{p,i}, l_{d,i}$)**                          | Per-request                  | Incrementally updated step-by-step during token generation on active decode instances.                                                                                            | Tracks the current length of each active request to monitor generation progress and provide telemetry signals.                                |
| **Calibration Coefficients**                                     | Global                       | Static, pre-computed offline during hardware characterization.                                                                                                                    | Constants ($\alpha, \beta, \gamma, \lambda, \mu$) used to convert predicted sequence lengths into execution FLOPs and compute times.          |



---

# 10. Objective

## What Is Being Optimized?

- [x]  Throughput
    
- [ ]  TTFT
    
- [ ]  TPOT
    
- [ ]  End-to-End Latency
    
- [x]  Tail Latency
    
- [ ]  Fairness
    
- [ ]  Cache Hit Rate
    
- [x]  GPU Utilization
    
- [x]  SLO Attainment
    
- [x]  Cost
    
- [ ]  Other:
    

## Formal Objective

1. **Inter-GPU Resource Optimization (Instance-Level Minimization):**

$$\text{Minimize } \left( n_p + n_d \right)$$

$$\text{subject to: } \mathbb{P}\left(\text{latency}_i \le d_i \right) \ge 1 - \delta_i \quad \forall r_i \in \mathcal{Q}$$

Where $n_p$ and $n_d$ represent the number of active prefill and decode instances, $d_i$ represents the request-specific SLO deadline, and $\delta_i$ is a negligible violation tolerance.

  

2. **Intra-GPU Memory Scheduling (Capacity Optimization):**

$$\text{Maximize } \mathcal{B}_{concurrency} \implies \text{Maximize } \sum_{r_i \in \mathcal{B}} \left( l_{p,i} + l_{d,i} \right)$$

$$\text{subject to: } \sum_{r_i \in \mathcal{B}} c_{kv} \cdot \left( l_{p,i} + l_{d,i} + \hat{l}_{d, \mathcal{B}_{\mathcal{W}_i}} \right) \le M_{GPU}$$

$$\text{and } \text{Eviction Rate} \le \epsilon \quad (\text{where } \epsilon < 0.5\% \text{ in practice [7, 41]})$$

## Optimization Tradeoff

> **Throughput-Latency Tradeoff:** By increasing batch concurrency and maximizing physical GPU utilization, PiLLM achieves much higher throughput and requires fewer physical instances, but this slightly shifts execution times to the right (longer per-request processing times).

> **Utilization-Eviction Tradeoff:** Overcommitting physical memory via batch-level statistical predictions allows the system to operate at near-maximum memory utilization (up to 96%), but introduces a controlled, non-zero risk of request evictions (preemption) when actual sequence lengths exceed the statistical bound.

---

# 11. Assumptions and Constraints

## Assumptions

- [x] Homogeneous GPUs *(Within each phase pool, prefill capacity $C_p$ and decode capacity $C_d$ are uniform constants )*

- [ ] Heterogeneous GPUs *(Partially supported across pools but not within pools)*

- [x] Centralized Controller *(A global scheduling layer manages routing, dispatching, and scaling )*

- [ ] Distributed Controller

- [x] Known Prompt Length *(Input lengths $l_{p,i}$ are captured immediately at the front-end API layer )*

- [ ] Known Output Length *(Output length is unknown and statistically predicted )*

- [ ] Prefix Information Available

- [x] Accurate Prediction Available *(Assumes statistical mean and variance of request lengths are stable and learnable via a sliding window )*

- [ ] Stable Workload *(Workload is bursty and highly variable, though sequence length distribution within windows is stable )*

- [ ] Static Cluster *(Cluster is elastic and dynamically scales )*

- [ ] Other:

## Hidden Assumptions

> **Short-Term Stationarity of Length Distributions:** The CLT-based statistical predictor assumes that the request input/output sequence length distributions remain stationary within the sliding window interval . If there is an instantaneous, extreme shift in the user query behavior (e.g., transitioning from code generation to creative writing), the rolling statistics will lag behind the transition, causing temporary prediction errors.

> **Deterministic Model Execution with Zero Jitter:** The inter-GPU FLOPs-to-instance scaling relies on the assumption that execution time can be deterministically fitted via offline polynomial regressions (Equation 2) . This requires compiled model operators (AOT, CUDA Graph, chunk-based layouts) to eliminate system-level jitters (like JIT compilation lag or dynamic memory allocation overhead) .

> **High-Speed Network for Disaggregated State Transfer:** Prefill-decode disaggregation assumes that streaming intermediate KV cache states progressively from prefill GPUs to decode GPUs adds negligible latency overhead . This necessitates extremely high-bandwidth cluster interconnects (like NVLink or Infiniband) .

## Constraints

- **Individualized SLO Deadlines ($d_i$):** Set specifically for each request at 20% higher than its baseline execution time under maximum resources to maintain fairness .

- **GPU KV Cache Memory Capacity Limit ($M_{GPU}$):** Physical GPU memory bounds the batch-level memory allocation ($c(\mathcal{B}) \le M_{GPU}$) .

- **Maximum Cluster Size:** Elastic scaling is constrained between 1 and 4 instances per phase in evaluations .

- **Eviction Rate Tolerances ($\epsilon$):** Upper-bounded by operators (e.g., targeting $<0.5\%$ in practice) .

---

# 12. Experimental Methodology

## Experimental Workloads

| Dataset / Trace  | Characteristics                                                                   | Why Used?                                                                                               |
| ---------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **BurstGPT**     | Public ChatGPT trace. Mean input: 11,099 tokens, Mean output: 3,206 tokens.       | Represents multi-tenant production chatbot patterns with moderate length but high temporal variability. |
| **MoonCake**     | Kimi AI trace. Mean input: 125,546 tokens, Mean output: 2,000 tokens.             | Evaluates extremely long context sizes and highly heterogeneous length distributions.                   |
| **Assistant**    | Task-oriented interactions. Mean input: 2,398 tokens, Mean output: 1,024 tokens.  | Represents short-to-moderate, structured dialogue workloads.                                            |
| **Conversation** | Multi-turn chat histories. Mean input: 16,670 tokens, Mean output: 25,589 tokens. | Tests extreme, long-tail output length distributions under interactive workloads.                       |
| **Document**     | Document analysis. Mean input: 123,831 tokens, Mean output: 736 tokens.           | Evaluates long-context document parsing (high prefill complexity, low decode).                          |
![[Pasted image 20260905223530.png]]
## Workload Generation

- **Arrival process:** Dynamic online request arrival pattern modeled after natural load fluctuations, featuring extreme load surges .

- **Tenant distribution:** Multi-tenant conditions are evaluated in mixed workload scenarios (such as combining MoonCake and BurstGPT, or Document, Conversation, and Assistant) .

- **Prompt distribution:** Skewed, heavy-tailed distributions where the 99th percentile input length is 3-10 times larger than the 90th percentile .

- **Output distribution:** Highly variable, long-tail distributions, with some workloads (like Conversation) generating outputs up to 25,000+ tokens .

- **Prefix overlap:** None (PiLLM does not assume or optimize for prefix overlap or cache locality).

- **Burst injection:** Includes production-derived bursty trace patterns with sharp spikes (Figure 1, 4) .

- **Synthetic modifications:** Anonymized production logs where all text content and personally identifiable information were removed, retaining only token length numbers and timestamps .

---

# 13. Simulation Setup

## Simulator

- **Simulator:** Custom discrete-event simulation framework.

- **Version:** Developed in Python.

- **What does it model?** Simulates request queues, arrival processes, scheduling policies, dynamic scaling latencies, and device utilization at large scale.

- **What does it NOT model?** Low-level hardware execution details, memory cache locality bank conflicts, kernel thread execution, or Infiniband/network packet-level routing contention.
## Simulated Cluster

| Parameter          | Value                                                                         |
| ------------------ | ----------------------------------------------------------------------------- |
| Number of GPUs     | Scaled to hundreds/thousands of GPUs to evaluate production-scale deployments |
| GPU Type           | Modeled after NVIDIA H800 / H100 specs using fitted execution curves          |
| Number of Replicas | Dynamically scaled based on simulation parameters                             |
| Model              | LLaMA-3.1 8B and DeepSeek V2 Lite                                             |
| GPU Memory         | 80 GB per GPU                                                                 |
| Network            | Perfect, latency-free interconnect simulated                                  |
| CPU                | Generically modeled host CPU                                                  |

## Simulation Assumptions

- Request execution timings behave according to the pre-profiled polynomial execution curves (Equation 2) .

- Instance scaling delays match uniform boot-time constants .

---

# 14. Real Hardware Setup

## Hardware

| Component      | Configuration                        |
| -------------- | ------------------------------------ |
| GPU            | NVIDIA H800 GPU                      |
| GPU Memory     | 80 GB HBM3 per GPU                   |
| Number of GPUs | 8                                    |
| CPU            | 2 Intel Xeon 6448Y CPUs              |
| RAM            | 1 TB DDR memory                      |
| Storage        | High-performance SSD cluster storage |
| Network        | NVLink all-to-all interconnect       |

## Serving Framework

- **Framework:** LightLLM (extended with PiLLM's custom global dispatcher, elastic Inter-GPU manager, and batch-aware memory sharing scheduler).

- **Version:** Extending ModelTC/LightLLM (repo state circa 2025/2026).

- **Model:** LLaMA-3.1 8B.

- **Precision:** FP16 / BF16 (typically standard for LLaMA-3.1 weights and KV cache in LightLLM).

- **Tensor Parallelism:** TP = 1 (each GPU instance hosts a complete copy of the model parameters).

- **Number of replicas:** Up to 4 Prefill instances and up to 4 Decode instances across the 8 physical GPUs.

- **Scheduler configuration:** Continuous batching with global SLO-aware dispatching (Algorithm 1) and utilization-aware Spike Reaction (Algorithm 2).

- **KV cache configuration:** Token-level linked-list memory slots (extends LightLLM) with batch-level statistical overcommitment based on the Central Limit Theorem.

---

# 15. Metrics

%%| Metric                | Definition | Why Important? |
| --------------------- | ---------- | -------------- |
| Throughput            |            |                |
| TTFT                  |            |                |
| TPOT                  |            |                |
| P50                   |            |                |
| P95                   |            |                |
| P99                   |            |                |
| Jain's Fairness Index |            |                |
| Cache Hit Rate        |            |                |
| GPU Utilization       |            |                |
| SLO Attainment        |            |                |
|                       |            |                |
|                       |            |                |
%%
%%## Metric Formulas

```text
Add important formulas here.
```
%%
---

# 16. Baselines

| Baseline                       | Why Compared?                                                                                                          | Strong / Weak?                                                                                                                                                                                                                                                                                                           |     |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --- |
|                                |                                                                                                                        |                                                                                                                                                                                                                                                                                                                          |     |
| **vLLM**                       | Industry-standard LLM serving framework using PagedAttention to maximize memory utilization.                           | **Strong:** Achieves high memory utilization and throughput by avoiding static memory blocks. <br>**Weak:** Highly aggressive; suffers from extremely high eviction (preemption) rates (peaking at 68.39% in Conversation workloads) because it cannot predict future sequence generation lengths.                       |     |
| **PastFuture**                 | Recent scheduling baseline that optimizes SLA/SLO compliance using request-level probabilistic output predictions.     | **Strong:** Successfully maintains a 0.00% request eviction rate. <br>**Weak:** Significantly underutilizes physical GPU memory (limiting average batch sizes) because running predictions and sampling at individual request level does not exploit the statistical smoothing of batches.                               |     |
| **SGLang**                     | State-of-the-art continuous batching and rate-based scheduling framework.                                              | **Strong:** Maintains a 0.00% request eviction rate by controlling entry rates. <br>**Weak:** Conservative memory allocation severely underutilizes GPU physical memory (limiting batch concurrency to only 483-431 active requests vs. PiLLM's 798-639).                                                                |     |
| **Metrics-based scaling (MR)** | Industry-standard auto-scaler (e.g., KServe) which scales GPU replicas dynamically using hardware utilization metrics. | **Strong:** Simple, hardware-independent, and works well for steady workloads. <br>**Weak:** Suffers from a delay/lag of one full scheduling window during sudden traffic spikes, violating SLO deadlines. It also cannot distinguish prompt complexity because both short and long prompts maintain steady utilization. |     |


## Missing Baselines?

> What baseline should have been included but was not?

---

# 17. Key Results


![[Pasted image 20260905223747.png]]

![[Pasted image 20260905223807.png]]

![[Pasted image 20260905223828.png]]

---

# 18. Failure Cases and Limitations

## Where Does It Fail?
- Physical Capacity Saturation during Massive Bursts
- Non-Stationary Workloads with Sudden Behavioral Shifts

## Limitations Stated by Authors
- **Centralized Controller Bottleneck:** The global control plane handles all request queuing, dispatching, and auto-scaling. In massive clusters with thousands of physical GPUs, this centralized scheduler can become a performance and latency bottleneck.

- **Disaggregated Phase Pool Waste:** Prefill and decode instances are physically separated into rigid GPU pools. If prefill instances are completely idle while decode instances are heavily saturated, resources cannot be shared, although they load the exact same model parameters.

- **Controlled Latency Tradeoff:** PiLLM improves GPU utilization by increasing batch sizes. For the computation-intensive prefill phase, these larger batches inevitably extend processing time for some requests, causing a minor rightward shift in execution time (representing the throughput-latency tradeoff).
## Limitations I Identified
	Locality and Fairness is missing
## Scalability Concerns

- **CPU Overhead:** Updating the sliding window statistics and executing the dispatching queue scheduling (Algorithm 1 and 2) in real-time adds continuous computational overhead to the host CPU as request concurrency scale to thousands.

- **Memory / State Transfer Overhead:** In the disaggregated setup, KV cache states must be transferred from prefill GPUs to decode GPUs progressively after each attention layer. As cluster size grows, this high-volume state transfer places a heavy burden on the cluster network interconnects.

- **Metadata Scaling:** Managing separate statistical sliding windows for hundreds of different tenants or workload scenarios increases control-plane metadata complexity and diminishes the statistical aggregation benefits of the Central Limit Theorem.

---

%%# 19. Challenges and Engineering Problems

|Challenge|Cause|Solution Used|
|---|---|---|
||||

## Failure / Edge Cases%%

---

# 20. Comparison With FairRoute

## Which Signal Does This Paper Optimize?

| Fairness | Locality | Prediction |
| -------- | -------- | ---------- |
| No       | No       | Yes        |

## What Can FairRoute Reuse?
Prediction

## What Does FairRoute Need Beyond This?

## Threat to FairRoute

> Could this paper invalidate the FairRoute research gap or make the proposed contribution trivial?
> No way

## How Could This Paper Be Extended to Compete With FairRoute?

Add fairness and locality

---

# 21. Baseline Decision

## Should We Implement This?

-  Yes, directly

### Reason - one of 3 pillers

## Comparison Priority

`High `

---

# 22. Implementation Difficulty

## Difficulty

 Hard`

## Why?

No code and artifacts

## Dependencies
Python ....


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

- No pre-emption (Not Fair)
- Not KV-Cache aware (No Locality)
- Does Batched Prediction {if the request can be served by a node?, more nodes required?, }
- Assumes all prefill instances are of same type and all decode instances are of same type
- Instances required is calculated considering only the FLOPS, memory constrains are omitted.

%%
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

- [x]  Yes
    
- [ ]  No