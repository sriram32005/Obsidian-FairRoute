---
tags:
  - "#llm-serving/fundamentals"
---

```
LLM
│
├── Transformer
│   ├── Embedding
│   ├── Attention
│   ├── FFN / MLP
│   └── LayerNorm / RMSNorm
│
├── Autoregressive generation
│
├── Tokenization
│
├── Context length
│
├── Model parameters
│
└── KV Cache
```

Need to understand

```
Prompt
  ↓
Tokens
  ↓
Embedding
  ↓
Transformer layers
  ↓
Logits
  ↓
Next token
  ↓
Feed token back
  ↓
Next token
```

## Prefill : Process the entire input prompt

```
Input:
[T1 T2 T3 T4 T5]

      ↓

Transformer

      ↓

KV Cache
```

## Decode: Generate one token at a time

```
T6
 ↓
T7
 ↓
T8
 ↓
T9
```

