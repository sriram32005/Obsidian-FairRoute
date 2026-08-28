---
tags:
  - "#llm-serving/disaggregation"
---

Instead of the same GPU do both Prefill and Decode, We can have:

```
                 Requests
                    │
             ┌──────┴──────┐
             ▼             ▼
        Prefill Pool    Decode Pool
        GPU GPU GPU     GPU GPU GPU
             │             ▲
             │             │
             └── KV Cache ─┘
                  Transfer
```

Concepts:

```
Disaggregation
│
├── Prefill / Decode
├── KV transfer
├── KV storage
├── KV routing
├── GPU specialization
├── Heterogeneous GPUs
└── Rack-scale memory
```
