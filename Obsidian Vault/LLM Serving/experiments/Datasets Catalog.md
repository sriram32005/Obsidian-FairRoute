# FairRoute — Complete Dataset Catalog

> Compiled from the [[Experimental Roadmap]] and all 38 PDF papers in the project's literature base.
> Last updated: September 10, 2026

---

#  My findings 😎

| #   | Dataset                   | Notes (useful columns)                                                                                                                                                                                             |
| --- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | ShareGPT                  | Conversations<br><br>https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/blob/main/README.md (yet to go through the dataset)<br><br>https://huggingface.co/datasets/shibing624/sharegpt_gpt4 |
| 2   | LMSYS-Chat-1M             | Conversations<br><br>https://huggingface.co/datasets/lmsys/lmsys-chat-1m<br>                                                                                                                                       |
| 3   | Azure LLM Inference Trace | Content and LLM tokens<br><br>https://github.com/Azure/AzurePublicDataset/blob/master/AzureLLMInferenceDataset2023.md                                                                                              |
| 4   | BurstGPT                  | conversation with requst and response tokens count<br><br>https://github.com/HPMLL/BurstGPT/blob/main/README.md                                                                                                    |
| 5   | WildChat-1M               | conversation with ip-hash (maybe useful in fairness) and http headers<br><br>https://huggingface.co/datasets/allenai/WildChat-1M                                                                                   |
| 6   | Mooncake                  | conversation, tool-agent, synthetic<br><br>https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/README.md                                                                                               |
I think this is more than enough (all links at the end of this file)

---

## Quick Navigation

- Summary Matrix
- **Primary Datasets** (Sections 1–5): ShareGPT, LMSYS-Chat-1M, Azure LLM Trace, BurstGPT, WildChat
- **Secondary Datasets** (Sections 6–12): Alpaca, LongBench, Arena-Hard, ToolBench, MMLU, LooGLE, Mooncake Traces
- **Specialty Datasets** (Sections 13–21): MultiHop-RAG, τ-bench, UltraChat, MixInstruct, AIME, TeleQnA, ALFWorld, APPS, NExT-QA
- Synthetic Workload Methods
- Recommended Stack for FairRoute
- Download Links

---

## Summary Matrix

| # | Dataset | Type | Primary Use | Papers Using It | FairRoute Priority |
|---|---------|------|-------------|:-:|:---:|
| 1 | ShareGPT | Real conversations | Prompt/response distributions, prefix sharing | **35+** | ✅ **Core** |
| 2 | LMSYS-Chat-1M | Real conversations | Larger/more diverse chat distribution | ~20+ | ✅ **Core** |
| 3 | Azure LLM Inference Trace | Production trace | Arrival patterns, diurnal cycles, bursts | ~15+ | ✅ **Core** |
| 4 | BurstGPT | LLM service trace | Bursty traffic stress testing | 5 | ✅ **Core** |
| 5 | WildChat-1M | Real ChatGPT interactions | Diverse, multilingual stress test | 3 | ✅ **Core** |
| 6 | Alpaca | Instruction-following | Short, single-turn workload contrast | 4 | ⚡ Secondary |
| 7 | LongBench | Long-context benchmark | KV-cache pressure testing | 3 | ⚡ Secondary |
| 8 | Arena-Hard-Auto | Hard benchmark prompts | Compute-heavy request stress test | 3 | ⚡ Secondary |
| 9 | ToolBench | Tool-use conversations | Very high prefix sharing (tool schemas) | 2 | ⚡ Secondary |
| 10 | MMLU | Multi-task knowledge QA | Structured few-shot prefix sharing | 2 | ⚡ Secondary |
| 11 | LooGLE | Long-context understanding | Extreme long-prefix scenarios | 3 | ⚡ Secondary |
| 12 | Mooncake Traces | Production LLM traces | Real prefix-sharing patterns (Conversation + Tool&Agent) | 3 | ⚡ Secondary |
| 13 | MultiHop-RAG | RAG benchmark | Prefix reuse in RAG workloads | 1 | 🔶 Optional |
| 14 | τ-bench | Tool-agent interactions | Agent-mode prefix locality | 1 | 🔶 Optional |
| 15 | UltraChat | Instructional conversations | Multi-turn benchmark diversity | 1 | 🔶 Optional |
| 16 | MixInstruct | Instruction ensemble | Short instruction diversity + routing eval | 1 | 🔶 Optional |
| 17 | AIME | Math competition | Quality-aware routing test | 1 | 🔶 Optional |
| 18 | TeleQnA | Telecom domain QA | Domain-specific routing | 1 | 🔶 Optional |
| 19 | ALFWorld | Embodied agent tasks | Agent-style prefix sharing | 1 | 🔶 Optional |
| 20 | APPS | Program generation | Code prefix sharing patterns | 1 | 🔶 Optional |
| 21 | NExT-QA | Video QA | Extreme prompt-to-decode ratio | 1 | 🔶 Optional |
| 22 | Azure Functions Trace (2019/2021) | Serverless traces | Bursty arrival patterns (non-LLM) | 1 | 🔶 Optional |
| 23 | MT-Bench | Multi-turn benchmark | Quality evaluation (80 conversations) | 1 | 🔶 Reference |
| 24 | LMSYS Chatbot Arena | Live platform | ELO ratings, model quality scores | Referenced | 🔶 Reference |

---

## Primary Datasets (Core Stack)

### 1. ShareGPT

> [!IMPORTANT]
> **THE most widely-used dataset in the LLM serving literature.** Used by virtually every paper in the corpus. The Experimental Roadmap lists it as a primary dataset.

| Field | Details |
|-------|---------|
| **Full Name** | ShareGPT / ShareGPT_Vicuna_unfiltered |
| **Type** | Real-world multi-turn conversational dataset |
| **Download** | 🔗 https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered |
| **Alt Mirror** | 🔗 https://huggingface.co/datasets/shibing624/sharegpt_gpt4 (GPT-4 subset) |
| **Format** | JSON — each entry is a multi-turn conversation with user/assistant turns |

#### Parameters Provided
| Parameter | Value | Source |
|-----------|-------|--------|
| Average input tokens | ~215–306 tokens | EWSJF, Llumnix |
| Average output tokens | ~217–500 tokens | EWSJF, Llumnix |
| Mean output length | 265 tokens (P90: 486, P99: 768) | Robust KV Cache |
| Llumnix stats | In: P50=74, P80=348, P95=1484, P99=3388 / Out: P50=487, P80=781, P95=988, P99=1234 | Llumnix |
| Total responses | ~368,000 assistant responses | Robust KV Cache |

#### What It Contributes to FairRoute
- **Prefix/KV-cache locality** — multi-turn structure creates natural prefix sharing
- **Direct comparability** — used by virtually every baseline (VTC, Preble, D²LPM, CacheRoute, LMETRIC, etc.)
- **Realistic length distributions** — balanced input/output for general workload testing
- **Multi-tenant simulation** — can assign conversations to different "clients" for fairness testing

#### Papers Using It
VTC, Preble, Lluminix, D²LPM, NexusSched, Past-Future, SkyWalker, kLPM, Equinox, BanaServe, FairBatching, CacheRoute, DualMap, ELDR, EWSJF, PRISM, PiLLM, QUARTZ, Robust KV Cache, RouteBalance, LMETRIC, Universal LB, HW-Router, LAPS, Lodestar, Online LP, BalanceRoute, Fairness-Aware Scheduling, Geometry-Aware Scheduling, General Non-Clairvoyant, Efficiency & Cost, Randomization Boosts KV, P2P Inference

---

### 2. LMSYS-Chat-1M

| Field | Details |
|-------|---------|
| **Full Name** | LMSYS-Chat-1M |
| **Type** | Large-scale real-world LLM conversation dataset |
| **Size** | **1,000,000 conversations** from 25 different LLMs |
| **Source** | Collected from LMSYS Chatbot Arena and Vicuna demo |
| **Download** | 🔗 https://huggingface.co/datasets/lmsys/lmsys-chat-1m |
| **Paper** | arXiv:2309.11998 |
| **Format** | JSON/Parquet |

#### Parameters Provided
- **1M conversations** — ~5× larger than ShareGPT for better statistical coverage
- **25 different LLMs** — captures diverse response styles and lengths
- **Multi-turn conversations** — similar prefix-sharing potential as ShareGPT
- **Topic diversity** — broader than ShareGPT due to scale
- **Turn 1 (LAPS)**: ~63% of prompts < 256 tokens; Turn >1: ~81% < 256 tokens

#### What It Contributes to FairRoute
- **Generalization check** — verify that results from ShareGPT hold on a larger, more diverse distribution
- **Robustness validation** — different user population than ShareGPT
- **Scale** — 1M conversations provides stronger statistical grounding

#### Papers Using It
CacheRoute, ELDR, EWSJF, PRISM, PiLLM, QUARTZ, HW-Router, LAPS, Lodestar, Online LP, BalanceRoute, BanaServe, Equinox, D²LPM, NexusSched, Past-Future, SkyWalker, kLPM, RouteBalance, LMETRIC, Universal LB, Preble

---

### 3. Azure LLM Inference Trace

> [!IMPORTANT]
> **The only publicly available production-scale LLM serving trace.** Essential for realistic arrival-time modeling. The Roadmap names it "Azure-2024."

| Field | Details |
|-------|---------|
| **Full Name** | Azure LLM Inference Trace / AzurePublicDataset |
| **Type** | Production LLM inference trace from Microsoft Azure |
| **Download** | 🔗 https://github.com/Azure/AzurePublicDataset |
| **Specific file** | `AzureLLMInferenceTrace_conv.csv` (conversation split) |
| **Size** | 44 million requests over 7 days (full); 10K filtered subset common in papers |
| **Format** | CSV |

#### Parameters Provided
| Parameter | Value | Source |
|-----------|-------|--------|
| Full dataset | 44M requests, 7 days | DynamoLLM / Robust KV |
| Azure-Conv mean output | 117 tokens (P90: 398) | Robust KV Cache |
| Azure-Code mean output | 23 tokens (P90: 49) | Robust KV Cache |
| Filtered subset (output>1K) | 10K reqs, mean prompt 4652, mean output 1052 | BalanceRoute |
| Chat arrival rate | ~5 req/s | Preble |
| Programming arrival rate | ~7 req/s | Preble |
| Chat inter-arrival time | Mean 118ms (range: 2μs–217s) | Preble |

#### Sub-Workloads
- **Conversation split** — multi-turn chat interactions
- **Code split** — code generation requests (shorter outputs)

#### What It Contributes to FairRoute
- **Realistic arrival-time bursts and diurnal patterns** — critical for the Prediction Engine (P term)
- **Production-grade traffic model** — validates against real deployment patterns
- **Non-stationary load** — day/night cycles test adaptivity of all four combination forms (H0–H3)
- **Recommended three-way split** — combine with ShareGPT + BurstGPT per Robust KV Cache methodology

#### Papers Using It
BalanceRoute, Robust KV Cache, NexusSched, SkyWalker, Equinox, BanaServe, FairBatching, Adaptively Robust, HW-Router, Lodestar, QUARTZ, RouteBalance, Universal LB, BalanceRoute, Lluminix, Efficiency & Cost, Geometry-Aware, General Non-Clairvoyant, Preble

---

### 4. BurstGPT

> [!TIP]
> The Roadmap recommends combining BurstGPT + Azure + ShareGPT to separate burstiness effects from length-distribution effects — a methodology from the Robust KV Cache Management paper.

| Field | Details |
|-------|---------|
| **Full Name** | BurstGPT |
| **Type** | Real-world LLM service traces from Azure OpenAI |
| **Size** | **1.4 million requests** collected over 61 days (ChatGPT + GPT-4) |
| **Download** | 🔗 https://github.com/HPMLL/BurstGPT |
| **Paper** | arXiv:2401.17644 |
| **Format** | CSV / JSON |

#### Parameters Provided
| Parameter | Value | Source |
|-----------|-------|--------|
| Total entries | ~1.4M requests (122K commonly used subset) | PiLLM |
| Collection period | 61 days | PiLLM |
| Mean output tokens | 125 (P90: 276, P99: 1586) | Robust KV Cache |
| Mean input tokens | ~1029 (in the 122K subset) | EWSJF |
| Mean output tokens (122K) | ~73 | EWSJF |
| Input 90th percentile | Mean=1792, Std=367 | PiLLM |
| Input 99th percentile | Mean=3395, Std=719 | PiLLM |
| Llumnix GPT4-Conv stats | In: P50=582, P80=1427, P95=2345, P99=3549 / Out: P50=243, P80=434, P95=669, P99=964 | Llumnix |

#### What It Contributes to FairRoute
- **Explicit burst injection** — stresses queueing/admission behavior and the overload guard
- **Skewed I/O ratio** — long inputs, short outputs create different KV-cache pressure than ShareGPT
- **Production-grade realism** — collected from actual Azure OpenAI services
- **Three-way separation** — combined with Azure + ShareGPT, isolates burstiness from length effects

#### Papers Using It
Robust KV Cache, EWSJF, PiLLM, Lluminix (GPT4-Conv split)

---

### 5. WildChat (WildChat-1M)

| Field | Details |
|-------|---------|
| **Full Name** | WildChat-1M |
| **Type** | Large-scale real-world ChatGPT interaction dataset |
| **Size** | ~1 million conversations |
| **Source** | Real user interactions with ChatGPT (GPT-3.5 and GPT-4), collected by Allen AI |
| **Download** | 🔗 https://huggingface.co/datasets/allenai/WildChat-1M |
| **Paper** | arXiv:2405.01470 |
| **Format** | JSON/Parquet |

#### Parameters Provided
- **Multi-turn conversations** — real user/assistant exchanges
- **68 languages** — much broader linguistic diversity than ShareGPT
- **Language distribution (ELDR)**: English 47.6%, Chinese 27.8%, Russian 9.2%, French 2.4%, Portuguese 2.1%, Spanish 1.8%, Other 9.2%
- Top 2 languages (EN+ZH) account for ~75% of requests

#### What It Contributes to FairRoute
- The Roadmap cites it for "stress-testing prefix diversity/skew assumptions"
- **Multilingual diversity** — tests whether prefix-sharing patterns hold across languages
- **Language-based routing** — natural test for MoE expert-locality routing (ELDR uses it for this)

#### Papers Using It
FLARE, BalanceRoute, ELDR (14,000-prompt subset)

---

## Secondary Datasets

### 6. Alpaca (Stanford)

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/tatsu-lab/stanford_alpaca |
| **Type** | Instruction-following dataset generated by GPT-4 |
| **Format** | JSON — single-turn instruction-response pairs |

#### What It Contributes
- **Short, single-turn** workload — contrasts with ShareGPT's multi-turn pattern
- Tests FairRoute when there is **minimal prefix-sharing opportunity**
- Papers: Efficiency & Cost, LAPS, Past-Future, Geometry-Aware, Online LP

---

### 7. LongBench

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/THUDM/LongBench |
| **Type** | Multi-task long-context benchmark (6 categories: summarization, multi-doc QA, few-shot, code, synthetic tasks) |
| **Input lengths** | Many tasks exceed 10K+ tokens |

#### What It Contributes
- Tests behavior under **extreme prefix lengths** → heavy KV-cache pressure
- Validates the Locality Estimator and Prediction Engine at long-context scale
- Papers: NexusSched, Fairness-Aware Scheduling, HW-Router (English tasks)

---

### 8. Arena-Hard-Auto (v0.1)

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/lm-sys/arena-hard-auto |
| **Size** | 500 challenging user prompts from Chatbot Arena |

#### What It Contributes
- **Compute-heavy requests** → stress-tests the overload guard
- Useful for quality-aware routing experiments (H3 learned combiner)
- Papers: Cluster Route Escalate, Fairness-Aware Scheduling, PiLLM

---

### 9. ToolBench

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/OpenBMB/ToolBench |
| **Type** | Tool-use evaluation (16,000+ APIs) |
| **Prefix sharing** | 85% ± 13% of prompt is shared prefix |

#### Key Stats (from Preble)
| Parameter | Value |
|-----------|-------|
| Prompt Length | 1835 ± 742 tokens |
| Output Length | 43 ± 16 tokens |
| Shared Prefix | 85% ± 13% |
| Key Portion | 76% ± 16% |
| Requests Sharing Key | 39 ± 64 |

#### What It Contributes
- **Extremely high prefix sharing** — ideal for testing DART and the Locality Estimator
- Growing real-world use case (tool-augmented LLMs / agents)
- Papers: Preble, PRISM

---

### 10. MMLU

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://huggingface.co/datasets/cais/mmlu |
| **Type** | 57-subject knowledge benchmark (multiple-choice) |

#### What It Contributes
- Very high prefix sharing from **shared few-shot prompts** within each subject
- Tests structured, repetitive workload patterns
- Papers: Preble, ELDR (Professional Medicine subset)

---

### 11. LooGLE

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://huggingface.co/datasets/bigainlp/LooGLE |
| **Alt** | 🔗 https://github.com/bigai-nlco/LooGLE |
| **Type** | Long-document QA (776 documents, 6400+ questions) |

#### Key Stats (from Preble)
| Parameter | Value |
|-----------|-------|
| Prompt Length | 23,474 ± 6,105 tokens |
| Output Length | 16 ± 9.9 tokens |
| Shared Prefix | 91% ± 24% |
| Requests Sharing Key | 18 ± 8.6 |

#### What It Contributes
- **Extreme long-prefix scenarios** — prompts averaging 23K tokens
- Tests KV-cache capacity limits and extreme prefix locality
- Papers: ELDR, Preble, Lodestar (via Mooncake Synthetic), Randomization Boosts KV

---

### 12. Mooncake Traces

> [!NOTE]
> Real production traces from Kimi AI (Moonshot AI), released with the Mooncake paper. Uniquely valuable because they provide **real prefix-sharing patterns** from production.

| Field | Details |
|-------|---------|
| **Source** | Mooncake (Qin et al., FAST 2025) |
| **Download** | 🔗 https://github.com/kvcache-ai/Mooncake |
| **Type** | Production LLM serving traces |

#### Three Sub-Traces

| Trace | Avg Input | Avg Output | Prefix Cache Ratio | Prefix Reuse Distance | Key Characteristic |
|-------|-----------|------------|--------------------|-----------------------|-------------------|
| **Conversation** | 12,035 tokens | 343 tokens | 40% | 733 requests between reuses | Moderate reuse, long intervals |
| **Tool&Agent** | 8,596 tokens | 182 tokens | 59% | 306 requests between reuses | High prefix sharing, frequent tool calls |
| **Synthetic** | 15,300 tokens | 149 tokens | 42% | 409 requests between reuses | Mix of ShareGPT + LeVal + LooGLE |

#### What It Contributes
- **Real prefix-sharing patterns** from production — not synthetic
- **Tool&Agent trace** has 76% of requests sharing ≥50% of prefix → ideal for Locality Estimator testing
- Documented prefix skew with specific "whale" tool prompts concentrating traffic
- Papers: DualMap, Lodestar, PiLLM

---

## Specialty Datasets (Optional — for specific experiments)

### 13. MultiHop-RAG

| Field | Details |
|-------|---------|
| **Download** | arXiv:2401.15391 |
| **Type** | Multi-hop retrieval-augmented generation benchmark |
| **Used by** | PRISM |
| **Key params** | 981 unique passages; prompts 768–1430 tokens; k∈{5,7,10,15} retrieved chunks |

Useful for testing RAG-style prefix reuse with shared evidence passages.

---

### 14. τ-bench (tau-bench)

| Field | Details |
|-------|---------|
| **Source** | ICLR 2025 |
| **Used by** | PRISM (as AGENTPREFIX trace) |
| **Key params** | 2,048 requests; Poisson 30–50 QPS; max prefill 40,960 tokens |

Tool-agent-user interaction benchmark with deterministic tool observations as reusable prefixes.

---

### 15. UltraChat

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://aclanthology.org/2023.emnlp-main.183/ |
| **Used by** | Randomization Boosts KV |
| **Key params** | 128 clients; 2–8 conversation rounds; 1024-token user inputs |

Multi-turn instructional conversations for KV-cache routing evaluation.

---

### 16. MixInstruct (from LLM-Blender)

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/yuchenlin/LLM-Blender |
| **Used by** | HW-Router |
| **Key params** | 6,000 prompts sampled; short instruction-style queries (<256 tokens) |

Short instruction prompts for inducing concurrency in router evaluation.

---

### 17. AIME (American Invitational Mathematics Examination)

| Field | Details |
|-------|---------|
| **Source** | Public competition benchmark (1983–2024) |
| **Used by** | Cluster, Route, Escalate |
| **Key params** | 921 train (1983–2023), 30 test (2024); max output 40,960 tokens (extended CoT); 3 clusters |

Mathematical reasoning benchmark for quality-aware routing.

---

### 18. TeleQnA

| Field | Details |
|-------|---------|
| **Source** | Maatouk et al. (2025), IEEE Network |
| **Used by** | Cluster, Route, Escalate |
| **Key params** | 9,000 train / 1,000 test; telecom domain multiple-choice QA; avg 40 tokens output |

Domain-specific benchmark for specialized routing evaluation.

---

### 19. ALFWorld

| Field | Details |
|-------|---------|
| **Source** | ICLR 2021 (Shridhar et al.) |
| **Used by** | Preble |
| **Key params** | 7,500 requests; prompt 2285±471 tokens; output 16±13 tokens; 97%±14% shared prefix |

Embodied agent workload with extremely high prefix sharing (97%).

---

### 20. APPS (Program Generation)

| Field | Details |
|-------|---------|
| **Source** | NeurIPS Datasets & Benchmarks 2021 (Hendrycks et al.) |
| **Used by** | Preble |
| **Key params** | Prompt 3871±1656 tokens; output 190±343 tokens; 97%±7.4% shared prefix; 126±2157 requests sharing key |

Competitive programming with parallel candidate code generations sharing prefixes.

---

### 21. NExT-QA (Video QA)

| Field | Details |
|-------|---------|
| **Source** | CVPR 2021 (Xiao et al.) |
| **Used by** | Preble |
| **Key params** | 8,500 questions; prompt 9865±5976 tokens; output 4±1.5 tokens; ~2500× prompt-to-decode ratio |

Extreme prompt-to-decode ratio — nearly all compute is in prefill.

---

### 22. Azure Functions Traces (2019/2021)

| Field | Details |
|-------|---------|
| **Download** | 🔗 https://github.com/Azure/AzurePublicDataset (Azure Functions section) |
| **MAF1 (2019)** | 2 weeks of serverless invocations; steady, gradually shifting rates |
| **MAF2 (2021)** | 2 weeks; extremely bursty, up to 50× average rate spikes |
| **Used by** | AlpaServe |

General serverless arrival patterns — useful as an alternative arrival model.

---

## Synthetic Workload Generation Methods

> [!NOTE]
> Most papers describe synthetic methods that should be replicated for FairRoute's experiments. These are not datasets but workload generation techniques.

### Arrival Processes

| Method | Description | Used By |
|--------|-------------|---------|
| **Poisson Arrival** | Standard exponential inter-arrival times; rate λ controls load | Nearly all papers |
| **Gamma Arrival** | Heavier-tailed; CV parameter controls burstiness (CV∈[2,8]) | DualMap, Lodestar, Lluminix, AlpaServe, Geometry-Aware |
| **Moving-Block-Bootstrap** | Preserves short-range gap correlations from real traces (CV=2.73) | CacheRoute |
| **Micro-burst** | Periodic short high-intensity spikes | HW-Router |
| **Sustained Overload** | Continuous rate exceeding capacity | HW-Router |

### Workload Shaping

| Method | Description | Used By |
|--------|-------------|---------|
| **Zipfian tenant skew** | Few heavy clients dominate traffic — stresses Fairness Tracker | D²LPM, EWSJF, QUARTZ, PRISM, Roadmap §8 |
| **Adversarial-conflict** | Starved client targets saturated replica with cached prefix | Roadmap §3.3 & §8 (FairRoute-specific) |
| **Multi-turn extension (10 turns)** | Extends ShareGPT conversations to amplify prefix sharing | DualMap, D²LPM |
| **P/D ratio control** | Fixed prompt-to-decode ratio (1:4 or 4:1) to isolate compute vs. memory pressure | Online LP |
| **Rate drift simulation** | Lognormal random walk on key rates (σ=0.3/step, 6 intervals) | CacheRoute |
| **Whale injection** | Inject high-rate "whale" keys (5–45% head share) | CacheRoute |
| **Distribution shift** | Multiply output distribution by factor s∈{1.0, 1.5, 2.0, 2.5, 3.0}× | Robust KV Cache |
| **Prefix sharing synthesis** | Controlled prefix ratios: 10%, 30%, 50%, 70% | Lodestar |

### Output Length Distributions

| Distribution | Parameters | Used By |
|-------------|-----------|---------|
| **Uniform** | 50–500 tokens | EWSJF, General Non-Clairvoyant |
| **Log-normal** | Varies | Geometry-Aware, Robust KV Cache |
| **Pareto** | Heavy-tailed | General Non-Clairvoyant |
| **Power-law** | S(128), M(256), L(512) mean; capped at 6K tokens | Lluminix |

---

## Recommended Dataset Combination for FairRoute

> [!IMPORTANT]
> The Experimental Roadmap (Section 8) explicitly recommends the 4-dataset combination below, following Robust KV Cache Management's precedent of combining BurstGPT + Azure + ShareGPT.

### Primary Stack (Required)

```
┌──────────────────────────────────────────────────────────────────┐
│                      FairRoute Primary Stack                     │
├──────────────────┬───────────────────────────────────────────────┤
│ ShareGPT         │ Prompt/response distributions; multi-turn     │
│                  │ prefix sharing; direct baseline comparability │
├──────────────────┼───────────────────────────────────────────────┤
│ WildChat-1M      │ Diverse, multilingual, large-scale chat;      │
│                  │ stress-test prefix diversity/skew             │
├──────────────────┼───────────────────────────────────────────────┤
│ Azure LLM Trace  │ Production arrival patterns; diurnal cycles;  │
│                  │ realistic burstiness                          │
├──────────────────┼───────────────────────────────────────────────┤
│ BurstGPT         │ Explicit burst injection; skewed I/O ratio;   │
│                  │ queueing/admission stress                     │
└──────────────────┴───────────────────────────────────────────────┘
```

### Generalization Check

| Dataset | Purpose |
|---------|---------|
| **LMSYS-Chat-1M** | Verify results hold on a larger, more diverse distribution |

### Targeted Experiments

| Dataset | Experiment | FairRoute Component Tested |
|---------|-----------|---------------------------|
| **LongBench** | Long-context KV-cache pressure | Locality Estimator, Prediction Engine |
| **Alpaca** | Short single-turn (no prefix sharing) | Fairness Tracker (without locality benefit) |
| **ToolBench** | Very high prefix sharing (85%+) | Locality Estimator (DART), cache-hit rate |
| **Mooncake Traces** | Real production prefix patterns | All three signals (F, L, P) |
| **Arena-Hard-Auto** | Compute-heavy requests | Overload guard, Prediction Engine |

### Synthetic Overlays (on top of real datasets)

| Overlay | Purpose | Roadmap Reference |
|---------|---------|-------------------|
| **Zipfian tenant skew** | Stress the Fairness Tracker with heavy/light clients | Section 8 |
| **Adversarial-conflict injection** | Starved client targets saturated replica with cached prefix | Sections 3.3, 8 |
| **Poisson + Gamma arrivals** | Vary burstiness characteristics | Standard practice |
| **Multi-turn extension (10 turns)** | Amplify prefix-sharing opportunities | DualMap, D²LPM |
| **P/D ratio control (1:4 and 4:1)** | Isolate prefill-heavy vs decode-heavy regimes | Online LP |

---

## Quick-Reference Download Links

| #   | Dataset                      | URL                                                                       |     |
| --- | ---------------------------- | ------------------------------------------------------------------------- | --- |
| 1   | ShareGPT                     | https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered |     |
| 2   | LMSYS-Chat-1M                | https://huggingface.co/datasets/lmsys/lmsys-chat-1m                       |     |
| 3   | Azure LLM Inference Trace    | https://github.com/Azure/AzurePublicDataset                               |     |
| 4   | BurstGPT                     | https://github.com/HPMLL/BurstGPT                                         |     |
| 5   | WildChat-1M                  | https://huggingface.co/datasets/allenai/WildChat-1M                       |     |
| 6   | Alpaca                       | https://github.com/tatsu-lab/stanford_alpaca                              |     |
| 7   | LongBench                    | https://github.com/THUDM/LongBench                                        |     |
| 8   | Arena-Hard-Auto              | https://github.com/lm-sys/arena-hard-auto                                 |     |
| 9   | ToolBench                    | https://github.com/OpenBMB/ToolBench                                      |     |
| 10  | MMLU                         | https://huggingface.co/datasets/cais/mmlu                                 |     |
| 11  | LooGLE                       | https://huggingface.co/datasets/bigainlp/LooGLE                           |     |
| 12  | Mooncake Traces              | https://github.com/kvcache-ai/Mooncake                                    |     |
| 13  | UltraChat                    | https://aclanthology.org/2023.emnlp-main.183/                             |     |
| 14  | MixInstruct / LLM-Blender    | https://github.com/yuchenlin/LLM-Blender                                  |     |
| 15  | MT-Bench                     | https://huggingface.co/spaces/lmsys/mt-bench                              |     |
| 16  | LMSYS Chatbot Arena          | https://lmarena.ai/                                                       |     |
| 17  | Azure Functions Trace        | https://github.com/Azure/AzurePublicDataset (Functions section)           |     |
| 18  | MultiHop-RAG                 | arXiv:2401.15391                                                          |     |
| 19  | Lodestar Per-Request Dataset | https://github.com/gangmuk/Lodestar                                       |     |
| 20  | DualMap Code                 | https://github.com/ASISys/DualMap                                         |     |
| 21  | Online LP for Vidur          | https://github.com/qqwetidx/Online-Linear-Programming-for-Vidur           |     |
| 22  | HW-Router Code + Data        | https://github.com/UCF-ML-Research/HW-Router                              |     |
| 23  | KV Routing (Randomization)   | https://github.com/fzwark/KVRouting                                       |     |
| 24  | VTC Artifact                 | https://github.com/Ying1123/VTC-artifact                                  |     |
| 25  | Preble Code                  | https://github.com/WukLab/preble                                          |     |

---

> [!NOTE]
> **Lodestar** releases the first **public per-request LLM routing dataset** pairing cluster snapshots, request features, and per-token/TTFT latencies from real cloud GPU clusters — available at https://github.com/gangmuk/Lodestar. This could be valuable for training FairRoute's H3 (learned combiner) model.
