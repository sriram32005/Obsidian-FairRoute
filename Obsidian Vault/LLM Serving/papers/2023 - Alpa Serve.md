---
title: "AlpaServe: Statistical Multiplexing with Model Parallelism for Deep Learning Serving"
authors:
  - Zhuohan Li, Lianmin Zheng, Yinmin Zhong, Vincent Liu, Ying Sheng, Xin Jin, Yanping Huang, Zhifeng Chen, Hao Zhang, Joseph E. Gonzalez, Ion Stoica.
year: "2023"
venue: ""
paper_type: research
status: unread
rating:
url: ""
pdf: "[[2023 - Alpa Serve.pdf]]"
tags:
---


# 2023 - Alpa Serve

> [!abstract] One-Line Summary  
> Write the entire paper's contribution in **1–2 sentences**.

---

## 1. Paper Metadata

| Field                   | Notes                                                                                                                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Authors**             | Zhuohan Li, Lianmin Zheng, Yinmin Zhong, Vincent Liu, Ying Sheng, Xin Jin, Yanping Huang, Zhifeng Chen, Hao Zhang, Joseph E. Gonzalez, Ion Stoica. |
| **Year**                | 2023                                                                                                                                               |
| **Venue**               |                                                                                                                                                    |
| **Paper Type**          | Scheduling / Routing                                                                                                                               |
| **Code Available?**     | Yes                                                                                                                                                |
| **Artifact Available?** | Yes / No                                                                                                                                           |
| **Related System**      | Custom - Alpa                                                                                                                                      |

---

# 2. Problem

## Problem Statement

The paper solves the problem of efficiently placing and parallelizing multiple large deep learning models concurrently across a distributed cluster to maximize the percentage of requests served within strict latency Service Level Objectives (SLOs) under highly bursty traffic conditions.

**Why Does This Problem Matter?** 
Model sizes are growing exponentially, demanding massive GPU memory and computation just to execute real-time inference. Production environments increasingly serve multiple fine-tuned variations of these large architectures for different downstream tasks and A/B testing. Real-world inference traffic is highly unpredictable, frequently experiencing demand spikes up to 50x the average rate. Provisioning sufficient dedicated hardware to guarantee tight latency deadlines during these traffic spikes is prohibitively expensive and leaves expensive accelerators underutilized during normal operation.

**Why Do Existing Approaches Fail?**

- **Replication-Only Strategies:** Standard serving systems (e.g., Triton, Clipper, Nexus) allocate a dedicated GPU to a single model as long as it fits in device memory. When a traffic burst targets one model, its dedicated GPU creates a queue and misses SLOs, while GPUs dedicated to other models sit idle. Compensating for this requires over-provisioning hardware to create peak-load replicas, stranding resources.
    
- **Swapping Overheads:** Systems that dynamically swap models in and out of memory to handle changing traffic (e.g., Clockwork) incur multi-second delays when transferring large model weights to the GPU, making it impossible to meet tight latency constraints on the critical path.
    
- **Ignored Multiplexing Opportunities:** Traditional systems view model parallelism strictly as a necessity for models that exceed the memory of a single device. They fail to recognize that partitioning models—even those that fit on one GPU—reduces the per-model memory footprint, enabling model co-location. This prevents systems from statistically multiplexing pooled cluster resources to handle sudden traffic bursts.
    
- **Imbalanced Manual Partitioning:** Standard manual parallelization strategies assign an equal number of layers to each pipeline stage, ignoring the heterogeneous computational costs of modern network layers. This creates pipeline bottlenecks and uneven partition overheads that degrade inference performance.


---

# 3. System Context

## System Layer

- [ ]  Cluster Routing
    
- [ ]  Request Scheduling
    
- [ ]  GPU Scheduling
    
- [ ]  Distributed Inference
    
- [ ]  Simulation / Evaluation Infrastructure
    
- [ ]  Other: **Model Placement & Auto-Parallelization Compilation**
    

## Target Workload

- [ ]  Multi-tenant
    
- [ ]  Bursty Traffic
    
- [ ]  SLO-aware
    
- [ ]  Heterogeneous Requests
    
- [ ]  Interactive Serving

### Workload Characteristics

- [ ] Number of tenants: multiple (for upto 60 models)
    
- [ ] Arrival pattern: Poisson and Gamma processes and real word serverless traces (Azure)
    
- [ ] Prompt length: 2048 tokens per query 
    
- [ ] Output length: non auto regressive models, so NA
    
- [ ] Prefix overlap: doesnt work on prefixes
    
- [ ] Burstiness: high, upto 50x avg request rates
    
- [ ] SLO assumptions: under strict Service Level Objectives (SLOs) defined as tight multipliers.
    

---

# 4. Core Contribution

## Core Insight

> Model parallelism is not just a memory workaround for massive models; it is a powerful mechanism for statistical multiplexing during prediction serving. By partitioning a model across multiple GPUs—even when that model easily fits onto a single GPU—a system can pool distributed compute resources to instantly absorb unpredictable traffic spikes. This trades minor parallelization overheads for massive gains in burst tolerance and reduced average completion times.

**Main Contributions**

- **Trade-off Analysis:** A rigorous mathematical and empirical breakdown of the complex design space balancing statistical multiplexing benefits against the latency overheads introduced by inter-operator and intra-operator parallelism.
    
- **Serving-Specific Auto-Parallelization:** Extensions to existing model-parallel training compilers that adapt dynamic programming and integer linear programming specifically for inference. This optimizes solely for the forward-pass execution and minimizes maximum pipeline stage latency.
    
- **Novel Placement Algorithms:** A two-level scheduling mechanism combining an enumeration-based cluster partitioning search with a simulator-guided greedy heuristic. This automatically determines the optimal way to allocate device groups, select model replicas, and assign parallel configurations across a cluster.
    
- **Comprehensive Evaluation:** Empirical proof utilizing a 64-GPU cluster and production traces demonstrating the system handles up to 10x higher request rates or 6x more bursty traffic while maintaining >99% SLO attainment.
    

# What Is Actually Novel?

The fundamental paradigm shift is utilizing model parallelism proactively for latency reduction and multi-model resource pooling, rather than treating it purely as a memory-expansion necessity for training or for models that physically exceed a single device's capacity. Furthermore, AlpaServe proves that a strategically calculated **static** placement using model parallelism can consistently outperform state-of-the-art **dynamic** serving algorithms (such as Clockwork) that attempt to aggressively swap models in and out of memory on the fly to chase traffic changes.

---
# Serving Constraints and Parallelism Types

- **The Serving Bottleneck:** Dynamically swapping large models into GPU memory is too slow to handle unpredictable, bursty request traffic on the critical path. This forces standard systems to over-provision expensive hardware to meet tight latency Service Level Objectives (SLOs).
    
- **Intra-operator Parallelism:** Partitions a single mathematical operator (like matrix multiplication) across multiple devices. This expands available compute and memory to reduce single-request latency, but it incurs significant collective communication overheads that cannot be overlapped with computation.
    
- **Inter-operator (Pipeline) Parallelism:** Assigns different operators in the execution graph to distributed devices in a sequential pipeline. This reduces per-device memory usage and communicates less data than intra-op parallelism, but it typically increases single-request execution time due to pipeline delays.

**The Multiplexing Advantage** 

Using a 2-GPU, 2-model case study, the authors demonstrate that co-locating partitioned models across both GPUs allows the system to utilize the entire cluster to process a sudden burst of requests for a single model. This statistical multiplexing drastically reduces average completion time (e.g., up to a 6.6x reduction in skewed traffic) by avoiding the queueing delays that occur when a dedicated, single-GPU replica is overwhelmed.

**When Model Parallelism is Beneficial**

The paper defines a specific operational envelope where partitioning out-performs replication:

- **Device Memory:** Highly beneficial when memory is constrained; the advantage disappears entirely if a single GPU is large enough to replicate and hold all models simultaneously.
    
- **Traffic Burstiness:** Bursty traffic (indicated by a high Coefficient of Variance) significantly amplifies the benefits of model parallelism.
    
- **Request Arrival Rate:** Most effective at low-to-medium arrival rates. As traffic approaches the cluster's absolute peak processing capability, standard replication overtakes model parallelism because parallel overheads start bottlenecking the saturated cluster.
    
- **SLO Strictness:** Crucial for satisfying tight latency SLOs (deadlines less than 10x the model's base execution latency).
    

**The Cost of Parallelism**

- **Inter-op Overheads:** The primary performance penalty is "uneven partition overhead," meaning the overall pipeline is bottlenecked by the execution time of its slowest individual stage.
    
- **Intra-op Overheads:** The primary penalty is communication overhead from synchronizing intermediate tensors across devices.

- **The above two logics are part of alpa, from which alpaserve evolved**
    
- **Queueing Theory Proof:** Using deterministic queue analysis, the authors mathematically prove that a unified pipeline queue yields lower average waiting times ($W_{pipeline} \le W_{simple}$) than independent dedicated queues, provided that the communication overhead factor ($\alpha$) and the uneven stage factor ($\beta$) remain below calculated thresholds.


	**In depth understanding of the queuing theory** 

The queueing theory proof models the serving system using M/D/1 queues, representing a state where requests arrive randomly (following a Poisson process) but execute in highly predictable, constant time (Deterministic service time). The mathematical analysis proves why replacing two isolated queues with a single, unified pipeline reduces overall latency, even when parallelization overheads are introduced.

**1. The Single-Device Baseline** For a single GPU handling a request rate of $\lambda_0$ with a deterministic execution latency of $D$, the average request latency $W$ (which includes both the physical execution time and the time spent waiting in the queue) is calculated as:

$$W = D + \frac{\lambda_0 D^2}{2(1 - \lambda_0 D)}$$

**2. Simple Placement (Two Independent Queues)** When two models are placed on two dedicated GPUs, the system operates as two completely isolated queues. If the total request rate is $\lambda$, and the traffic is split such that one model receives a fraction $p$ and the other receives $1-p$, the overall average latency is the average of these two distinct queues:

$$W_{simple} = D + \frac{p^2 \lambda D^2}{2(1 - p\lambda D)} + \frac{(1 - p)^2 \lambda D^2}{2(1 - (1 - p)\lambda D)}$$

This total latency reaches its absolute mathematical minimum only when the request traffic is perfectly balanced across both models ($p = 1/2$). If one model becomes more popular and receives a larger share of the traffic, its specific queueing delay spikes, dragging the overall average latency significantly higher.

**3. Model-Parallel Placement (One Unified Queue)** By partitioning both models across both GPUs using a 2-stage pipeline, the system collapses the architecture into a single unified queue handling the full combined rate $\lambda$. The latency calculation relies on two new variables: the total time for a single request to traverse the entire pipeline ($D_s$) and the time taken by the slowest individual pipeline stage ($D_m$). The new average latency becomes:

$$W_{pipeline} = D_s + \frac{\lambda D_m^2}{2(1 - \lambda D_m)}$$

**4. The Comparison (Assuming Zero Overhead)** If the pipeline splits the computational work perfectly without introducing any communication delays, the total execution time remains the same ($D_s = D$), but each stage only takes half the time ($D_m = D/2$). Substituting these values into the formulas for a perfectly balanced traffic scenario ($p = 1/2$) yields:

$$W_{simple} = D + \frac{\lambda D^2}{4 - 2\lambda D}$$

$$W_{pipeline} = D + \frac{\lambda D^2}{8 - 4\lambda D}$$

The mathematical result demonstrates that the queueing delay for the model-parallel pipeline is exactly half that of the isolated simple placement. Furthermore, when traffic becomes skewed ($p \neq 1/2$), $W_{simple}$ rapidly increases while $W_{pipeline}$ remains completely unaffected, making the performance gap even wider in favor of model parallelism.

**5. Establishing Overhead Thresholds** Because real-world parallel execution introduces latency penalties, the analysis models specific overhead factors to determine viability:

- **Communication Overhead ($\alpha$):** Scales both the total latency and the stage latency ($D_s = 2D_m = \alpha D$, where $\alpha \ge 1$).
    
- **Uneven Partition Overhead ($\beta$):** Inflates the bottleneck stage latency due to imbalanced computation ($D_m = \beta D / 2$, where $\beta \ge 1$), while the total execution time remains roughly $D$.
    

 The system remains beneficial as long as the inequality $W_{pipeline} \le W_{simple}$ holds true. The maximum tolerable overheads ($\alpha$ and $\beta$) are mathematically bounded by the total cluster utilization ($\lambda D$). When cluster utilization is exceptionally high, the queues are saturated, meaning overheads must be kept extremely low to avoid missing deadlines; conversely, when utilization is low, the pipeline can tolerate significantly higher parallelization overheads while still outperforming simple placement.

---

# Methodology

	To improvise the Alpa into a better framework, following algorithms were embedded as a strategy.

**Automatic Parallelization for Inference**

- **Purpose:** The system generates a comprehensive list of potential parallelization strategies for every model so the placement algorithm can select the optimal combination.
    
- **Compiler Modifications:** AlpaServe extends Alpa, an existing auto-parallelization training system, and modifies its two compilation passes (inter-op and intra-op) specifically for serving inference workloads.
    
- **Inter-op Pass (Dynamic Programming):** AlpaServe reformulates the dynamic programming algorithm to focus entirely on minimizing the maximum latency of any single pipeline stage. Because inference only requires a forward pass (no backward propagation or weight synchronization), the system accelerates profiling by only measuring $K$ layers instead of exploring $O(K^2)$ combinations.
    
- **Intra-op Pass (Integer Linear Programming):** The integer linear programming formulation is explicitly modified to drop any configurations that use data parallelism. Data parallelism is instead managed purely through model replication by the placement algorithm.

The placement algorithm is AlpaServe's core decision engine, responsible for mapping a set of deep learning models onto a physical GPU cluster to maximize the number of requests that meet their latency deadlines (SLO attainment). Because finding the perfect setup is a massive combinatorial optimization problem, AlpaServe splits the task into two interacting algorithms: one to partition the cluster, and one to select models for those partitions.

The auto-parallelization compiler defined in Section 4.1 acts as the foundational configuration generator for both of these placement algorithms. Instead of the placement algorithms blindly guessing how to split a model, they rely on 4.1 to provide mathematically optimized parallelization blueprints.

Here is exactly how the 4.1 compiler is integrated into the two placement algorithms:


**Algorithm 2 (Cluster Partitioning & Strategy Enumeration)**

- This outer algorithm clusters models of similar sizes into disjoint "buckets" and tests different ways to divide the cluster's GPUs into device groups to serve those buckets.

- **Where 4.1 comes in:** For each device group, Algorithm 2 calls a function `get_potential_parallel_configs`. This function relies on the auto-parallelization compiler from Section 4.1 to enumerate all the viable inter-operator and intra-operator parallel configurations for that specific hardware group. Algorithm 2 then passes this list of valid configurations down to Algorithm 1 to be evaluated.
    
**Algorithm 1 (Simulator-Guided Model Selection)**

- This inner algorithm acts as a beam search, testing which specific models should be placed into the device groups using the parallel configurations provided by Algorithm 2.
    
- **Where 4.1 comes in:** At every iteration, when Algorithm 1 tests a new (model, device group) pair, it calls the function `parallelize(m, g, p)`. This directly applies the specific intra- and inter-op partitioning plan optimized by Section 4.1's dynamic programming and integer linear programming algorithms.
    
- Once the model is parallelized according to 4.1's blueprint, Algorithm 1 checks if the sliced model actually fits within the group's GPU memory constraints before running the simulator to calculate the resulting SLO attainment.
---

## Placement Algorithm

- **Purpose:** The algorithm navigates a complex combinatorial optimization space to partition the cluster, assign model replicas, and select shared model-parallel configurations to maximize the percentage of requests meeting their Service Level Objective (SLO).
    
- **Algorithm 1 (Simulator-Guided Model Selection):** Given a fixed cluster group partition and parallel configuration, this algorithm uses a beam search to assign model replicas. It enumerates all valid model-to-group pairs that satisfy memory constraints, runs a continuous-time discrete-event simulator using workload profiles to calculate exact SLO attainment for each valid pair, and iteratively advances the top-k solutions.
    
- **Algorithm 2 (Enumeration-Based Group Partition):** This outer loop enumerates various cluster partitions and parallel configurations, calling Algorithm 1 to evaluate each setup.

![[Pasted image 20260905192302.png]]

- **Model Bucketing:** To prevent "convoy effects"—where fast requests for small models get stuck in a queue behind slow requests for massive models—Algorithm 2 clusters models into disjoint "buckets" based on their size and latency. The algorithm determines the optimal placement for each bucket individually before concatenating the final solution

## Runtime Scheduling

- **Centralized Dispatch:** A central controller receives all incoming HTTP requests and dispatches each request to the specific device group that hosts the required model and currently has the shortest queue length.
    
- **Predictive Admission Control:** Each device group manages a simple first-come-first-serve (FCFS) queue. Because deep learning inference execution times are highly predictable, the group instantly calculates whether an incoming request can finish before its deadline. If the request will violate the SLO, the group immediately rejects it.
    
- **Exclusion of Swapping:** AlpaServe intentionally avoids swapping models dynamically between the CPU/disk and the GPU, as the loading overheads for large models take multiple seconds and would severely violate tight latency constraints.
    
- **Batching Optimization:** While the system supports dynamic batching, it is left disabled by default in the architecture; the authors note that for very large models, even a small batch size fully saturates the GPU, making the throughput gains of batching negligible under tight SLOs.

---
# Experimental setup and Evaluation

- **Evaluation Setup:** The system was tested on a 64-GPU cluster with 8 nodes serving large BERT and GShard MoE models, using highly bursty real-world Microsoft Azure traces to measure how well it meets strict latency deadlines (SLO attainment).
    
- **Massive Performance Gains:** Compared to state-of-the-art baselines like Clockwork++, AlpaServe achieved 99% SLO attainment using fewer GPUs, while handling 10x higher request rates, tolerating 6x more traffic burstiness, and meeting 2.5x tighter deadlines.
    
- **Scalability to Massive Models:** For models so large they require 16+ GPUs just to store their weights in memory, AlpaServe's automated space-sharing placement outperformed traditional manual, dedicated-GPU assignments.
    
- **Traffic Robustness:** Even when real-time incoming traffic wildly deviated from historical predictions, AlpaServe's static model-parallel setup absorbed the unexpected bursts better than baselines that dynamically swap models in and out of memory.
    
- **Batching Ineffectiveness:** The authors demonstrated that for large models under tight latency constraints, dynamic batching provides virtually no benefit because even tiny batch sizes instantly saturate the GPU.
    
- **Component Necessity (Ablation):** The evaluation proved that AlpaServe's custom inference compiler (which cuts parallelization overheads by up to 46.7%) and its two-stage placement algorithm are both strictly required to achieve the system's throughput and latency gains.

---
# Final Analysis 

- **Redefining Resource Efficiency:** AlpaServe consistently achieves a 99% Service Level Objective (SLO) attainment rate using significantly fewer GPUs than baseline systems like Selective Replication. By pooling device memory and compute, it successfully processes up to 10x higher request rates and meets 2.5x tighter latency deadlines without the need to over-provision hardware.
    
- **Superiority Under Unpredictable Traffic:** Real-world traffic is highly skewed and bursty. Instead of dynamically swapping models in and out of memory to chase shifting traffic—which incurs massive loading penalties—AlpaServe relies on an optimized static, model-parallel placement. This design instantly absorbs unexpected demand spikes, tolerating 6x more traffic burstiness and outperforming reactive online systems like Clockwork++ even when real-time traffic wildly deviates from historical predictions.
    
- **Universal Scalability:** The statistical multiplexing advantage holds true across all scales. For massive models that physically require 16 or more GPUs just to store their weights (e.g., BERT-104B), AlpaServe's automated space-sharing approach drastically outperforms the industry standard practice of assigning dedicated GPUs to individual models.
    
- **Validation of System Architecture:** The evaluation proves that AlpaServe's underlying innovations are strictly necessary to achieve these gains. The custom auto-parallelization compiler reduces pipeline overhead by up to 46.7% by balancing heterogeneous neural network layers, while the ablation study confirms that both the group partitioning and the greedy model selection algorithms are required to prevent the system from falling back to suboptimal performance.
    
- **Batching is Ineffective for Tight SLOs:** The analysis confirms that standard dynamic batching optimizations provide almost no benefit for large models under tight latency constraints, as even tiny batch sizes instantly saturate the GPUs and waiting to form batches causes requests to miss their strict deadlines.
---

# 10. Objective

## What Is Being Optimized?

- [ ]  Throughput : Evaluated as a secondary constraint; intra-op/inter-op affects throughput
    
- [ ]  End-to-End Latency : Evaluated in motivating trade-offs and queueing analysis
    
- [ ]  Tail Latency : Evaluated via P99 latency
    
- [ ]  Fairness
    
- [ ]  Cache Hit Rate
    
- [ ]  GPU Utilization : as a driver of burst absorption, but not the primary objective
    
- [ ]  **SLO Attainment : Primary objective: fraction of requests completed before deadline**
    
- [ ]  Cost : as minimizing the total number of physical GPUs needed to attain $\ge 99\%$ SLO

## Formal Objective

1. **Global Serving Objective (Section 4.2):** Maximize overall cluster SLO attainment across all requests in workload $W$ under GPU memory constraints:
    $$\max_{\text{Placement}} \frac{1}{\vert{}W\vert{}} \sum_{r \in W} \mathbb{I}\Big(\text{Latency}(r) \le \text{SLO}(r)\Big)$$
    $$\text{s.t.} \quad \sum_{m \in \text{Group}_g} \text{Memory}(m, p_g) \le \text{Memory\_Capacity}(g), \quad \forall g \in G$$
    
    where $\mathbb{I}(\cdot)$ is the indicator function, $p_g$ is the parallel configuration assigned to device group $g$, and requests that would violate the deadline are dropped.
    
2. **Inference Compiler Objective (Section 4.1):** Minimize the maximum pipeline stage execution latency during forward-pass execution (Dynamic Programming):
    
    $$F(s, k) = \min_{1 \le i \le k} \Big\{ \max \big\{ F(s-1, i-1), \, \text{latency}(i, k) \big\} \Big\}$$

    where $F(s, k)$ is the maximum stage latency when partitioning layers $1$ through $k$ into $s$ pipeline stages, and $\text{latency}(i, k)$ is the profiled forward-pass latency of layers $i$ through $k$.


## Optimization Tradeoff

> What gets worse when the optimized metric improves?
- **Parallelism Overhead vs. Peak Saturated Throughput:** Using intra-operator parallelism reduces single-request latency, but collective communication overhead reduces aggregate system throughput.
    
- **Pipeline Slicing Overhead vs. Low-Traffic Latency:** Using inter-operator pipeline parallelism enables statistical multiplexing across devices, but introduces uneven partition imbalance and stage-to-stage communication overhead ($D_s = \alpha D$), which increases execution time for individual requests under light or non-bursty traffic.
    
- **Replication vs. Multiplexing at Saturation:** When cluster arrival rates approach total saturation ($\lambda D \to 1$), the multiplexing benefits diminish, and model-parallel overheads cause AlpaServe to perform worse than simple replication.

---

## Workload Generation

- **Arrival process:** Poisson arrival process; synthetic Gamma arrival process (varying arrival rate and Coefficient of Variation $CV$); and production serverless traces from Microsoft Azure Functions 2019 (MAF1) and 2021 (MAF2).
    
- **Tenant distribution:** Skewed multi-model distributions modeled via a power-law distribution (with an exponent of 0.5), an asymmetric 20%/80% split in two-model tests, and round-robin mapping of Azure function invocations to model instances.
    
- **Prompt distribution:** Fixed input sequence length of 2048 tokens across all benchmark queries.
    
- **Output distribution:** N/A (Evaluated on non-autoregressive encoder/MoE models performing a single forward pass).
    
- **Prefix overlap:** 0% (None; assumes full-weight tuning without shared parameters or prompt-prefix reuse).
    
- **Burst injection:** Slicing production traces into time windows (60 seconds for MAF1; 5,400 seconds for MAF2) and fitting arrivals to a Gamma process parameterized by rate $\lambda$ and $CV$, scaling $CV$ up to 10 to inject traffic bursts.
    
- **Synthetic modifications:** Controlled scaling of traffic burstiness ($CV \in [1, 10]$), arrival rate scaling factors ($0.001\times$ to $100\times$), and synthetic arrival shifts to test robustness against mispredicted traffic
    

---

# 13. Simulation Setup

**Simulator**

- **Simulator:** Custom continuous-time, discrete-event simulator (DES) written in Python.
    
- **Version:** Custom research prototype open-sourced at `[https://github.com/alpa-projects/mms](https://github.com/alpa-projects/mms)`.
    
- **What does it model?**
    
    - Discrete request arrival timestamps, destination model IDs, and latency deadlines.
        
    - Centralized controller request routing using shortest-queue-first dispatching.
        
    - Group-level First-Come-First-Serve (FCFS) queueing dynamics.
        
    - Deterministic GPU execution durations per stage/operator (derived from cluster profiling).
        
    - Predictive admission control / request dropping when latency deadlines cannot be met.
        
- **What does it NOT model?**
    
    - Low-level GPU microarchitectural contention (SM warp scheduling, DRAM memory bus conflicts).
        
    - Dynamic host-to-device weight swapping delays (assumes models reside statically in GPU memory; Clockwork++ baseline was simulated assuming zero swapping overhead).
        
    - Autoregressive multi-step token decoding dynamics and KV cache memory allocation.
        
    - Inter-node physical network packet drops, retransmissions, or variable network jitter.
## Simulated Cluster

| Parameter          | Value                                                                                                                        |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Number of GPUs     | Scaled from 4 up to 128 GPUs (default evaluation sweeps: 8 to 64 GPUs)                                                       |
| GPU Type           | Modeled after NVIDIA Tesla V100 (SXM2)                                                                                       |
| Number of Replicas | Dynamically determined by Algorithm 1 and Algorithm 2                                                                        |
| Model              | BERT-1.3B, BERT-2.7B, BERT-6.7B, BERT-104B; MoE-1.3B, MoE-2.4B, MoE-5.3B                                                     |
| GPU Memory         | 16 GB physical capacity (configured to 13 GB usable budget for weights, reserving ~3 GB for activations and runtime context) |
| Network            | Profiled intra-node NVLink and 25 Gbps inter-node interconnect                                                               |
| CPU                | Host execution overhead assumed negligible relative to deterministic GPU forward pass                                        |

## Simulation Assumptions

- Deep learning inference latency is deterministic and predictable based on ahead-of-time profiling.
- Coarse-grained request arrival distributions over longer timescales (e.g., hours/days) can be approximated from historical traces.
- All model parameters are pinned in GPU accelerator memory (zero runtime swapping overheads).

---

# 14. Real Hardware Setup

## Hardware

| Component      | Configuration                                                                 |
| -------------- | ----------------------------------------------------------------------------- |
| GPU            | NVIDIA Tesla V100 (16 GB)                                                     |
| GPU Memory     | 16 GB per GPU (effective 13 GB budget for model weights)                      |
| Number of GPUs | 64 GPUs total (distributed across 8 nodes)                                    |
| CPU            | Intel Xeon E5-2686 v4 (standard on AWS EC2 `p3.16xlarge` instances)           |
| RAM            | 488 GiB system memory per node (AWS EC2 `p3.16xlarge` instance specification) |
| Storage        | AWS local NVMe / EBS instance storage                                         |
| Network        | 25 Gbps inter-node network; intra-node NVIDIA NVLink (AWS `p3.16xlarge`)      |

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