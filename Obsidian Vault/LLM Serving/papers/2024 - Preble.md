---
title: "PREBLE: EFFICIENT DISTRIBUTED PROMPT SCHEDULING FOR LLM SERVING"
authors:
  - Vikranth Srivatsa∗, Zijian He∗, Reyna Abhyankar, Dongming Li, Yiying Zhang
year: "2024"
venue: ""
paper_type: research
status: unread
rating:
url: ""
pdf: "[[2024 - Preble.pdf]]"
tags:
---


# 2024 - Preble

> [!abstract] One-Line Summary  
> Write the entire paper's contribution in **1–2 sentences**.

---

## 1. Paper Metadata

| Field                   | Notes                                  |
| ----------------------- | -------------------------------------- |
| **Authors**             |                                        |
| **Year**                | 3 Oct 2024                             |
| **Venue**               | ICLR 2024                              |
| **Paper Type**          | Scheduling                             |
| **Code Available?**     | Yes - https://github.com/WukLab/preble |
| **Artifact Available?** | Yes                                    |
| **Related System**      | vLLM, SGLang                           |

---

# Problem

## Problem Statement

**What precise problem does this paper solve?**

Recent works Zheng et al. (2023b); Gim et al. (2024); Moore & Li (2024) propose to cache computed key- value (KV) state in GPU memory and reuse the cached KV when a new request sharing a prompt prefix arrives. These works aim to improve the serving performance of LLMs with long and shared prompts in a single GPU setting. 

However, in-production LLM serving systems typically utilize a distributed set of GPUs to serve user requests. Current distributed LLM serving systems are not prompt-cache-aware; they attempt to distribute LLM computation load equally across GPUs to achieve high cluster-level GPU utilization. Yet, this distribution could result in requests with shared prefixes being sent to different GPUs, causing KV computation at all these GPUs that could otherwise be avoided if prefixes are cached and reused on the same GPU. 

On the other hand, a naive solution that always sends requests with shared prefixes to the same GPU would result in imbalanced loads and low overall GPU utilization because the GPU that initially serves a request with a popular prefix will accumulate a huge load of new requests all trying to reuse the calculated prefix KV.

## Why Does This Problem Matter?

Heavy re-computation of long shared prompt in case of distributing load equally within GPU without prompt-cache awareness leading to low efficiency and high latency.
## Why Do Existing Approaches Fail?

Cluster Level Utilization Load Balancing -  Not Prompt Cache Aware - Recomputation of long shared prompt 


---

# System Context

## System Layer

-  Cluster Routing
    
-  Request Scheduling
    
-  Continuous Batching
    
-  KV Cache Management
    
-  GPU Scheduling
    
-  Distributed Inference
    

## Target Workload

5 types of long and shared prompts workload:
- LLM with tool calling Guo et al. (2024)
- LLM as embodied agents in virtual environments Huang et al. (2022)
- LLM for code generation Nijkamp et al. (2023)
- Embed- ded video QA Xiao et al. (2021)
- Long document QA Li et al. (2023a)

Characterstics
	- Prompts are 37× to 2494× longer than generated sequences
	-  85% to 97% tokens in a prompt are shared with other prompts
	- Request often shares prompts with multiple other requests at different amounts

Anaylsed Traces Dataset
	- Azure LLM request trace Patel et al. (2024): end-user traces to have large prompt-to-output ratios

---

# Core Contribution

## Core Insight

>  A distributed LLM request scheduling algorithm called E2 (standing for Exploitation + Exploration) that co-designs model computation load-balancing and prefix-cache sharing. E2 allows requests to exploit (i.e., reuse) computed prompt prefixes on the same GPU but also gives chances for requests with shared prefixes to explore other GPUs.

## Main Contributions

E2 chooses exploitation when the amount of recomputation saved is larger than that of new computation, which happens when the number of shared prefix tokens is larger than the remaining non-shared tokens. Otherwise, E2 chooses exploration. For exploitation, we send the request to the GPU that caches the longest-matched prefix

## What Is Actually Novel?

When E2 decides to explore GPUs, it chooses the GPU with the lightest “load”, using a prompt-aware load definition we propose. This prompt-aware load includes three parts all calculated as GPU computation time. The parts are: 
	1. GPU’s computation load in a recent time window H - measured by the total prefill time and decode time incurred by all the requests in H
	2.  Cost of evicting existing KVs on the GPU to make memory space to run the new request
	3. Cost of running the new request on the GPU

 E2 picks the GPU with the lowest sum of the three parts to explore, which balances loads while accounting for cached prompt behavior.

**Preble**: A distributed LLM serving system that aims to provide high serving throughput and low request average and tail latency for long and shared prompts. Preble consists of a global, request-level scheduler and a per-GPU, iteration-level schedule. Novel Techniques:
	1. Basic E2 algorithm, a prefix is cached at a GPU after its initial assignment until its eviction. The amount of requests sharing it can change over time, which can cause load imbalance. To mitigate this issue, Preble detects load changes and redirects requests from a heavily loaded GPU to a light GPU. If the load hitting a popular prefix increases beyond what a single GPU can handle, Preble automatically scales (autoscales) the prefix by replicating it on multiple GPUs.
	2. A prompt hitting a cached prefix can be treated as decoding-phase computation, while a missed prompt can be treated as prefill-phase computation because of the high prompt-to-decoding token length ratio. Thus, our global scheduler tries to mix the two types of requests on a GPU to balance prefill and decoding computation needs
	3. Unlike existing works that either honor request fairness or maximize prefix matching, Preble aims to achieve high prefix reusing while ensuring fairness, which is important in multi-tenancy environments. We achieve this by assigning priorities to waiting requests based on their prefix cache hit ratio and giving each priority their respective quota of requests to serve.



---

# System Architecture

![[Pasted image 20260831200456.png]]

 A two-level scheduling system
	 1. Global scheduler performs request-level scheduling decisions and orchestrates the overall load balancing across GPUs
	 2. Per-model-instance local scheduler performs iteration-level scheduling for requests assigned to the GPU
Preble scales to at least 70 to 391 GPUs. To offer larger, data-center-level scales, one can deploy several Preble clusters, each having one global scheduler.

Design Benefits:
	1. By having all requests in a cluster go through the global scheduler, we have a centralized place to maintain a global view of cluster load and prompt caching information, both being essential for E2.
	2. By performing coarse-grained, request-level scheduling, a single global scheduler can scale to hundreds of GPUs, avoiding the complexity of maintaining multiple distributed global schedulers for a cluster.
	3. By performing fine-grained, iteration-level scheduling at each GPU, the local scheduler can quickly adapt to GPU resource and request availability changes and make precise decisions.
## E2 Global Scheduler

**Data Structure**: Global prefix trees, implemented as radix trees. 

>[!Radix Trees Implementation]-  
>Each tree has a distinct root storing the shared prefix of all prompts under the tree. When inserting a new request to the tree, we match its tokens from the beginning (i.e., prefix matching) until no match exists, and we insert the remaining tokens as a new leaf node. If no match exists at all, we create a new tree with this request’s prompt as the root node. If an existing tree node only matches partially to the new request (i.e., the prefix of a node matches a sub-sequence of the new request), we split the node into the matched part and the remaining part. For each tree node, we record three pieces of information: 
>	1. Number of tokens in the tree node
>	2. Set of GPUs caching the tree node KVs
>	3. Per-GPU number of requests sharing the tree node in a history window H. 
>When a tree node has no caching GPU and there is no request within the window H in the whole system sharing it, we remove it from the tree.

**Per-request scheduling Policy**: 
![[Pasted image 20260831201542.png]]

E2 unifies three types of costs when calculating the per-GPU load:
	1. Computation cost aggregated across all requests within a time window, 
	2. Recomputation cost needed for evicting memory to run the new request
	3. Computation cost of the new request.
E2 calculates all three costs as GPU computation time and finds the GPU with the lowest sum
*Trick: Instead of profiling the actual computation time, we only maintain token counts at the global scheduler, which largely reduces the system overhead*

### First Cost: Overall GPU Computation Load (L)

![[Pasted image 20260831202149.png]]

**Default time window (H)** = 3 mintues (Analysis showedvarying time lengths doesn't affect the results)

They do not use GPUi’s load at the exact request scheduling time for two reasons: 
	1.  GPU’s load can change between the time of scheduling Rk to the time of running it 
	2.  Placement of a prefix has a longer-term effect than a single load in time because of other requests’ future exploitation of it.

==Therefore for each request Rr in the history, they estimate:==
	==1. Prefill time PTr:  with a regression function using the number of tokens in Rr that do not match any prefixes on GPUi==
	==2. Decoding time DTr:  with another regression function using the average request output length observed on GP Ui in window H==
==The regression functions used in this calculation are captured from offline profiling for each GPU type.==

### Second Cost: Potential cost to free GPU memory so that the new request, Rk, can run.

The more tokens in a sequence that are shared and by more requests, the more costly it is to evict the sequence.

### Third Cost: Actual cost, Pi, to run the new request Rk on GP Ui

which is simply the prefill time of the missed tokens in request Rk. We do not count its decoding time, as it is the same across GPUs, and our goal is to compare the per-GPU load across GPUs.
The total cost of assigning the current request to GP Ui is Li + Mi + Pi and we choose the GPU with the lowest total cost to assign the request to.

### Post Assginment Load Adjustment

With the above algorithm, after the global scheduler assigns a request to a GPU, its prefix lives there until its eviction.This greedy approach works well in cases where the load to a prefix is relatively stable but not otherwise. Proposed two methods of managing load: 
	1. Shift load between GPUs and is applicable when the load surge can be handled by a single GPU in the cluster. The global scheduler maintains a per-GPU load as discussed above. If the most heavily loaded GPU’s load is more than Thbal times higher than the lightest GPU, it shifts load from the former to the latter until their difference is below Thbal
	2. Auto-scale a prefix by replicating it and splitting its subtree by load when we detect that a certain prefix’s request load is still too high (average queueing time doubles over H) even after the above load rebalancing.

### Prefill-Decode Balancing

LLM prefill has a larger compute-to-memory ratio than decoding, causing inefficient GPU resource utilization and perfor- mance degradation. Instead of chunking-prefill (Sarathi) or PD Disaggregation, they propose a new method leveraging prompt sharing features at cluster level. 

Insight, A Request with 
	1. Its entire prompt shared and cached would only perform the decoding phase. Thus, it can be treated as a decoding-phase comput- ing unit
	2. A long prompt not cached and a short output length can be treated as a prefill-phase computing unit
	3.  partially cached prompt can be treated as being between the prefill- and decoding-phase units. Thus, we can balance prefill-decoding by combining requests with more or less prompt sharing instead of or in addition to existing balancing techniques.

when a request is about to be explored, the global scheduler first considers the prefill and de- coding balancing for each GPU. If a GPU is heavily loaded with decoding-phase computing units, the global scheduler directs the current request to it, as a request to be explored will incur recomputation for prompt and is considered a prefill-phase unit. 

*We prioritize this policy over the load-cost comparison (Algorithm 2) because a GPU with heavy decoding has unused computation capacity that we can almost freely use. The global scheduler performs the load-cost comparison if all GPUs have relatively balanced decoding-prefill loads.* 

## Local Scheduler

We run one local scheduler per GPU and schedule requests at the iteration level. Each local scheduler maintains a request wait queue, a prefix (radix) tree, and the number of active requests sharing each prefix tree node

When a new request arrives, the local scheduler matches it to the local prefix tree and updates the tree accordingly. It also inserts the request into the waiting queue. After each model iteration, the local scheduler forms the next batch by selecting waiting requests using a priority-based algorithm.

If a selected request has a long and non-shared prompt, we chunk the prompt similar to Sarathi Agrawal et al. (2023). If the GPU memory is not enough to run the batch, the local scheduler selects a tree node(s) or part of a tree node (if a part is enough) to evict based on the request ==accessing time (LRU) of tree nodes==. The local scheduler then asynchronously informs the global scheduler about the eviction, and the latter processes it in the background.


### Waiting queue request ordering

2023/24 systems used 
	1. FCFS - Ignore prompt sharing -> More recomputation
	2. Prefix Sharing - Ignores fairness -> Resulting starvation

**Proposed Approach: Priority based which considers both fairness and prefix sharing**
1. Create P (a configurable parameter) priority groups and assign a request to the priority group according to its cached token percentage.
2. Then selecting requests to form the next batch, the scheduler proportionally selects requests from each priority group, with the higher priority group getting more requests selected than lower priority ones. 

---
# Workloads and Environments

## Hardware

- Model: 
	- Llama-3 70B
	- Mistral 7B
- GPU Setup:
	- 4 - Nvidia-A6000 
	- 8 - Nvidia-H100 
- Framework: vLLM and SGLang

## Baselines:

Load balancer that sends requests in a round-robin fashion to individual SGLang/vLLM instances (i.e., non-prompt-aware data parallelism).
As round- robin distributes requests evenly, these baselines capture a distributed serving system that balances request loads and then performs prefix sharing within each parallel instance


## Metrics

Three key metrics:
	- Request per second: Which measures serving capacity
	- Average end-to-end request latency ( including scheduling time, queuing time, prefill and decoding time)
	- p99 request latency

Not used:
	- TPOT (Time per output token)
	- TTFT (time to first token)

Reason: Target LLM use has short output lengths, rendering TPOT not as meaningful and TTFT close to the request latency. We consider p99 latency since it is important to control the tail latency in LLM serving as with all other user-facing services

---

# Key Results

Average and p99 latency against increasing requests arriving per second (RPS) of Preble and SGLang on the five workloads, two LLMs, and two GPU environments,

![[Pasted image 20260831210401.png]]

**Key Takeaway**: Preble significantly outperforms the data-parallel SGLang baseline for all settings, as can be seen from Preble’s lower average and p99 latency, especially under higher RPS (or the other way around, for the same latency target, Preble can serve higher RPS). 
Our improvements over SGLang range from 1.5× to 14.5× in terms of average latency and 2× to 10× in p99 latency.

**Inferences:** Bigger improvements of Preble over SGLang on the Toolbench, em-bodied agent, video QA, and LooGLE workloads than the programming workloads. The programming workload has the longest decoding length among all the workloads. 
==As decoding time starts to dominate total request latency, and we do not improve decoding performance, the room for improvement for Preble is smaller.== 
Nonetheless, Preble still achieves 1.56× to 1.8× improvement in average latency and 3× to 4× in p99 latency over SGLang in the programming workload

**Experiment Methodology**: Experiments above use a Poisson request arrival distribution (which is the same as most existing LLM works’ experimental methodology Kwon et al. (2023); Li et al. (2023b)). 

**Azure Trace and Mixed Workloads**
![[Pasted image 20260831210759.png]]

---


# Comparison With FairRoute

## Which Signal Does This Paper Optimize?

| Fairness | Locality | Prediction |
| -------- | -------- | ---------- |
| Yes      | Yes      | No         |

## What Can FairRoute Reuse?

Two layer schedulers
E2 strategy

## What Does FairRoute Need Beyond This?

1. Better ways for estimating prefill and decode time in the global scheduler than simple linear regression
2. Better ways than LRU cache eviction in the local scheduler
3. Handle decoding phase time
## Threat to FairRoute

> Could this paper invalidate the FairRoute research gap or make the proposed contribution trivial?
> Simple algorithms are used for prediction of Prefill and decode time. Might need to read other papers to understand what type of prediction and where is that used compare to the prediction used here.

## How Could This Paper Be Extended to Compete With FairRoute?

By adding better prediction strategies. I need clear idea of prediction used in NexusSched First. 

---

# Baseline Decision

## Should We Implement This?
 
-  Use as conceptual baseline only
### Reason: Two Layer Scheduler Design

### Comparison Priority: Medium

---