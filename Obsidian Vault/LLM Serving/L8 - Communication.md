---
tags:
  - "#llm-serving/communication"
---

Parallelism creates communication. Parallelism is the algorithmic organization. 
Communication is the physical mechanism required to make that organization work.

For example:
```
GPU0              GPU1
 │                  │
 │      Tensor      │
 │     Parallel     │
 │                  │
 └──── AllReduce ───┘
```

```
Communication Heirarchy

GPU
 │
 ├── Registers
 ├── Shared Memory
 ├── L2
 └── HBM
      │
      ▼
   PCIe
      │
      ▼
   NVLink
      │
      ▼
   NVSwitch
      │
      ▼
   NIC
      │
      ▼
   RDMA
      │
      ▼
 Other Node
```

Communcation Primitives:

```
NCCL (NVIDIA Collective Communcations Library)
│
├── AllReduce
├── AllGather
├── ReduceScatter
├── Broadcast
└── AllToAll
```
