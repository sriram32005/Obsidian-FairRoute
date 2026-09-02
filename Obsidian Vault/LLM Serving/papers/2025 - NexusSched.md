---
title: A Predictive and Synergistic Two-Layer Scheduling Framework for LLM Serving
authors:
  - Yue Zhang
year: 1 OCT 2025
venue: ""
paper_type: research
status: read
rating:
url: ""
pdf: ""
tags:
---


# 2025 - NexusSched

> [!abstract] One-Line Summary  
> Write the entire paper's contribution in **1–2 sentences**.

---

# Problem

## Problem Statement

**What precise problem does this paper solve?**

LLM inference serving typically scales out with a two-tier architecture: a cluster router distributes requests to multiple inference engines, each of which then in turn performs its own internal scheduling. 

However, this commonly used paradigm suffers from critical, systemic inefficiency caused by the information gaps across two layers. 

At the cluster-layer, the router mainly relies on lagging, coarse-grained metrics, such as average latency and queue length to make decisions, resulting in “decision lag” that leads to suboptimal request routing. At the engine-layer, static heuristic scheduling policies cannot effectively handle the dynamic workloads, leading a poor balance between latency and throughput. Besides, these gaps may cause SLO violations and resource waste, especially in heterogeneous cloud environments.

## Why Does This Problem Matter?

one inevitable limitation of these static schedul- ing policies is the internal information gap. They rely heavily on predefined heuristics (e.g., a static token budget) that cannot accommodate to the dynamic nature of real- world workloads, including shifting request characteristics (e.g., context/generation lengths) and arrival patterns (e.g., burstiness, rate changes)



## Why Do Existing Approaches Fail?

These static policies fail to optimize performance under Service Level Objective (SLO) constraints adaptively. This results in an unavoidable dilemma: either sacrificing Quality of Service (QoS) or re- sorting to inefficient over-provisioning to handle peak loads.

Horizontal scaling is also widely adopted to serve massive user requests by deploy- ing large clusters with tens or even hundreds of inference instances. This paradigm naturally formulates the two-tier scheduling architecture: where an upper-level cluster router acts as a traffic gateway, distributing incoming requests to multiple downstream inference engine instances, and each engine instance performs its own engine-layer batching and scheduling operations

This two-tier architecture introduces one critical external information gap between the router and the engines. Existing routers typically rely on hand-crafted heuristics or coarse-grained, lagging metrics like average latency for load balancing. This creates a significant “decision lag”, making it difficult for the router to perform reason- able operations, and can result in unpredictable suboptimal placements that cause SLO violations.


---

# Target Workload

-  SLO-aware
    
-  Long-context
    
-  Heterogeneous Requests
    
-  Online Serving

---

# 4. Core Contribution

## Core Insight

> Existing black-box based approaches, such as polynomial regression models, generalize poorly because that they are agnostic to an instance’s service capability, and fail to model the complex, non-linear effects of batching.

## Core Problems To Address

**1. Internal Information Gap: Static Scheduling vs. Dynamic Workloads**:  

Information gap between the scheduler’s static sched- uling policy and the dynamic nature of the workload. Existing inference engines use parameters such as `
	- `num-batched-tokens`, which determines the max number of tokens processed in asingle forward pass and thus directly controls the batch size and computational efficiency of each iteration 
A larger `num-batched-tokens` value allows the system to complete prefill in fewer, more efficient passes, thereby reducing TTFT. However, this is prone to cause head-of-line blocking that worsens the TPOT of concurrent requests.

A smaller `num-batched-tokens` value tends to lower TPOT and improve generation throughput, but it significantly increases TTFT by breaking long prompts into multiple inefficient iterations

**2. External Information Gap: Blind Cluster-Layer Routing**: 

Production routers are functionally “blind”, lacking awareness of the real-time, fine-grained state of the engines. They typically rely on coarse-grained, time-aggregated met- rics (e.g., 5-second rolling average latency) to make routing decisions.

During this lag, the router with stale information may route latency-sensitive requests to an engine that is already saturated. Such reactive routing fundamentally undermines proactive, capacity-aware load distribution based on past performance and is a primary cause of unpredictable SLO violations.

**3. Heterogeneity as an Inefficiency Amplifier**

In modern inference clusters, it has become a common prac- tice to deploy hybrid hardwares (e.g., NVIDIA H100s with A100s/A6000s) to balance infrastructure cost and instance performance.
Such heterogeneity can fundamentally amplify the inherent inefficiency of both engine-layer scheduling and cluster-layer routing

At the engine-layer, hardware heterogeneity makes static scheduling policies less effective. 
An empirically-tuned heuristic or parameter optimal for one configuration may be severely suboptimal for another. Relying on static, offline profiling to solve every possible combination is operationally infeasible and can incur prohibitive overhead, which high- lights the need for dynamic, online adaptation.

At the cluster-layer, traditional load balancing metrics like queue length can mislead proxy for actual loads when introducing heterogeneity, and greatly degrade the performance. 


## Main Contributions

 - Our model is structurally-informed which an accurately model an instance’s performance capability across diverse workload deployments
- Its online learning capability enables a “zero-config” deployment that rapidly adapts to new hardware without costly offline profiling

---

# System Architecture

NexusSched:  a cross-layer framework that enables predictive decision-making at both layers.
At the engine-layer: 
	- Performance model enables LENS (Latency-Enhanced Node Scheduler) to serve as a proactive, SLO-aware scheduler. 
	- LENS bridges the internal information gap by using the model’s latency prediction results to dynamically balance the scheduling performance between latency and throughput. 
	- LENS then crafts reliable batching plans that meet SLOs under real-time loads
At the cluster layer: 
	- PRISM (Predictive Routing with Instance State Metrics) uses predictive signals exported from LENS as the forward-looking state vectors to bridge the external gap. 
	- PRISM replaces reactive load balancing with predictive request orchestration, using a multi-dimensional scoring function that evaluates signals like predicted latency and pending workload to match each request to the suitable instance.


![[Pasted image 20260902201712.png|699]]


## End-to-end Request Lifecyle

1. When request 𝑅 ar- rives, the cluster-layer PRISM queries for the latest forward- looking state of all engines.(e.g., predicted processing la- tency b𝐿𝑒 , pending workload 𝑊𝑒 ).
2. It then scores candidate engines and dispatches 𝑅 to the highest-scoring engine 𝐸 before execution begins, enabling proactive routing
3. Upon arrival at 𝐸, request 𝑅 joins the local queue and is handled by LENS, which drives a continuous optimization loop. 
4. In each Scheduling cycle, LENS  
	1. Senses queue and memory status via the **State Monitor**; 
	2. Sets an adaptive latency target from the SLOs provided by the **Constraint Manager** and the current load
	3. Invokes the **Scheduling Optimizer** to produce the batching plan for the next iteration. This plan specifies which requests (including 𝑅) will run and how the next iteration is batched (e.g., per-iteration token budget, prefill-slice sizes).
5. As requests are processed, LENS continuously self-calibrates its performance model with real-time data. 
6. It exports these refined, forward-looking capacity signals back to PRISM. 

## LENS: Proactive Engine-Layer Scheduling

### 1. Performance Model

Accurate, generalizable performance prediction is the cor- nerstone of proactive, SLO-aware scheduling. It provides the “foresight” that static heuristics lack, which is key to bridging the information gap within the engine. 
To achieve this, we designed a performance model that not only predicts latency but also perceives an instance’s underlying service capability.

- **Workload dependency ( token-linear scaling)**: For a fixed batch size 𝐵, the per-step latency 𝑇 grows approximately linearly with the total tokens in the batch 𝑆.This reflects that the total amount of computation is the primary reason to cause latency; 𝑆 dominates 𝑇 at fixed 𝐵.
- **Parallelism saturation (memory-bound at fixed 𝑆)**:  For a roughly constant total token bud- get 𝑆, increasing the batch size 𝐵 leads to a rise in model run time. 
- The root cause is that higher parallelism triggers more reads/writes and page management for the KV-cache, which lets the memory subsystem (HBM/cache) become the main bottleneck and shifts execution from compute-bound to memory-bound. 

 Single- iteration latency 𝑇 (𝐵, 𝑆) into three interpretable parts: 
	 1. fixed overhead
	 2. computation time
	 3. linear overheads related to batch size and sequence length
![[Pasted image 20260902203113.png|645]]
 here,
	 - S = total scheduled tokens
	 - B = Batch size / Parallelism
	 - 𝜏0  = load-independent fixed overhead
	 -  Work(S) = total workload -> Approximated by Work(𝑆) = 𝑤0 + 𝑤𝑠𝑆
	 - Thr(B,S) = effective throughput 
	![[Pasted image 20260902203408.png]]
	
To balance stability and agility, we draw on the concept of multi-timescale adaptation from control theory, stratifying the update rates of the model parameters:
	1. **Structural Parameters ({𝑃max, 𝑘𝐵, 𝑘𝑆 }):** These parameters define the basic shape of the performance curve and the physical limits of the hardware, making them relatively sta- ble. We employ a low-frequency, long-window strategy, aggregating data from thousands of batches for non-linear fitting to ensure the model robustly converges to the system’s physical characteristics.
	2. **Linear Parameters ({𝑤𝑠, 𝜏𝐵, 𝜏𝑆 }):**  These parameters reflect short-term performance drifts caused by the current load, software stack version, etc. We use a high-frequency, short-window strategy, employing lightweight regression on the last few dozen batches for rapid fine-tuning to ensure the model remains sensitive to the current system state.
### 2. SLO - Aware Scheduling

**Scheduling Optimizer:** 
	- It formulates decisions using a dynamic optimization algorithm (see Algorithm 1) that is both SLO-aware and load-adaptive. 
	- This algorithm translates user-defined SLOs into a performance budget that dynamically adjusts to real- time load and solves a constrained optimization problem to find the suitable batching plan that maximizes throughput. 
	- This shifts the engine scheduler from passive execution to proactive scheduling.
	
![[Pasted image 20260902203935.png|351]] 

>[!Algortihm Explanation]
>The algorithm is invoked at the start of each iteration, prior to model execution. First, it ob- tains a real-time snapshot of the engine’s internal state—the current running queue 𝑄run 𝑖 and waiting queue 𝑄wait 𝑖 —via the State Monitor component. Next, it parses the SLO ob- jects from the Constraint Manager and, together with the current load, calls TargetLatency (Sec. 4.2.2) to produce the adaptive latency target 𝑇adaptive (line 3). 
>
>The core of the algorithm is a constrained search process (lines 6–24) aimed at finding the suitable batch configuration for the next execution step. The process iterates through all reasonable batch sizes 𝐵 as candidate solutions. For each can- didate 𝐵, instead of performing a complex inverse calculation, the scheduler leverages the monotonicity of the performance model to efficiently find the suitable token budget 𝑆 that can be processed within the adaptive target latency 𝑇adaptive via a binary search (lines 8–16).
>
>After determining this target configuration (𝐵, 𝑆), the al- gorithm constructs the actual candidate batch A using a greedy policy (line 17). This policy adheres to the princi- ple of continuous batching by prioritizing requests already in progress, and then adding new requests from the wait- ing queue in a FCFS order until the limits of either 𝐵 or 𝑆 are reached. Once constructed, the algorithm again uses the model’s forward prediction capability to accurately calculate the expected latency of this actual batch, 𝑇 ★ pred (line 18), and evaluates its error relative to the target latency (line 19). Fi- nally, the algorithm selects the batch configuration among all candidate batches that most closely matches the dynamic target as its final decision.

While the theoretical time complexity is 𝑂 (𝑁𝑤𝑎𝑖𝑡 · 𝑄𝑚𝑎𝑥 ), the practical overhead is negligible (Fig. 11b). This efficiency stems from the fact that both the number of waiting re- quests, 𝑁𝑤𝑎𝑖𝑡 , and the maximum batch size, 𝑄𝑚𝑎𝑥 , are small 6 and bounded in a limited serving engine, while the internal binary search is constrained to a maximum of 10 iterations. This efficiency is further improved by an early-exit mechanism (lines 23–24) that terminates the search once a sufficiently suitable solution is found

**A modelable TTFT–TPOT relationship**: Under continuous batching with chunked prefill, the measured TTFT (𝑡𝑝 ) decreases approximately linearly with increasing TPOT (𝑡𝑑 ) within the operating range.
	Function ˆ𝑡𝑝 (𝑡𝑑 ), which estimates the TTFT for a given TPOT: ˆ𝑡𝑝 (𝑡𝑑 ) ≈ 𝛼 − 𝛽. 𝑡𝑑 , 𝛽 > 0. 
	Here, 𝛼 is the theoretical maximum TTFT (the y-intercept), and 𝛽 is the trade-off rate between TTFT and TPOT.

For a request with expected decode length ¯𝐿 (tokens), the expected end-to-end latency is 
	𝑇e2e (𝑡𝑑 ) = ˆ𝑡𝑝 (𝑡𝑑 ) + ¯𝐿 · 𝑡𝑑 = 𝛼 + ( ¯𝐿 − 𝛽) 𝑡𝑑 .


## Cluster-Layer Routing Design

Two core components: 
	1. A set of **forward looking state metrics** that provide predictive insights
	2. **PRISM Routing Algorithm** that consumes these metrics to make globally optimal decisions
### 1. State Sharing for Predictive Routing

To enable predictive routing, our design is founded on a set of forward-looking state metrics exported by each LENS-enabled engine. These metrics serve as an “informa- tion bridge”, providing the cluster-layer with a precise, near- future view of each engine’s real capacity and availability.

At each moment 𝑡, each engine 𝑒 exposes a state vec- tor s𝑒 (𝑡) to the routing layer:
![[Pasted image 20260902210642.png]]

### 2. PRISM: Predictive Routing
Armed with the real-time, forward-looking state exported by each engine, 
PRISM solves a single-objective problem: *maximize throughput subject to SLO constraints.*

PRISM employs a scoring function score(e,r). This scoring matches each request r to most suitable candidate engine e after evaluation of all the engine candidates. 
![[Pasted image 20260902211022.png]]
wi = weights (default value of 1). Each Si denotes on of the scoring dimensions, which are: 
	1. SLO Margin Assessment (𝑆latency): Evaluates engine immediate responsiveness using the predicted processing latency Le.
	2. Relative Load Assessment (𝑆load): Assesses the engines medium-to-long term load pressure using composite metric We which captures both current and future concurrency pressure. 
	3. Capacity Admission Assessment (𝑆capacity): Acts as a reliability gatekeeper to ensure scheduling success by using effective available KV Cache.
	4. Session Affinity Assessment (𝑆affinity): Designed to optimize performance for conversational applications. Default value is neutral 1. If an engine 𝑒 has already cached the historical KV cache for request 𝑟 , its score is boosted to a constant 𝛽 > 1, incentivizing the router to select nodes where cache reuse is possible

---

# Experimental Setup

Cluster: 
1. Homogeneous Cluster: 8 x NVIDIA H100 connected via NVLink
2. Heterogeneous Cluster: 1x NVIDIA H100 + 1x NVIDIA A100 + 2x NVIDIA A6000

Models:
1. Llama3 - 8B
2. QwQ-32B (FP8-Quantized)

Single model-per-GPU replicated config with sole exception of Llama3-8B, which used 2-way tensor paral- lelism across two A6000 GPUs.

Workloads: 
1. Production Workloads: FlowGPT Traces - Real production system
2. Public Workloads: 
	1.  arXiv-Summarization (long-text sum- marization),
	2. Code-Reasoning (code generation)
	3. ShareGPT (dialogue)

![[Pasted image 20260902212155.png]]

Baselines: 
1. Engine Layer Baselines: 
	1. vLLM (v0.9.2, v0)
	2. Sarathi-Serve
2. Router Layer Baselines:
	1. Round-Robin(RR)
	2. Session Affinity
3. Combinations compared:
	1. vLLM + RR
	2. Sarathi + Session

Metrics:
1. p50, p90 end-to-end latency
2. p50 TTFT
3. p50 TPOT
4. SLO Attainment: Percentage of requests successfully completed within SLOs (QoS)\

SLOs:
1. FlowGPT: 
	1. TPOT SLO: 12ms
	2. TTFT SLO: 2s

---

# Key Results

## Main Quantitative Results

![[Pasted image 20260902212732.png]]

![[Pasted image 20260902213021.png]]

## Main Takeaway
1. NexusSched is almost always at the lowest level for all latency metrics, with a flatter growth curve.
2. This translates to a 20%–50% reduction in P50 and P90 end-to- end latency 
3. An average SLO attainment improvement of over 43%
4. NexusSched serves over 3× as many requests as the best baseline under SLO constraints.
---

# Comparison With FairRoute

## Which Signal Does This Paper Optimize?

| Fairness | Locality | Prediction |
| -------- | -------- | ---------- |
| Yes / No | Yes      | Yes        |

## What Can FairRoute Reuse?

Prediction Algorithm (PRISM + LENS)

## What Does FairRoute Need Beyond This?

Integrate the concept of fairness

## Threat to FairRoute

> Could this paper invalidate the FairRoute research gap or make the proposed contribution trivial?
> Nope


## How Could This Paper Be Extended to Compete With FairRoute?

Consider fairness as a parameter to optimize 
