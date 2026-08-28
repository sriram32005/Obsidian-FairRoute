# LLM Serving System Execution Path

```
                    ┌──────────────────────┐
                    │      WORKLOAD        │
                    │  Requests + Traces   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
		            │         API          │
                    │      Admission       │  
                    |    (Not Important)   |
                    └──────────┬───────────┘
		                       │
                               ▼
                    ┌──────────────────────┐
		            │       REQUESTS       │
                    │       Routing        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      SCHEDULER       │
                    │  Which request when? │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       BATCHER        │
                    │ Which requests/tokens│
                    │ execute together?    │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴───────────┐
                    ▼                      ▼
                PREFILL                  DECODE
                    │                      │
                    └──────────┬───────────┘
                               ▼
                    ┌──────────────────────┐
                    │      KV CACHE        │
                    │ Memory for state     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ PARALLEL EXECUTION   │
                    │ TP / PP / DP / EP    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ COMMUNICATION        │
                    │ NVLink / PCIe / RDMA │
                    │ NCCL / Collectives   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │    GPU EXECUTION     │
                    │ Kernels / SM / HBM   │
                    │ Tensor Cores / GEMM  │
                    └──────────────────────┘
```

# Dependency Graph

```
                    WORKLOAD
                       │
                       ▼
                    ROUTING
                       │
                       ▼
                  SCHEDULING
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          BATCHING            KV CACHE
             │                   │
             └─────────┬─────────┘
                       ▼
                 PREFILL/DECODE
                       │
                       ▼
                  PARALLELISM
                       │
                       ▼
                COMMUNICATION
                       │
                       ▼
                   GPU WORK
                       │
                       ▼
                   METRICS
```

# LLM Serving Stack

[[L0 - Fundamentals of LLMs]]
[[L1 - Workloads]]
[[L2 - Requests Router]]
[[L3 - Requests Scheduler]]
[[L4 - Requests Batcher]]
[[L5 - Request Execution]]
[[L6 - KV Cache Management]]
[[L7 - Parallel Execution]]
[[L8 - Communication]]
[[L9 - GPU Execution]]
[[L10 - System Level Optimizations]]
[[L11 - Disaggregated Serving]]
[[L12 - Multi-Tenancy]]
[[L13 - Simulation & Execution]]


# Literature Review 

## Research Scope

```
                    ┌──────────────────────┐
                    │      WORKLOAD        │
                    │  Requests + Traces   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
		            │       REQUESTS       │
                    │       Routing        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      SCHEDULER       │
                    │  Which request when? │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       BATCHER        │
                    │ Which requests/tokens│
                    │ execute together?    │
                    └──────────┬───────────┘
```

[[LLM Serving/papers/Index|Literature Review - Papers]]
