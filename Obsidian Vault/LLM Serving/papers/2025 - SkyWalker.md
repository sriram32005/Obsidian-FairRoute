---
title: "SkyWalker: A Locality-Aware Cross-Region Load Balancer for LLM Inferenc"
authors:
  - Tian Xia† Ziming Mao† Jamison Kerney† Ethan J. Jackson† Zhifei Li§ Jiarong Xing†¶ Scott Shenker†⋄ Ion Stoica†
year: 9 NOV 2025
venue: EUROSYS ’26
paper_type: research
status: read
rating:
url: ""
pdf: ""
tags:
---


# 2025 - SkyWalker

> [!abstract] One-Line Summary  
> Simplest algorithm + Simple Implementation + Nothing to takeaway to fairroute

---

# Problem

## Problem Statement

Serving Large Language Models (LLMs) efficiently in multi-region setups remains a challenge. Due to cost and GPU availability concerns, providers typically deploy LLMs in multiple regions using instance with long-term commitments, like reserved instances or on-premise clusters, which are often underutilized due to their region-local traffic handling and diurnal traffic variance.

The reserved instances or on-premise clusters are inflexible—providers cannot increase or decrease capacity within a region as demand shifts. As a result, providers must allocate enough instances in each region to meet peak demand, which can result in high and often wasted costs. 

The challenge is further exacerbated in multi-region deploy- ments, where providers must provision for peak load in every region independently. This leads to resource fragmentation and underutilization during off-peak hours.

Providers must either provision for peak demand for all regions or pay for the flexibility of on-demand instances and take the risk of GPU unavailability.

---

# Solution Introduction

we suggest relaxing the regional rigidity of current approaches. Instead of attempting to match regional capacity to regional demand, providers make reservations for peak global demand and partition those reservations across the regions closest to their users.

Following this approach, when a region is overloaded, it can offload requests to other regions with excess capacity. By reserving for global peak demand and enabling cross-region traffic handling, a system can improve GPU utilization and reduce overall serving costs.

Unfortunately, this cannot be achieved by simply deploy- ing centralized load balancers in a single zone , as this introduces high latency due to cross-region communication and creates both a performance bottleneck and a single point of failure.

To enable cross-region traffic handling without sacrific- ing performance, we design SkyWalker, a cross-region load balancer for LLM inference.

![[Pasted image 20260903194519.png]]

As shown in Figure 1(c), SkyWalker deploys at least one load balancer in each re- gion as the first point of contact for requests, ensuring low- latency and avoiding centralized bottlenecks. These regional load balancers collaboratively coordinate traffic across re- gions to handle load imbalances. 

Two key challenges in multi-region load balancing:
1. Key-Value (KV Cache) awarness -> Soln: Two routing algorithms : consistent hashing + multi-region prefix trie
2. LLM Inference Load unpredictability -> Soln:  selective pushing algorithm

---

# 4. Core Contribution

## Core Insight

> What is the **one central idea** that distinguishes this paper?
> 
> Serving Large Language Models (LLMs) efficiently in multi-region setups remains a challenge. Due to cost and GPU availability concerns, providers typically deploy LLMs in multiple regions using instance with long-term commitments, like reserved instances or on-premise clusters, which are often underutilized due to their region-local traffic handling and diurnal traffic variance.
> The reserved instances or on-premise clusters are inflexible—providers cannot increase or decrease capacity within a region as demand shifts. As a result, providers must allocate enough instances in each region to meet peak demand, which can result in high and often wasted costs. 

## Main Contributions

- Identifying the need for cross-region traffic handling to aggregate diurnal patterns across multiple geographical regions and reduce global serving cost. 
- Proposing two mechanisms to provide effective cross-region routing: prefix-aware routing to improve cache locality and performance, and selective pushing based on pending re- quests at each replica to reduce load imbalance. 
- A comprehensive evaluation for SkyWalker, compared to existing production and research systems across a variety of workloads. 
- An open-source system SkyWalker to stimulate further research on cross-region load balancing for LLMs.


---

# System Architecture

SkyWalker deploys load balancers in multiple regions as the first point of contact for local requests, and introduces a cross-region traffic handler that coordinates traffic between regional load balancers to mitigate cross-region load imbalance.

It preserves the benefits of prefix sharing by supporting prefix-aware routing in two ways:
1. A simple yet effective policy based on consistent hashing that requires minimal changes to existing load balancers
2. A prefix-aware routing using partial prefix tree snapshots maintained at each load balancer

To handle the LLM load unpredictability, SkyWalker introduces a **novel selective pushing mechanism** that balances load based on pending requests at each replica

## Cross-Region Traffic Handling

Two layer cross-region routing. The key idea is to coordinate cross-region traffic between load balancers, rather than directly between replicas. Each load balancer either routes requests to local replicas or forwards them to other load balancers in remote regions, which then make the final placement decisions within their region.

## Multi-Region Prefix-Aware Routing

*Observation:* 
1.   *The average prefix similarity within the same user is significantly higher than that across different users (2.47-7.60× more).*
2.  *There is still some degree of cross-user prefix similarity, and the relative ratio between within-user and cross-user prefix similarity is workload-dependent (2.47× for ChatBot Arena and 7.60× for WildChat).*

Two Solutions:
### 1. SkyWalker-CH

- It uses consistent hashing on user-provided keys (e.g., user ID, session ID) and routes a user request to a corresponding replica. 
- It is implicitly prefix-aware: requests from the same user tend to share similar prefixes (e.g., context, chat history) and consistent hashing will map them to the same replica.
- It adopts a ring hash scheme where each virtual node on the hash rings assigned to a replica and each replica can have multiple virtual nodes, allowing balanced key distribution across replicas.

It perfoms consistent hashing at both layers:
1. Load balancer routes requests to other balancers based on consistent hashing
2. Each balancer applies consistent hashing to assign the request to one of its managed replicas

Since SkyWalker-CH focuses only on within-user prefix similarity, there are cases where SkyWalker-CH falls short of being optimal. They are: 
1. Cross-User Prefix Sharing
2. Bursty Request
3. Heterogeneous User Program

### 2. Skywalker with regional snapshot

-  It is explicitly prefix-aware: in this design, each load balancer maintains prefix trees to keep an approximate view of prefix information on the load balancing targets. 
- Between load balancers, the targets are remote load balancers, and between the load balancer and the replica, the targets are local replicas managed by that load balancer.
- The prefix tree is a logical trie augmented with metadata to track active load balancing targets at each node. Each node stores a set of active targets associated with the prefix formed by the path from the root to that node.
- To bound memory usage, enforces a configurable maximum tree size and evicts entries when the tree exceeds this limit, starting with the earliest in- serted records.
- SkyWalker filter targets based on whether it is available to serve requests and pick the available target with the longest matching prefix.

Each load balancer maintains two prefix trees
1. one for local replicas it manage
2. one for a partial view (snapshot) of other load balancers in other regions

Regional snapshots do not strictly record all prefixes reside in replicas of remote regions.
Instead, it is an approximation of prefixes that are possible to be utilized by local region forwarding to that remote region.

## Selective Pushing to Mitigate Load Imbalance

Existing approaches: 
1. Blind Pushing: Route each request to a replica immediately upon arrival
2. Selective Pushing: Requests are temporarily queued at the load balancer and sent only to replicas that meet certain conditions
	1. By Limiting outstanding requests: load balancer selectively pushes to a replica only when the number of of outstanding requests for that replica is less than a fixed threshold
	2. By Checking pending requests: A pending request is a request that has not been scheduled to the continuous batch yet, which indicates that the current batch is full and cannot admit more requests, as constrained by GPU memory. A background heartbeat probe is periodically sent to replicas to obtain their pending queue size (Listing 1, line 3-8). If a replica has no pending request, it is ready to serve more requests.
Novel Approach: 
**Selective Pushing and cross-region routing**: 
- Each load balancer tracks the number of replicas it manages with full continuous batches and periodically synchronizes this state with peer load balancers through heartbeat messages
- If a load balancer has at least one non-full replica and its request queue size does not exceed a small buffer, it is considered available to accept additional requests.When at least one local replica is not full, requests are always routed locally to maximize responsiveness.
-  If all local replicas are full, the system considers remote regions and forwards requests only to regions with available replicas and short load balancer queue 
- When multiple candidates are avail- able, either among local replicas or remote load balancers, the system breaks ties using the consistent hashing key (for SkyWalker-CH) or the prefix hit rate (for SkyWalker) to select a candidate with more prefix sharing,

Algorithm
![[Pasted image 20260903203348.png]]

---

# Experimental Setup

Probing frequency = 100ms

Model = meta-llama/Llama-3.1-8B- Instruct
GPU =  L4 GPU with up to 12 replicas 

Metrics:
- TTFT (p10,p25,p75,p90)
- E2E (p10,p25,p75,p90)

Baselines compared: 
- GKE Gateway 
- Round Robin
- Least Loaded
- Consistent Hasing (CH)
- SGLang Router

Workloads: 
- ChatBot Arena
- WildChat
- Tree of Thoughts (ToT)
- Mixed Tree
---

# Key Results

## Main Quantitative Results

![[Pasted image 20260903203956.png]]

## Main Takeaway

- We show the service throughput of multi-turn conversation datasets (ChatBot Arena and Wild- Chat) in Figrue 8a, 8b. Both variants of SkyWalker im- prove service throughput by 1.12-1.2× compared to single load balancer solutions.
- 1.12-2.06× higher throughput and 1.74-6.30× lower latency compared to existing load balancers, while reducing total serving cost by 25%

---
