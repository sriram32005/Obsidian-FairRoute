---
tags:
  - "#llm-serving/parallelism"
---

> **Question**: How do we distribute the model computation across GPUs ? 

```
                 LLM
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
       GPU0      GPU1      GPU2
```

```
PARALLELISM
│
├── Tensor Parallelism
│
├── Pipeline Parallelism
│
├── Data Parallelism
│
├── Expert Parallelism
│
└── Sequence Parallelism
```

Combinations also exists:

```
TP
TP + DP
TP + PP
TP + EP
DP + EP
TP + EP
TP + PP + DP
```

This becomes essential for:
- large models
- MoE
- Multi-GPU Serving
- Distributed Serving
