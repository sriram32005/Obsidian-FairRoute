---
tags:
  - "#llm-serving/gpu"
---

All the way down to the GPU

```
GPU
│
├── SMs
├── CUDA cores
├── Tensor Cores
├── Registers
├── Shared Memory
├── L2 Cache
└── HBM
```

Operations:

```
Transformer
│
├── GEMM
├── GEMV
├── Attention
├── Softmax
├── LayerNorm
├── RoPE
└── Sampling
```

