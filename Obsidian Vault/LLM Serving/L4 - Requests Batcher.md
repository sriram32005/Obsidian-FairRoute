---
tags:
  - "#llm-serving/batching"
---

>**Question**: Which requests/tokens to execute together ? 

```
R1 ──────┐
R2 ──────┼──→ Batch
R3 ──────┤
R4 ──────┘
```

```
Batching
│
├── Static batching
├── Dynamic batching
├── Continuous batching
├── Iteration-level batching
├── Chunked prefill
├── Decode-maximal batching
├── SplitFuse
├── Staggered batching
└── Layered prefill
```

