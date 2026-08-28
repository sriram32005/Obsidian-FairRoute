---
tags:
  - "#llm-serving/multi-tenancy"
---

>**Question**: How should scarce GPU resources be divided among competing tenants ?

Add multiple users/models/workloads

```
                Cluster
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    Tenant A    Tenant B    Tenant C
```

Concepts:

```
Multi-tenancy
│
├── Isolation
├── Fairness
├── Priority
├── SLOs
├── Resource quotas
├── Multi-model serving
├── Multi-LoRA serving
└── Cost allocation
```