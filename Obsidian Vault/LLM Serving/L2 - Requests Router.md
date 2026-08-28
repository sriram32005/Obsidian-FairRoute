---
tags:
  - "#llm-serving/routing"
---

> **Question**: Where should the request execute ? 

```
                    Router
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    Instance 0     Instance 1     Instance 2
     GPU 0,1        GPU 2,3        GPU 4,5
```

Routing Policies:

```
Routing
│
├── Round Robin
├── Random
├── Least Loaded
├── Load-aware
├── Length-aware
├── KV-cache-aware
├── Prefix-aware
├── Session-aware
|── Migration
|- etc...
```
