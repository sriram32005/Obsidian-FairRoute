Prefill / Decode Execution

```
                 Request
                    │
             ┌──────┴──────┐
             ▼             ▼
          PREFILL        DECODE
             │             │
             │             │
       Many tokens      One token
       in parallel      at a time
             │             │
             └──────┬──────┘
                    ▼
                GPU work
```

Prefill: Compute Intensive
Decode: Memory-bandwidth / KV-Cache Intensive
