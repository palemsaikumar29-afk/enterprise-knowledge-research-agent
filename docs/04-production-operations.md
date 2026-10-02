# Production Operations — Enterprise Knowledge Research Agent

**Interview Kickstart — Applied Agentic AI, Week 1**
**Author: Sai Kumar Palem**

---

## 1. Evaluation

### 1.1 Golden dataset
- A curated **golden set of ~50 queries** (growing toward ~500) with human-written reference answers, known relevant documents, and expected citations. It covers the query types we care about: broad research (like our running example), narrow factual lookup, freshness-sensitive questions, follow-ups, and adversarial probes (injection attempts, unauthorized-data requests).
- The golden set is versioned alongside prompts and retrieval config — a prompt change that moves golden-set scores is a deliberate decision, not an accident.

### 1.2 LLM-as-judge (with bias controls)
- Human grading can't cover 300 queries/day, so an LLM judge scores production samples against the golden references on: citation precision/recall, freshness compliance, coverage, and abstention correctness (did it abstain exactly where evidence was missing?).
- **Bias controls:** position shuffling (judge sees candidate answers in random order), reference-based scoring (judge compares against the golden answer, not its own priors), and periodic human spot-checks that measure judge-vs-human agreement. If agreement drifts, the judge is recalibrated, not trusted.

### 1.3 Offline + online
- **Offline:** every config/prompt/model change runs against the golden set before deploy. Regressions block the rollout.
- **Online:** production traffic is sampled into the judge pipeline; answer acceptance (north star), citation precision, and cost per query are tracked as live metrics. Online findings feed back into golden-set growth — real failures become new golden cases.

---

## 2. Observability

### 2.1 Traces and spans
- **One trace per query.** Plan steps, tool calls, retrieval stats, synthesis decisions, and human-review events are spans with timestamps — this is the **time journal** the escalation team asked for (§1.3.2).
- **Error traces** capture the failing tool, its arguments, the error class, and the recovery action taken — the **error traces** from the same requirement.
- Traces record the permission scope the query ran under, enabling exact replay for leak investigations.

### 2.2 Metrics (by level)
- **Query level:** latency (p50/p95 against the 8s target), answer acceptance, citation precision, cost per query.
- **Component level:** retrieval recall/coverage on the golden set, rerank quality, synthesis abstention rate, planner replan rate, injection-flag rate.
- **System level:** queries/hour vs the 100/hr peak, queue depth, worker utilization, cache hit rate (per permission scope), judge-vs-human agreement.

### 2.3 Cost accounting (Finance §1.5.1)
- Every query carries a **cost ledger**: tokens × model rates per model used (loop model vs synthesis model) + tool-call costs (rerank calls, fetches). The ledger is in the trace and aggregated into dashboards: cost per query (p50/p95/max), cost by query pattern, cost trend as users scale 1,000 → 5,000.
- The **per-query budget cap** is enforced from this ledger in real time — Finance's tracking requirement and the cost-runaway mitigation are the same mechanism.

### 2.4 Logging, alerts, replay
- Structured logs for every span; alerts on: p95 latency breaching 8s, error-rate spikes, injection-flag spikes, budget-cap hit rate spikes, judge-score drops, review-queue depth growth.
- **Replay:** any trace can be re-executed (same permission scope, same evidence snapshot) to reproduce a bug or audit a leak claim. The escalation console (§6.4 in the design doc) is built on traces + replay.

---

## 3. Feedback loops

1. **Explicit:** thumbs up/down on answers; "report an issue" on any report section.
2. **Implicit:** answer acceptance (copied, shared, followed-up vs rephrased), follow-up patterns (a rephrase signals the first answer missed).
3. **Corrections:** user-flagged errors are recorded; corroborated or human-approved corrections enter long-term memory (glossary, correction store) at v1.
4. **Eval loop:** production failures become new golden-set cases; judge scores gate deploys. The system gets harder to fool over time instead of drifting.

---

## 4. Invocation patterns & cost

(Design detail in `01-system-design.md` §5; operations view here.)

| Pattern | Ops notes |
|---|---|
| Sync (≤8s) | Latency SLO is p95 ≤ 8s; alert on breach. Cheapest per query (single pass, no status infra). |
| Streaming (10–30s) | Step-level progress updates are emitted from real loop state. Slightly higher cost (longer loop); justified by answer quality on broad queries. |
| Async (long reports) | Job queue with status endpoint (queued → researching → synthesizing → done). Absorbs peak overflow: when sync workers saturate near 100 queries/hour, excess shifts to async with honest progress instead of timing out. |

**Cost control levers:** model routing (small model for loop, large for synthesis), permission-scoped caching (follow-ups and repeated queries hit cache), evidence shortlisting (synthesis sees top-k, not the raw dump). All three are visible in the per-query cost ledger, so their effect is measurable, not assumed.

---

## 5. Scale & operations

### 5.1 Breakpoints (from the stakeholder numbers)

| Load | What breaks first | Response |
|---|---|---|
| 300 queries/day average | Nothing — baseline is light | Small stateless worker pool; shared retrieval cluster |
| 100 queries/hour peak (8× average) | Sync worker saturation on bursts | Queue + async pattern absorbs overflow; autoscale workers on queue depth |
| 1,000 → 5,000 users (3 years) | Permission-scope cardinality (cache hit rate), identity-resolution load, cost growth | Scope-hierarchy caching (evaluate carefully — see residual risks), cached identity resolution, model routing savings compound with volume |

Concurrency math: at peak, 100 queries/hour × 8s average ≈ **0.22 concurrent queries** — the fleet is small; the shared expensive resources are the ~10M-doc vector index and the cross-encoder reranker, which scale independently of user count.

### 5.2 Graceful degradation
- Reranker down → fall back to hybrid scores without rerank (slightly lower quality, still fresh and permission-safe).
- Synthesis model degraded → shorter answers with more abstention rather than lower-quality claims.
- Retrieval slow → streaming pattern keeps the user informed; async if it exceeds the streaming budget.
- **Degradation never touches the guardrails:** freshness cutoff, ACL pre-filter, and cite-or-abstain hold in every degraded mode. We degrade completeness and latency, never safety.

### 5.3 Prompts as config
Planner prompts, synthesis instructions, and judge rubrics are versioned configuration, not code. Changes go through the golden-set gate (§1.3) and canary deploy. A bad prompt rolls back like a bad config — in minutes, without a code deploy.

### 5.4 On-call playbook
1. **Alert fires** (latency, errors, injection flags, cost spike) → open the trace dashboard filtered to the alert window.
2. **Identify the layer:** retrieval (recall drop?), synthesis (citation precision drop?), loop (replan-rate spike?), infra (queue depth?).
3. **Replay** a failing trace to reproduce.
4. **Mitigate:** rollback prompt/config via the config system; shift traffic to async; page for index/reranker issues.
5. **Grow the golden set** with the failure so it can't regress silently.

---

## 6. v0 → v1 evolution

| | v0 (initial rollout, 1,000 users) | v1 (toward 5,000 users) |
|---|---|---|
| Invocation | Sync + streaming | + Async long-report jobs with status endpoint |
| Retrieval | Hybrid + ACL pre-filter + 2-year cutoff | Tuning from golden-set growth; scope-hierarchy caching (carefully) |
| Memory | Session memory (follow-ups work) | + Long-term memory: glossary, corrections, permission-scoped past answers |
| Synthesis | Cite-or-abstain + citation verifier | Same, with judge-measured precision targets |
| Eval | Golden set ~50, human spot-checks | Golden set → ~500, LLM-as-judge with bias controls gating deploys |
| Cost | Per-query ledger + budget cap | Model routing savings compound; distilled loop models evaluated |
| Safety | Injection flags logged; review queue for low-confidence answers | Review queue metrics; memory-poisoning safeguards on corrections |
| Ops | Traces, escalation console, replay | On-call playbook exercised; canary deploys on prompt config |

v0 proves the contract — fresh, cited, permission-safe answers with honest progress and full traceability. v1 scales the contract: more users, longer reports, compounding memory, and an eval loop that keeps quality from drifting as volume grows.
