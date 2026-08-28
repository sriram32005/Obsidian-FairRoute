---
tags:
  - "#llm-serving/scheduling"
---

> **Question** : When should it execute ? 

The scheduler determines the order of execution of requests assigned to a particular instance and resource allocation 

```
Instance (eg: vLLM Instance)
   │
   ▼
Scheduler
   │
   ├── R1
   ├── R2
   ├── R3
   ├── R4
   └── R5
```

```
Scheduling
│
├── FCFS
├── Request-level scheduling
├── Iteration-level scheduling
├── Continuous scheduling
├── Preemption
├── Priority scheduling
├── SLO-aware scheduling
├── Length-aware scheduling
├── Fair scheduling
└── Session-aware scheduling
```

There are multiple levels of scheduling:
1. Cluster Level
2. Instance Level
3. GPU Level