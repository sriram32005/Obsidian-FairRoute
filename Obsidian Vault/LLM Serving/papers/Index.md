# Research Scope

```
                    ┌──────────────────────┐
                    │      WORKLOAD        │
                    │  Requests + Traces   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
		            │       REQUESTS       │
                    │       Routing        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      SCHEDULER       │
                    │  Which request when? │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       BATCHER        │
                    │ Which requests/tokens│
                    │ execute together?    │
                    └──────────┬───────────┘
```

# Foundational Papers:

