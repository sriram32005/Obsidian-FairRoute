---
tags:
  - "#llm-serving/kv-cache"
---

```
Transformer
     │
     ▼
Attention
     │
     ▼
K,V tensors
     │
     ▼
KV Cache
```

Instead of recomputing previous tokens, use the cached values

``` 
Without KV cache:

T1 T2 T3 T4
 ↓  ↓  ↓  ↓
recompute everything
```

```
With KV Cache: 

T1 T2 T3 T4
 │  │  │  │
 └──┴──┴──┴──→ KV Cache

Next token
    ↓
reuse KV
```

During the prefill phase, We will calculate Q,K and V values for all the input tokens but during the decode phase we we will querying the old tokens with the new token Query Value, So only the new token will have Q,K,V values and we only require the old tokens K and V values not their Q values. 
So only K,V values of the processed tokens are stored. And for each new token generated, Only its K,V values are appended to the cache and the Q value is dropped as it is not required further.

```
KV CACHE
│
├── Allocation
├── Memory management
├── Fragmentation
├── PagedAttention
├── Block management
├── Prefix caching
├── Sharing
├── Eviction
├── Compression
├── Offloading
├── CPU storage
├── Distributed KV
└── KV migration
```

