# FairRoute — Refined Scope & Experimental Roadmap
### Online + Fairness-Aware + Locality/Prefix-Cache-Aware + Predictive Cluster Routing for Multi-Replica LLM Serving

**BCSE497J Project-I → Project-II transition document**
Team: Sasank V (23BAI1010), Naveen B S (23BAI1069), Sriram S (23BAI1117) · Guide: Dr. Sakthivel V, SCOPE
Prepared as a working document for the experimental-design phase. All external claims below are checked against the 20 papers already in the project's literature base and cross-validated against Consensus (220M+ peer-reviewed/arXiv corpus) and open web sources as of September 2026. Every non-obvious number is attributed to a specific paper — treat this as a living document and re-verify anything older than a few months before it goes into the Review-3 report.

---

## 0. How to use this document

Section 1 restates the project's positioning in sharper, falsifiable terms. Section 2 is a literature audit — it checks whether the "gap" claimed in the Review-2 deck still holds. Sections 3–4 turn the scoring formula into something with precise, measurable terms. Sections 5–11 are the actual experiment plan: metrics, objectives, three parallel testbeds (simulation, single-GPU, 4×A100), datasets, baselines, the combinatorial sweep, and statistical rigor. Section 12 is a risk register. Section 13 is a phased timeline. Section 14 is a single traceability table mapping every design decision to its supporting source, and the References section lists everything cited.

---

## 1. Refined Project Scope

### 1.1 One-line repositioning

> **FairRoute is an online cluster router for multi-replica LLM serving that makes a per-request routing decision using three jointly-scored, continuously-updated signals — fairness debt, prefix/KV-cache locality, and near-future capacity — and that treats the *way these signals are combined* as an open experimental question rather than a fixed design choice.**

The shift from the Review-2 framing is small but important: earlier the project's contribution was "a router that optimizes fairness + locality + prediction jointly." The refined scope makes the *combination function itself* a first-class variable under test, alongside the traditional weighted-sum coefficients (α, β, γ). This directly answers the flag already raised in the project description — the scoring formula was synthesized during slide construction, not derived — by turning it into hypothesis H0 (linear weighted sum) against three alternatives (Section 3).

### 1.2 Formal problem statement

At each request arrival $r$ at time $t$, given a set of candidate replicas $J = \{1, ..., m\}$, FairRoute must choose $j^* = \arg\max_{j \in J} \text{Score}(r, j, t)$ subject to:
- an admission/overload constraint (no replica may be pushed past a safe queue-depth or KV-occupancy bound),
- a per-decision latency budget compatible with the request arrival rate (sub-millisecond to low-double-digit-millisecond, matching the ranges reported for comparable routers — RouteBalance reports ≈32 ms at 12 req/s for a heavier learned-predictor stack, while BalanceRoute's BR-0 targets sub-100 ms decode-step budgets on a 144-NPU cluster).

This is exactly the "online resource allocation with time-coupled constraints" framing used in the *Online Linear Programming for Multi-Objective Routing* paper already in the corpus, which is also the paper with the strongest existing precedent for integrating a novel router into the Vidur simulator — a template worth reusing directly (Section 6).

### 1.3 Non-goals (unchanged from Review-2, restated for clarity)

- **Not** intra-replica scheduling or batching policy — the replica's own scheduler (vLLM / Sarathi-Serve chunked-prefill) is reused unmodified.
- **Not** heterogeneous-replica routing, MoE expert-routing, or formal worst-case fairness guarantees in Project-I. These are explicitly deferred to Project-II (Section 13).
- **Not** a claim that the linear weighted sum is correct — it is now one of four candidate combination forms under test (Section 3.3).

---

## 2. Literature-Grounded Gap Validation (as of September 2026)

The Review-2 deck's central claim — "no existing system scores fairness + locality + prediction jointly" — was re-checked against the project's own 20-paper base plus a fresh Consensus/web sweep. **The gap still holds.** The table below places every closely-related system (anchors plus newly surfaced near-misses) on the three axes.

| System | Fairness | Locality | Prediction | Reported headline result | Source |
|---|:---:|:---:|:---:|---|---|
| **Round-Robin / P2C / JSQ** | ✗ | ✗ | ✗ | Baseline-tier only | Standard load-balancing literature |
| **VTC** (Virtual Token Counter) | ✓ | ✗ | ✗ | Proven 2× tight upper bound on service difference between backlogged clients | Sheng et al., 2023 (arXiv, 121 citations) |
| **Preble** | ✗ (load-aware, not fairness-aware) | ✓ | ✗ | Prefix-aware scheduling; scores reuse against live load | Srivatsa et al., 2024 (arXiv:2407.00023) — confirmed cited by CacheRoute, DualMap, A-Universal-LB-Principle, ELDR, Lodestar, Simple-Is-Better |
| **DLPM / D²LPM** | ✓ | ✓ | ✗ (reactive, current-snapshot only) | Up to 2.87× higher throughput than VTC; up to 7.18× lower per-client latency than the strongest distributed baseline | Cao et al., 2025, *Locality-aware Fair Scheduling in LLM Serving* (arXiv, 16 citations) — **numbers match the project description exactly** |
| **NexusSched** (LENS + PRISM*) | ✗ (no per-tenant fairness term) | partial (via predictive routing) | ✓ | 43% average SLO-attainment improvement; up to 3× throughput speedup in long-context/heterogeneous scenarios | Zhang et al., 2025, *A Predictive and Synergistic Two-Layer Scheduling Framework for LLM Serving* (arXiv) — **numbers match the project description exactly** |
| **CacheRoute** | ✗ | ✓ (periodic warm-set planning) | partial (rate-based planning, not per-step forecast) | 2.3× the strongest of five baselines' SLO-capacity at p99≤3.5s; cache-hit rate 93.2±0.5% vs. 64.1±1.3% cache-blind | Huang Cheng, Meta, 2026 (project file) |
| **DualMap** | ✗ | ✓ | ✗ (reactive SLO-threshold switch, not forecast) | 2.25× effective request capacity at equal TTFT SLO | Yuan et al., ICLR 2026 (project file) |
| **HW-Router** | ✗ | ✗ | ✓ (hardware-aware latency predictor) | 3.4–3.9× lower latency, 46–48 pp higher SLO attainment vs. CARROT/IRT baselines | Kabir et al., UCF, 2026 (project file) |
| **Lodestar** | ✗ | partial (reward predictor implicitly values reuse) | ✓ (online reward predictor) | 1.41× lower avg TTFT; up to 4.38×/4.42× on heterogeneous clusters vs. a prefix-cache+load-aware heuristic | Lim et al., UIUC/ByteDance, 2026 (project file) |
| **RANDOMIZATION_BOOSTS_KV_CACHING** | ✗ | ✓ (theory) | ✗ | First unified theoretical model of the KV-eviction/query-routing trade-off; cites SkyWalker as the applied locality-aware cross-region baseline | Wu, Silwal, Zhang, ICLR 2026 (project file) |
| **Equinox** | ✓ (dual-counter: user + operator) | ✗ | ✓ (MoPE predicts latency/throughput/utilization) | 1.3× throughput, 60% lower TTFT, 13% higher fairness vs. VTC | Wei et al., 2025 (Consensus) — **closest external near-miss; still lacks explicit prefix-locality term** |
| **RouteBalance** | ✗ (quality/cost/latency, not per-tenant share) | ✗ | ✓ (learned TPOT predictor) | Traces the upper region of a quality–cost–throughput frontier on a 13-instance/28-GPU cluster | Da & Kalyvianaki, Cambridge, 2026 (project file) |

*Note: NexusSched's internal cluster-layer component is also named "PRISM," which is a different, unrelated system from the "PRISM (Prefix Reuse Optimization Integrated Scheduling and Memory)" paper already in the project's file base — the two should not be conflated in the Review-3 write-up.*

**Conclusion:** Equinox (2025) is the closest external system — it is the only one with both a fairness mechanism *and* a prediction mechanism — but it still optimizes fairness and prediction without an explicit prefix/KV-locality term, and its "prediction" is about service-quality metrics (latency, throughput, utilization) rather than near-future *cluster capacity* in the NexusSched/PiLLM sense. No system found — inside or outside the project's own corpus — jointly scores all three axes with locality operationalized as prefix/KV-cache affinity. **The three-way gap the project is built around is real and, as of this sweep, still open.**

---

## 3. Refined Joint Objective — From Assumed Formula to Falsifiable Hypothesis

### 3.1 The baseline scoring hypothesis (H0)

$$\text{Score}(r, j) = \alpha \cdot \text{Fairness}(\text{client}(r)) + \beta \cdot \text{Locality}(r, j) + \gamma \cdot \text{Prediction}(j)$$

This is retained as **H0**, the default/null hypothesis — not the final design. It must be benchmarked against alternatives, because the linear weighted-sum has two well-documented, distinct failure modes directly relevant here:

1. **It cannot reach every point on a non-convex Pareto frontier.** This is a settled result in multi-objective optimization theory (Kim & de Weck's adaptive weighted-sum work; the broader scalarization literature) and was recently reconfirmed specifically for the case of *conflicting, jointly-trained objectives* — linear scalarization is provably incapable of finding balanced trade-off solutions in general, even though this contradicts some earlier empirical claims (Hu et al., 2023, *Revisiting Scalarization in Multi-Task Learning: A Theoretical Perspective*, 75 citations).
2. **It requires brittle, workload-specific hyperparameter tuning that degrades when traffic shifts** — this is the *exact* failure mode documented for LLM cluster scheduling specifically, not just multi-objective optimization in the abstract. The OSDI 2026 paper *Simple is Better: Multiplication May Be All You Need for LLM Request Scheduling* (Zhang, Han, Zhang, Wei et al., SJTU/Alibaba — already in the project's corpus) studies exactly the two-objective version of this problem (KV-cache-awareness vs. load balancing) and finds that linear combination needs continual re-tuning as workload changes, while a multiplicative score cancels the hyperparameter out of the comparison entirely and needs no tuning. Their system (LMETRIC) is deployed in production at Alibaba BAILIAN and reduces TTFT by 39–92% and TPOT by 24–51% over vLLM and an in-production scheduler.

Given that a paper already in FairRoute's own literature base makes this exact argument for a two-term version of the same problem, **testing the combination form is not optional — it is the single most literature-supported experimental addition this project can make.**

### 3.2 Term-by-term operationalization

| Term | Definition | Grounded in |
|---|---|---|
| **Fairness**$(c)$ | Deficit counter: target cumulative token-share for client $c$ minus virtual (weighted) tokens actually served, normalized by a rolling window | VTC's virtual-token-counter mechanism (Sheng et al., 2023); D²LPM's deficit generalization to the distributed setting (Cao et al., 2025); classical deficit round-robin fairness in networking |
| **Locality**$(r, j)$ | Longest-common-prefix match length between $r$'s tokenized prompt and replica $j$'s live radix-tree/prefix-cache index, normalized by prompt length | Preble's prefix-history scoring (Srivatsa et al., 2024); PRISM's demand-aware radix tree, DART (project file); CacheRoute's warm-set admission (project file) |
| **Prediction**$(j)$ | Forecast of replica $j$'s near-future free capacity: expected queueing delay + free KV-block budget over a short horizon $H$ | NexusSched's structurally-informed online performance model (Zhang et al., 2025); BalanceRoute's BR-H horizon-discounted lookahead (project file); PiLLM's workload-prediction-driven resource efficiency (project file) |

### 3.3 Combination forms to test (the actual novel experimental axis)

| Form | Formula sketch | Rationale / precedent |
|---|---|---|
| **H0 — Linear (weighted sum)** | $\alpha F + \beta L + \gamma P$ | Default baseline; industry-common per LMETRIC's own characterization of current practice (e.g., Alibaba BAILIAN's prior scheme) |
| **H1 — Multiplicative (log-linear)** | $F^{\alpha} \cdot L^{\beta} \cdot P^{\gamma}$ (or $\exp(\alpha \log F + \beta \log L + \gamma \log P)$ for numerical stability) | Directly modeled on LMETRIC's finding that multiplication removes the tuning burden and is robust to workload drift (Zhang et al., OSDI 2026) |
| **H2 — Lexicographic / gated** | Hard fairness floor as a *filter* (only candidates within a fairness-debt tolerance are considered), then rank survivors by $\beta L + \gamma P$ | Modeled on the "filter-based strategy" LMETRIC itself benchmarks against (filter by load-balance suspicion, then rank by cache hits) — and on the project's own SkyWalker-style overload guard idea (Xia et al., 2025) |
| **H3 — Learned combiner** | Gradient-boosted tree or small MLP over $(F, L, P, \text{context features}) \to \text{score}$, retrained online | Modeled on RouteBalance's per-tier gradient-boosted latency heads (project file) and HW-Router's lightweight latency predictor (project file) |

This turns the original three design tensions in the project description into concrete, testable sub-questions:
1. *Fairness vs. locality* — does a hard deficit floor (H2) preserve more cache-hit rate than a soft linear penalty (H0) at equal fairness?
2. *Locality vs. load balance* — replicate the SkyWalker-style "selective pushing" overload guard (Xia, Mao, Kerney, Jackson, Li, Xing, Shenker, Stoica, 2025, arXiv:2505.24095) as a safety valve on top of all four forms, so the comparison is about the *core score*, not about who avoids hotspotting by accident.
3. *Fairness debt vs. predicted load* — genuinely open; use the adversarial-conflict scenario (Section 10) where a starved client's turn arrives exactly when the only replica with its cached prefix is predicted to be near-saturated, and log which combination form resolves it and how.

---

## 4. Metrics Framework

### 4.1 Fairness metrics

| Metric | Formula | Notes |
|---|---|---|
| **Jain's Fairness Index** | $J(x_1,...,x_n) = \dfrac{(\sum x_i)^2}{n \sum x_i^2}$, range $[1/n, 1]$ | $x_i$ = tokens served per client normalized by entitlement. Directly reported by QUARTZ (0.919 → 0.952 improvement on a mixed workload, 14B model — project file) and CacheCast (maintains >0.95 across tenants) — use the same convention for direct comparability |
| **Max service-difference bound** | $\max_i,j \lvert S_i - S_j\rvert$ for two backlogged clients | The metric VTC's 2× bound is stated in terms of — needed to check whether FairRoute preserves or improves on VTC's guarantee |
| **Starvation rate** | Fraction of clients whose realized share falls below X% of entitlement over a rolling window | Complements Jain's index, which can mask tail starvation |

### 4.2 Locality / prefix-cache metrics

| Metric | Formula | Notes |
|---|---|---|
| **Served KV-cache hit rate** | Fraction of prefill tokens served from cache vs. recomputed | CacheRoute reports 93.2±0.5% vs. 64.1±1.3% cache-blind (project file) — use as a calibration anchor |
| **Prefix-match ratio** | Matched-prefix length / total prompt length, averaged per request | Locality term's own operationalization (Section 3.2) |
| **KV-recompute savings (FLOPs or wall-time)** | Compute saved vs. a no-cache baseline | Ties locality directly to cost/energy framing used by the *Universal Load Balancing Principle* paper (project file) |

### 4.3 Prediction-quality metrics

| Metric | Formula | Notes |
|---|---|---|
| **Forecast error (MAPE / RMSE)** | On predicted queueing delay and free-KV-budget vs. realized | Standard forecasting metrics; NexusSched and PiLLM both implicitly require this for their reported gains to be attributable to prediction quality, not just architecture |
| **Calibration** | Reliability diagram / ECE on predicted-vs-realized capacity buckets | Needed because H3 (learned combiner) is otherwise unfalsifiable if predictions are systematically biased |
| **Lead-time sensitivity** | Metric quality as horizon $H$ grows | Directly probes BalanceRoute's BR-H's own reported trade-off between lookahead length and per-step compute budget (project file) |

### 4.4 System-level SLA metrics (the ultimate outcome variables)

| Metric | Notes |
|---|---|
| **TTFT P50/P99** | Standard across nearly every paper surveyed |
| **TPOT P50/P99** | Standard |
| **End-to-end latency P50/P99** | Standard |
| **Throughput (QPS, tokens/s)** | Standard |
| **SLO attainment %** | Fraction of requests meeting a joint TTFT+TPOT target — the primary metric NexusSched, HW-Router, and QLM all optimize for |
| **Goodput** | Tokens delivered within SLO, not just tokens delivered — the metric JITServe and SLOs-Serve use to avoid rewarding wasted work on requests that will miss SLO anyway |

### 4.5 Composite metrics

| Metric | Notes |
|---|---|
| **Pareto hypervolume** across (fairness, locality, prediction-accuracy, SLO-attainment) | Lets the four combination forms (H0–H3) be compared as *trade-off surfaces*, not single numbers — directly answers the "can weighted-sum reach the same frontier as the alternatives" question from Section 3.1 |
| **Regret vs. a clairvoyant oracle** | Route every request with perfect hindsight of arrivals/lengths, compute the gap — same style of oracle bound used by PACO ("stays within 27% of a clairvoyant oracle's cost") |

---

## 5. Objectives to Optimize

Framed as a **constrained multi-objective online decision problem**, following the bid-price / shadow-price framing in the *Online Linear Programming for Multi-Objective Routing* paper (already in the corpus, ICML 2026):

- **Primary objective:** maximize SLO attainment / goodput, subject to a fairness floor (Jain's index ≥ a target, e.g., 0.9) — this ordering matches how QUARTZ, HW-Router, and NexusSched all report results (SLA metrics as headline, fairness as a maintained property).
- **Secondary objective:** maximize served KV-cache hit rate at the achieved fairness/SLO operating point.
- **Tertiary objective:** minimize forecast error and its downstream effect on mis-routing (a diagnostic objective, not a deployment target).
- **Hard constraint:** overload guard — no replica may exceed a safe queue-depth / KV-occupancy threshold, enforced identically across all four combination forms so the comparison isolates the *scoring* decision, not the *safety valve*.

---

## 6. Experimental Setup — Simulation Track

### 6.1 Simulator choice and rationale

| Simulator | Role | Why |
|---|---|---|
| **Vidur** (Agrawal et al., 2024, arXiv:2405.05465) | **Primary simulator** for the full α/β/γ × combination-form sweep | Already the de facto standard *inside this exact literature base* — used directly by DualMap, the Online-LP paper (which integrates a custom router into Vidur and open-sources the integration), FairnessAware-Chunked-Prefill, and Simple-Is-Better ("VIDUR... the state-of-the-art LLM instance simulator"). Using it gives FairRoute's numbers a shared, comparable footing with four other papers already in the project's own base, and there is a published precedent (Online-LP) for exactly the kind of custom-router integration FairRoute needs. |
| **LLMServingSim / LLMServingSim 2.0** (Cho et al., KAIST, IISWC 2024 / ISPASS 2026) | **Secondary, hardware-detail cross-check** — not the primary sweep engine | Explicitly designed to fix two gaps Vidur-style simulators can have: it models KV-cache capacity and fragmentation in detail (via vLLM-style demand paging) and captures the dynamic, autoregressive nature of serving workloads. Reported accuracy: <14.7% error vs. real GPU-based serving, at 91.5× simulation speed. Use it for a **targeted hardware-sensitivity sub-study** (Section 10) rather than the main sweep, since RouteBalance's own literature review (project file) notes that Vidur-style simulators need re-integration whenever the backend engine's batching logic changes — a real cost worth paying once, not repeatedly across the full grid. |
| **ASTRA-Sim** — recommended **not** as a primary tool | Reference/context only | ASTRA-Sim is a scale-out AI-systems simulator built primarily for collective-communication and network-topology modeling in distributed *training*. LLMServingSim's own comparison against it is explicit: ASTRA-Sim's memory model is simple and lacks capacity/fragmentation constraints, which matters for LLM inference precisely because KV-cache memory is the resource FairRoute is scoring against. If a later phase adds multi-node interconnect effects (e.g., cross-region routing, following SkyWalker's setting), ASTRA-Sim or its descendants (e.g., ATLAHS, which reports outperforming ASTRA-Sim on both accuracy and runtime for realistic workloads) become relevant — but not for the Project-I routing-layer scope. |

### 6.2 Simulation configuration matrix

- **Cluster sizes:** 2, 4, 8, 16 homogeneous replicas (matches the project's stated 2–4 GPU Project-I scope at the low end, and stress-tests scalability at the high end the way BalanceRoute tested 16/48/96/144-NPU tiers).
- **Models:** at least one small (≈7–8B) and one mid (≈13–14B) dense model, matching PRISM's own 4B/13B evaluation split (project file) so locality-metric percentage-point gains are comparable.
- **Router integration point:** plug FairRoute's scorer into Vidur's global-scheduler hook, following the Online-LP paper's own open-sourced integration pattern ("Online-Linear-Programming-for-Vidur").

### 6.3 Sweep protocol

- **Phase A — signal isolation:** run each of Fairness-only, Locality-only, Prediction-only routing in Vidur to establish single-axis baselines before any combination is tested (exactly as already planned in the project description — this order is preserved because it avoids confounding).
- **Phase B — pairwise combinations:** F+L, F+P, L+P, still under H0 (linear) to characterize each two-way trade-off before adding the third term.
- **Phase C — full joint sweep:** for each of H0–H3 (Section 3.3), sweep the simplex $\{(\alpha,\beta,\gamma) : \alpha+\beta+\gamma=1, \alpha,\beta,\gamma \geq 0\}$ at a resolution of 0.25 (15 grid points) under both steady and adversarial traffic (Section 10). This is a simulation-only grid — full-resolution grids are not repeated on real hardware (Section 6.4/9).
- **Refinement:** once Phase C identifies promising regions, run a finer Bayesian-optimization or coordinate-descent pass around them (this is cheap in simulation and expensive on hardware, which is why it belongs here).

---

## 7. Experimental Setup — Real Hardware

### 7.1 Small-scale: single RTX 4060 (8 GB)

**Purpose:** functional/integration validation and *routing-overhead* micro-benchmarking — **not** a throughput or latency headline-number tier. Treat every real-hardware small-scale result as a correctness check on the implementation, not as evidence about FairRoute's performance at scale.

- **Feasible models:** a 7B-class model fits only with 4-bit quantization (e.g., AWQ/GPTQ). A concrete community data point: Llama-2-7B-Chat at Q4_K_M quantization uses ≈7.1 GB of the card's 8.0 GB, leaving under 1 GB of headroom — a "tight fit" with roughly 46 tok/s decode throughput on this class of card. For anything with more comfortable headroom, use a 1–3B dense model in bf16/fp16.
- **Emulating a "mini-cluster" on one GPU:** run 2–3 separate vLLM (or SGLang) server processes, each capped via `--gpu-memory-utilization` to a fraction of the 8 GB, to get multiple independently-schedulable "replicas" for FairRoute to route across. This is purely an engineering/debugging convenience — it shares the same physical memory bus and SM count, so **cross-replica load-balancing numbers from this setup are not meaningful and must not be reported as results**, only used to confirm the router's logic (scoring, fairness bookkeeping, cache-index lookups) behaves correctly end-to-end.
- **What this tier is good for:** (a) validating the Fairness Tracker's deficit-counter bookkeeping under real request streams, (b) validating the Locality Estimator's radix-tree/prefix-index lookups against a real vLLM/SGLang prefix cache, (c) measuring the router's own per-decision CPU-side latency overhead (compare against RouteBalance's reported ≈32 ms at 12 req/s for a similarly-featured stack), (d) smoke-testing the Prediction Engine's feature pipeline before it ever sees a multi-node cluster.
- **What NOT to claim from this tier:** no SLO-capacity numbers, no cache-hit-rate percentages presented as generalizable, no fairness-index values compared against the literature's cluster-scale numbers (consumer GPUs also have no ECC/ NVLink and are subject to thermal/clock throttling not present in datacenter parts, both real confounds).

### 7.2 Large-scale: 4× A100

This matches the project's own stated Project-I ceiling ("2–4 GPUs") and sits comfortably inside the range the literature base itself uses for credible cluster claims — Cluster-Route-Escalate uses 2×A100, Robust-KV-Cache-Management uses 8×A100, and the *Universal Load Balancing Principle* paper scales its theory validation to 256×A100 (project files) — so 4×A100 is a legitimate, literature-consistent "small real cluster" tier, not a toy.

- **Topology:** 4 homogeneous replicas, one A100 (80 GB preferred, 40 GB workable with a smaller model) per replica — keeps this phase strictly inside the stated homogeneous-replica scope; save any 3+1 heterogeneous split for Project-II.
- **Model matrix:** Llama-3.1-8B-Instruct or Qwen2.5-7B/14B as the primary model (aligns with RouteBalance's own Qwen2.5 family choice and CacheRoute's use of comparable dense models at smaller parameter counts for controlled experiments — project files), run unquantized or in fp8 depending on available memory headroom.
- **Serving engine:** vLLM with Sarathi-Serve-style chunked-prefill scheduling as already planned. Where a baseline specifically requires radix-tree-based prefix caching semantics (e.g., replicating Preble-/D²LPM-style behavior faithfully), consider an SGLang-backed variant for that baseline only, and note this in the write-up as CacheRoute does for its own common-harness reimplementations of Preble/DualMap/CHWBL ("compare routing behavior under matched hardware but do not reproduce the published systems end to end" — project file).
- **Load generation:** vLLM's own `benchmark_serving.py`, extended with a custom multi-tenant wrapper that tags each synthetic client with an ID and an injected request-rate profile; cross-check with NVIDIA GenAI-Perf or LLMPerf for a second, independently-implemented measurement of TTFT/TPOT/ITL to catch instrumentation bugs.
- **Telemetry:** per-request structured logs (tenant ID, arrival time, prompt length, matched-prefix length, assigned replica, queue depth and KV-occupancy at decision time, TTFT, TPOT) plus a Prometheus/Grafana dashboard for live queue-depth and KV-occupancy — the same signal set the Fairness Tracker / Locality Estimator / Prediction Engine need is also exactly what the evaluation needs, so instrument once.

---

## 8. Workload & Datasets

| Dataset | What it contributes | Used by (precedent in corpus) |
|---|---|---|
| **ShareGPT** | Realistic multi-turn conversational prompt/response length distributions; long-standing standard for prefix-sharing studies | Robust-KV-Cache-Management, Simple-Is-Better (retrofitted VIDUR sim), Equinox, DualMap, and effectively the whole field |
| **WildChat** | Larger, more diverse real-world chat distribution than ShareGPT, useful for stress-testing prefix diversity/skew assumptions | Cited across the corpus as a companion trace to ShareGPT |
| **Azure LLM Inference Trace (AzurePublicDataset / "Azure-2024")** | Realistic arrival-time bursts and diurnal patterns from a real production trace | BalanceRoute deploys against this exact trace on a 144-NPU cluster (project file); PACO's evaluation is also built on Microsoft Azure production traces |
| **BurstGPT** | Explicit burst-injection trace designed to stress queueing/admission behavior | Robust-KV-Cache-Management combines BurstGPT + Azure + ShareGPT precisely to separate burstiness effects from length-distribution effects (project file) — worth replicating that same three-way split for FairRoute |

**Synthetic skew injection (on top of the above):** per-tenant Zipfian request-rate skew (a small number of "heavy" clients responsible for a disproportionate share of traffic) to stress the Fairness Tracker, plus **adversarial-conflict scenarios** constructed so that a starved client's next request always targets the replica the Prediction Engine currently favors for load reasons — this is the concrete testbed for open design tension #3 (Section 3.3).

---

## 9. Baselines — a tiered ladder

| Tier | Baseline | Axis covered |
|---|---|---|
| **0 — no signal** | Random, Round-Robin, Power-of-Two-Choices, Join-Shortest-Queue | none (sanity floor) |
| **1 — single-axis SOTA** | VTC (fairness); Preble-style prefix/live-load scoring (locality); a Lodestar/HW-Router-style learned latency predictor (prediction) | one each |
| **2 — two-axis SOTA** | D²LPM-style (fairness+locality); NexusSched/PRISM-style (prediction+cluster-state); DualMap-style dual-hash-ring (locality+load-balance); CacheRoute-style periodic warm-set planning (locality+load) | two each |
| **3 — proposed** | FairRoute under H0 (linear), H1 (multiplicative), H2 (gated), H3 (learned) | all three |

**Reimplementation-fidelity caveat (carried over directly from the project's own literature):** CacheRoute's own methodology explicitly flags that its Preble/DualMap/CHWBL baselines are "common-harness reimplementations... not the original codebases" that "compare routing behavior under matched hardware but do not reproduce the published systems end to end" (project file). FairRoute's Tier-1/Tier-2 baselines should carry the identical caveat in the Review-3 report, and — following CacheRoute's own stated recommendation — **any decision to enable a locality/affinity mechanism in a real deployment context should be gated behind a shadow-replay validation, not enabled from workload statistics alone.**

---

## 10. Experimental Combinations / Ablation Matrix

| Axis | Values | Where it runs |
|---|---|---|
| Signal isolation | F-only, L-only, P-only | Simulation (Phase A) |
| Pairwise | F+L, F+P, L+P | Simulation (Phase B) |
| Combination form | H0 linear, H1 multiplicative, H2 gated, H3 learned | Simulation (full grid, Phase C) → hardware (Pareto-frontier points only) |
| Weight simplex | 15-point grid at 0.25 resolution, refined via Bayesian optimization near promising regions | Simulation only |
| Traffic scenario | Steady, bursty (BurstGPT-style), skewed-tenant (Zipfian), adversarial-conflict (Section 8) | Both tiers |
| Cluster scale | 2/4/8/16 (simulation), 1-GPU virtual mini-cluster + 4×A100 (hardware) | Both tiers |
| Replica homogeneity | Homogeneous only (Project-I scope) | Both tiers — heterogeneous deferred |

**Combinatorial-cost management:** a full grid across all axes above is intractable to run exhaustively on real hardware — this is why the plan is *simulate broadly, validate narrowly*: run the full Phase A–C grid in Vidur, identify the Pareto-optimal (combination form, weight) configurations per traffic scenario, and only carry those forward to the 4×A100 tier. This mirrors how nearly every paper in the corpus handles the same cost problem (e.g., the Online-LP paper explicitly ran its main sweep in Vidur "due to lack of GPU resources," project file) — it is standard practice, not a shortcut being introduced here.

---

## 11. Statistical Rigor & Threats to Validity

- **Seeds:** report 3–5 paired seeds minimum per configuration, in the $X\pm Y$ format used throughout the corpus (e.g., CacheRoute's "176±11 QPS," project file), not single-run point estimates.
- **Shadow-replay gate:** before treating any hardware-tier result as deployment-relevant, replay real traffic against the candidate configuration in shadow mode first — directly adopting CacheRoute's own stated recommendation (project file).
- **Simulator-to-hardware gap:** budget for a known discrepancy — LLMServingSim itself reports up to 14.7% error against real GPU-based serving even as its own validation baseline (project file/web-confirmed), so treat any simulation-only result as directional until cross-checked on the 4×A100 tier for at least the top 2–3 configurations per scenario.
- **Consumer-GPU confounds:** thermal throttling, lack of ECC memory, and absence of NVLink on the RTX 4060 mean single-GPU numbers are not comparable to datacenter-GPU numbers even for the same model — stated explicitly in Section 7.1 and repeated here because it is the single easiest mistake to make when writing up results.
- **Baseline-fidelity threat:** common-harness reimplementations of Preble/D²LPM/DualMap-style baselines may not match the original systems' tuning exactly (Section 9) — report this as a limitation, not silently.

---

## 12. Risk Register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Weighted-sum (H0) turns out adequate, undercutting the "combination form matters" narrative | Medium | Frame this as a valid, reportable negative result — LMETRIC's own paper shows the opposite is not guaranteed either way, so either outcome is publishable |
| Vidur re-integration breaks when the vLLM/Sarathi-Serve backend version changes | Medium | Budget explicit re-integration time each time the backend is upgraded; freeze backend version during the core sweep (Phase C) |
| Single-GPU tier results get mistakenly reported as generalizable | Medium-High if not flagged | Explicit reporting rule already stated in Section 7.1 — enforce at write-up review |
| 4×A100 access is time-limited/shared | High (typical for academic clusters) | Simulation-first strategy (Section 10) minimizes wasted hardware time; only run pre-selected Pareto-frontier configs on the real cluster |
| Adversarial-conflict scenario is under-specified and produces noisy, unreproducible results | Medium | Fix the scenario generator's random seed and document the exact construction (client-starvation-then-hot-prefix trigger) before running any comparison |
| Fairness debt vs. predicted load open question (tension #3) has no literature answer to fall back on | Certain — it's explicitly unresolved | Treat as the project's primary novel-contribution claim; report it as an empirical finding either way rather than assuming an answer in advance |

---

## 13. Phased Roadmap

| Phase | Focus | Primary testbed |
|---|---|---|
| **0** | Instrumentation-only: build Fairness Tracker, Locality Estimator, Prediction Engine as independently testable modules; validate each against synthetic traces before any routing decision uses them | Single-GPU (7.1) |
| **1** | Signal isolation + pairwise sweeps (Sections 6.3 Phase A/B) | Vidur simulation |
| **2** | Full joint sweep across H0–H3 × weight simplex × traffic scenario × cluster scale | Vidur simulation, LLMServingSim cross-check on a subset |
| **3** | Small-scale hardware smoke tests: confirm router logic and overhead on real vLLM/SGLang processes | Single-GPU (7.1) |
| **4** | Large-scale validation: carry forward only Pareto-optimal configurations per scenario | 4×A100 (7.2) |
| **5** | Shadow-replay gating check, write-up, sensitivity analysis as a primary deliverable (per the project description's own stated post-deck direction) | — |

---

## 14. Traceability Table — Design Choice → Source

| Design choice | Supporting source |
|---|---|
| Fairness gap is real and quantified | Sheng et al. 2023 (VTC, 2× bound); Cao et al. 2025 (D²LPM, 2.87×/7.18× numbers) |
| Prediction gap is real and quantified | Zhang et al. 2025 (NexusSched, 43%/3× numbers) |
| Combination form is an open, testable question | Zhang, Han, Zhang, Wei et al., OSDI 2026 (LMETRIC/Simple-Is-Better, project file); Hu et al. 2023 (scalarization theory) |
| Overload-guard design | Xia et al. 2025 (SkyWalker, arXiv:2505.24095) |
| Vidur as primary simulator | Agrawal et al. 2024; used directly by 5 papers in the project's own corpus |
| LLMServingSim as HW-detail cross-check, not primary | Cho et al., IISWC 2024 / ISPASS 2026; explicit KV-cache-memory-modeling advantage over ASTRA-Sim |
| ASTRA-Sim excluded from primary role | LLMServingSim's own published comparison (simple memory model, no capacity/fragmentation modeling) |
| 4×A100 is a literature-consistent cluster tier | Cluster-Route-Escalate (2×A100), Robust-KV-Cache-Mgmt (8×A100), Universal-LB-Principle (256×A100) — all project files |
| RTX 4060 8GB feasibility numbers | Community benchmark data (Llama-2-7B-Chat Q4_K_M ≈7.1GB/8.0GB, ≈46 tok/s) |
| Dataset combination (ShareGPT+WildChat+Azure+BurstGPT) | Robust-KV-Cache-Management's own precedent combining exactly BurstGPT+Azure+ShareGPT (project file) |
| Shadow-replay gating recommendation | CacheRoute's own stated deployment recommendation (project file) |
| Jain's Index as the fairness headline metric | QUARTZ (project file, 0.919→0.952) and CacheCast (Consensus, >0.95 maintained) |

---

## References

**Already in the FairRoute project literature base (see `/mnt/project/`):**
CacheRoute (Huang Cheng, Meta, 2026) · Cluster, Route, Escalate (Moslem, Kacmajor et al., 2026) · Fairness-Aware and Latency-Controllable Scheduling for Chunked-Prefill LLM Serving (Liu, Wang, Xu, Li, 2026) · HW-Router (Kabir, Xue, Zheng, Lou, UCF, 2026) · RouteBalance (Da & Kalyvianaki, Cambridge, 2026) · Randomization Boosts KV Caching, Learning Balances Query Load (Wu, Silwal, Zhang, ICLR 2026) · PRISM: Fast Online LLM Serving via Scheduling-Memory Co-design (2026) · DualMap (Yuan, Zuo, Wang, Chen, Tan, Yu, ICLR 2026) · EWSJF (Sidik, Levi, Kampeas, Huawei, 2026) · QUARTZ (Liu, Zheng, Kong, Zhao, ACL Findings 2026) · Lodestar (Lim, Zhao, Godfrey, Shan, Xu, Xie, UIUC/ByteDance, 2026) · Online Linear Programming for Multi-Objective Routing in LLM Serving (Chen, Ye, Zhou, ICML 2026) · ELDR (Choi, Cho, Xiong, Yang, Kwon, Cheng, KAIST/MSR, 2026) · Robust KV Cache Management for LLM Serving under Output Token Length Uncertainty (Cheng, Do, Nguyen, ASU, 2026) · BanaServe (He, Xu, Wu et al., Alibaba/CAS, 2025) · FLARE (Fu, Zhong, Huang, Lu, ACL 2026) · Tackling the Data-Parallel Load Balancing Bottleneck in LLM Serving / BalanceRoute (Bu, Lyu, Chen, Song, Liang, Gurung, Fan, Ye, Zhou, HKUST/Huawei, 2026) · Simple is Better: Multiplication May Be All You Need for LLM Request Scheduling / LMETRIC (Zhang, Han, Zhang, Wei, Shen, Fang, Yu, Zhou, Chen, SJTU/Alibaba, OSDI 2026) · A Universal Load Balancing Principle and Its Application to LLM Serving (Chen, Bu, Song, Lu, Ye, Zhou, HKUST/NUDT/Stanford, 2026) · PiLLM (Fan, Bai, Gong, Wang, Fan, ShanghaiTech/SenseTime, 2026)

**Validated externally via Consensus / web search during this roadmap revision:**
- Sheng, Y. et al. (2023). *Fairness in Serving Large Language Models* (VTC). arXiv.
- Cao, S. et al. (2025). *Locality-aware Fair Scheduling in LLM Serving* (DLPM / D²LPM). arXiv.
- Srivatsa, V. et al. (2024). *Preble: Efficient Distributed Prompt Scheduling for LLM Serving*. arXiv:2407.00023.
- Zhang, Y. et al. (2025). *A Predictive and Synergistic Two-Layer Scheduling Framework for LLM Serving* (NexusSched). arXiv.
- Xia, T., Mao, Z., Kerney, J., Jackson, E. J., Li, Z., Xing, J., Shenker, S., Stoica, I. (2025). *SkyWalker: A Locality-Aware Cross-Region Load Balancer for LLM Inference*. arXiv:2505.24095.
- Wei, Z. et al. (2025). *Equinox: Holistic Fair Scheduling in Serving Large Language Models*. arXiv.
- Sun, J. et al. (2026). *CacheCast: Learning-Guided KV Cache Placement for Multi-Tenant Cloud LLM Serving*. IoTAAI 2026.
- Hu, Y. et al. (2023). *Revisiting Scalarization in Multi-Task Learning: A Theoretical Perspective*. arXiv.
- Cho, J., Kim, M., Choi, H., Heo, G., Park, J. (2024). *LLMServingSim: A HW/SW Co-Simulation Infrastructure for LLM Inference Serving at Scale*. IISWC 2024 / arXiv:2408.05499.
- Cho, J., Choi, H., Heo, G., Park, J. (2026). *LLMServingSim 2.0: A Unified Simulator for Heterogeneous and Disaggregated LLM Serving Infrastructure*. ISPASS 2026. (github.com/casys-kaist/LLMServingSim)
- Agrawal, A., Kedia, N., Mohan, J., Panwar, A., Kwatra, N., Gulavani, B., Ramjee, R., Tumanov, A. (2024). *Vidur: A Large-Scale Simulation Framework for LLM Inference*. arXiv:2405.05465.
- Teng, D. (2025). *PACO: Predictive Auto-Configuration for SLO-Constrained LLM Inference Serving*.
- Community benchmark data on RTX 4060 8GB quantized-model feasibility (willitrunai.com, cross-checked against general vLLM/llama.cpp community reports).

*All numeric claims above are stated as reported by their respective sources; none have been independently re-derived by this document's authors. Re-verify any figure before it is quoted in the Review-3 report or thesis, and note that several source papers carry a 2026 date on preprints that may still be under peer review.*
