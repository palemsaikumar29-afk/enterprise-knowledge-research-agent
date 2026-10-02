# System Design Document — Enterprise Knowledge Research Agent

**Interview Kickstart — Applied Agentic AI, Week 1**
**Author: Sai Kumar Palem**

---

## 1. Stakeholder Requirements

These requirements come from the stakeholder Requirements Review and are the contract this design satisfies. Where they give concrete numbers, those numbers govern the design (they supersede any illustrative figures used during exploration).

### 1.1 Executive
1. The system shall receive a user query and provide a **summarized answer with valid citations**.

### 1.2 Business
1. Answers must be **fresh** — no source older than 2 years may contribute to an answer.
2. The system shall **allow follow-up queries** (conversational context over a session).

### 1.3 Support Team
1. The system shall produce **easy-to-use information** for quickly responding to end users.
2. The escalation team shall have **easy traceability into error traces and time journals** for any query.

### 1.4 Security / Safety
1. An **authenticated user shall receive only authorized information** (no permission leaks).
2. The system shall be **resilient to mal-intended queries and malicious data in the sources**.

### 1.5 Finance
1. The system shall provide **tracking of cost per query**.

### 1.6 Non-functional requirements
1. **Scale:** initial rollout 1,000 users; 5,000 users within 3 years.
2. **Peak load:** 100 queries per hour.
3. **Average load:** 300 queries per day.
4. **Response time:** 8 seconds average.
5. **Long answers:** a progress indicator must be shown for longer-running answers.

### 1.7 Requirements traceability

| Stakeholder | Requirement | Satisfied by (design section) |
|---|---|---|
| Executive §1.1.1 | Summarized answer with valid citations | §4.7 Synthesis engine (cite-or-abstain); §7 Grounding guardrails |
| Business §1.2.1 | Freshness ≤ 2 years | §4.3 Retrieval pipeline (recency weighting + hard freshness cutoff) |
| Business §1.2.2 | Follow-up queries | §4.6 Short-term memory (session context); §6.3 Follow-up handling |
| Support §1.3.1 | Easy-to-use info for end-user responses | §4.7 Synthesis (structured report + source list); §6.1 Answer formats |
| Support §1.3.2 | Traceability: error traces + time journals | §8 Observability (trace timelines, span journals); §6.4 Escalation console |
| Security §1.4.1 | Authenticated user → authorized info only | §4.4 ACL permission filter (pre-filter at retrieval, permission-scoped caches) |
| Security §1.4.2 | Resilient to malicious queries & source data | §7.3 Injection defense (untrusted-data discipline); §3 Failure-mode mitigations (see `03-failure-modes.md`) |
| Finance §1.5.1 | Cost per query tracking | §8.3 Cost accounting; `04-production-operations.md` §2 |
| NFR §1.6.1–3 | 1,000 → 5,000 users; 100 q/hr peak; 300 q/day avg | §9 Capacity planning |
| NFR §1.6.4 | 8s average response | §5 Invocation patterns (sync fast path, streaming); §9 latency budget |
| NFR §1.6.5 | Progress indicator for long answers | §5.2 Streaming step updates; §5.3 Async job status |

### 1.8 North star & guardrails

- **North star metric:** answer acceptance rate — did the employee accept and use the answer (thumbs up/down, copy/share, follow-up rather than rephrase)?
- **Guardrails (hard constraints):**
  - **Zero permission leaks** — no document content outside the user's authorization may ever enter the context window, the answer, or a cache readable by someone else.
  - **Zero uncited claims** — every factual claim in a delivered answer carries a citation; anything unanchored is abstained from, never emitted.

---

## 2. System overview

The agent is an **orchestrated Plan → Act → Observe loop** wrapped around three governed subsystems: a permission-aware retrieval pipeline, a grounded synthesis engine, and a session memory. An observability plane records everything the loop does.

```
Employee query + identity
        │
        ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│ ORCHESTRATOR │────▶│    PLAN      │────▶│   ACT (tools +       │
│ loop control,│     │ hybrid       │     │   retrieval, ACL     │
│ budgets,     │     │ planner      │     │   pre-filtered)      │
│ termination  │     └──────────────┘     └──────────────────────┘
└──────────────┘            ▲                       │
        │                   │                       ▼
        │            ┌──────────────┐     ┌──────────────────────┐
        └───────────▶│   OBSERVE    │◀────│ evidence sufficiency │
                     │ evaluate &   │     │ coverage / freshness │
                     │ refine       │     │ checks               │
                     └──────────────┘     └──────────────────────┘
                            │ sufficient
                            ▼
                     ┌──────────────┐
                     │  SYNTHESIZE  │──▶ cited report + audit log
                     │ cite-or-     │
                     │ abstain      │
                     └──────────────┘

   Side planes:  MEMORY (session + long-term)   OBSERVABILITY (traces, cost, metrics)
```

The loop is **bounded**: it terminates on goal satisfaction, a step/depth limit, a per-query cost budget, or repeated no-progress iterations. Autonomy exists strictly inside that envelope.

---

## 3. Main system components

### 3.1 Orchestrator
Owns the loop. Responsibilities:
- Accepts the query, identity context, and session state; emits the final answer package.
- Drives Plan → Act → Observe iterations until a termination trigger fires.
- Enforces budgets: max reasoning steps, max retrieval depth, per-query cost cap, wall-clock deadline (aligned to the 8s average response target for the sync path; async path for long reports).
- Selects invocation pattern per query (see §5): sync, streaming, or async.

### 3.2 Hybrid planner
- **Graph backbone** for known, repeatable flows (e.g., "broad research question" → decompose → retrieve per sub-topic → synthesize). Graphs are inspectable, testable, and cheap.
- **LLM reasoning at decision nodes** where the plan must adapt: decomposing an unfamiliar query, judging whether evidence is sufficient, choosing the next tool.
- **Termination triggers:** goal satisfied · step/depth limit reached · per-query budget cap hit · repeated iterations with no new evidence. The planner never decides to continue past a hard trigger.

### 3.3 Retrieval pipeline (deep dive §4.3)
Hybrid retrieval over ~10M enterprise documents:
1. **Query understanding** — expand the user query into sub-queries (e.g., our running example becomes "personalization initiatives 2024–2026", "recommendation systems projects", "teams owning personalization").
2. **Candidate generation** — BM25 (keyword precision, catches exact initiative names and acronyms) **+** dense vector search (semantic recall, catches paraphrases) **+** recency weighting (boosts 2024–2026 content).
3. **Hard freshness cutoff** — documents older than 2 years are excluded before ranking (Business requirement §1.2.1). Recency weighting then orders *within* the fresh set.
4. **Rerank** — cross-encoder over the merged candidate set for final ordering.

### 3.4 ACL permission filter (deep dive §4.4)
- **Pre-filter at retrieval, never post-filter.** The user's identity, group memberships, and clearance are resolved once per query; retrieval only ever sees documents the user is authorized for. Unauthorized content never enters the candidate set, the context window, or any cache.
- **Permission-scoped cache keys** — cached retrieval results are keyed by (query signature, permission scope hash), so a cache hit can never serve one user's authorized view to another user.
- Satisfies Security requirement §1.4.1 structurally rather than as a downstream check.

### 3.5 Tool schemas
All tools the agent can call are declared with **typed schemas** (name, typed inputs/outputs, side-effect classification, cost estimate, timeout, retry policy, failure modes). The planner can only invoke declared tools with validated arguments. Each tool declares how its failures surface (transient → retry with backoff; permission-denied → abort step and replan; malformed output → quarantine and retry once), so the Observe step can reason about failures instead of swallowing them.

### 3.6 Short-term vs long-term memory
- **Short-term (session) memory** — the working plan, the evidence log (what was retrieved, from where, with citations), and the conversation turns of the current session. This is what makes **follow-up queries** work: a follow-up like *"What about the mobile team's work?"* is resolved against the session context (the running example query's scope, its evidence set, its answer) rather than re-searched from scratch. Session memory is scoped to the authenticated user and expires with the session.
- **Long-term memory** — past queries and their accepted answers (per user, permission-scoped), a company glossary (initiative names, team aliases, acronyms that improve future retrieval), and recorded corrections (when a user flags a wrong answer, the correction is stored and consulted on similar future queries). Long-term memory is itself permission-scoped: a memory entry derived from restricted documents is only visible to users authorized for those documents.

### 3.7 Synthesis engine
- Groups evidence by theme/team/initiative and drafts a structured report.
- **Cite-or-abstain:** every factual claim must be anchored to retrieved evidence with a citation (document title, section, retrieval timestamp). Claims that cannot be anchored are dropped — the engine abstains rather than hallucinates.
- Produces two artifacts: the **end-user report** (easy-to-use, structured, with a source list — Support requirement §1.3.1) and an **audit log** of which sources grounded which claims (feeds the escalation console, §6.4).

---

## 4. Deep dives

### 4.1 Why Plan → Act → Observe (not single-shot RAG)
Our running example cannot be answered by one retrieval pass: it needs multiple sub-queries, freshness filtering, coverage judgment ("do we have *all* initiatives, or just the famous ones?"), and synthesis across dozens of documents. The loop lets the agent notice gaps (Observe) and repair them (re-Plan) instead of delivering a thin answer with false confidence.

### 4.2 Retrieval details
- **Corpus:** ~10M documents (wikis, design docs, launch posts, roadmaps, meeting notes).
- **Freshness:** hard cutoff at 2 years at retrieval time (not at display time), so stale documents can never silently anchor a claim.
- **Hybrid scoring:** `score = α·BM25 + β·vector + γ·recency`, tuned on the golden eval set; cross-encoder rerank on top ~100 for the final top-k.
- **Deduplication:** near-duplicate initiatives (e.g., the same launch announced in three places) are clustered so the report lists each initiative once, citing the canonical source.

### 4.3 Permissions details
Identity → group/clearance resolution happens once per query at the orchestrator, before any retrieval. The retrieval index carries ACL metadata per document; the query executes against the authorized subset only. Cache keys include a permission-scope hash. Audit logging records which permission scope a query ran under, so a leak investigation can replay exactly what the user was allowed to see.

### 4.4 Tools details
Example toolset: `hybrid_search`, `doc_fetch` (full document by ID, permission-checked), `org_lookup` (team/owner metadata), `time_filter` (date-range scoping), `calculator`/`aggregator` for structured comparisons. Each has a typed schema, cost estimate (for budget accounting), and declared failure behavior. Tools are allow-listed; the planner cannot invent tool calls.

### 4.5 Memory details
- Session memory: plan state, evidence log with citations, conversation turns; enables follow-ups and avoids re-retrieval.
- Long-term memory: permission-scoped past answers, glossary, corrections; improves retrieval and synthesis over time without leaking across permission boundaries.

### 4.6 Synthesis details
The synthesis prompt is constrained: it receives only retrieved, permission-checked, fresh evidence plus the session context, and is instructed to cite every claim or omit it. A post-generation citation verifier checks that every citation in the draft resolves to a real retrieved span; failures trigger regeneration or abstention. The final report includes a "Sources" section and an explicit "What I could not verify" note when coverage is incomplete — honest gaps beat confident fiction.

---

## 5. Invocation patterns

| Pattern | When | Behavior | Meets |
|---|---|---|---|
| **Sync** | Simple, narrow queries expected to complete quickly | Full answer within the response; target ≤ 8s average | NFR §1.6.4 |
| **Streaming** | Broad research queries (like our running example) that need 10–30s | Tokens stream as generated; **step-level progress updates** ("Searching personalization initiatives…", "Synthesizing 23 sources…") satisfy the progress-indicator requirement while work continues | NFR §1.6.5 |
| **Async** | Very long reports, deep multi-round research | Returns a job ID immediately; progress indicator via **job status endpoint** (queued → researching → synthesizing → done, with % complete and current stage); user is notified on completion | NFR §1.6.5 |

The orchestrator picks the pattern from a quick complexity estimate (query breadth, expected evidence volume); the user can also request async explicitly. **Progress indication is never a fake spinner** — it reflects real loop state (current plan step, sources gathered so far).

---

## 6. Answer delivery & support surfaces

### 6.1 Answer formats
The delivered answer is a structured, skimmable report: executive summary, initiatives grouped by theme/team with dates, each claim cited, a source list, and a "could not verify" section. Support staff can lift sections directly into end-user responses (§1.3.1).

### 6.2 Follow-up queries
Follow-ups run in the same session: the query is interpreted against session memory (prior question, evidence log, answer). *"Which of these launched in 2025?"* filters the already-gathered evidence; *"What about the mobile team?"* triggers a targeted retrieval round scoped to the new sub-topic, reusing the plan. A follow-up that drifts beyond the session's scope starts a fresh plan but inherits the glossary and corrections from long-term memory.

### 6.3 Malicious-query resilience
Mal-intended queries (jailbreak attempts, prompt-injection probes, requests for unauthorized data) are handled structurally: the planner's tool schemas admit no instruction-following tools; retrieval is permission-bounded so "show me the CEO's compensation doc" returns only authorized results; synthesis can only cite retrieved evidence. Attempts are logged to the trace for security review (§1.4.2).

### 6.4 Escalation console (Support §1.3.2)
Every query's trace is viewable internally: a **timeline** of plan steps with timestamps (the "time journal"), each step's inputs/outputs, **error traces** with the failing tool, arguments, and error class, plus the permission scope and cost ledger. An escalation engineer can replay the exact query (same permission scope, same evidence snapshot) to reproduce an issue.

---

## 7. Defense-in-depth guardrails

1. **Grounding & citations** — cite-or-abstain in synthesis; post-generation citation verification.
2. **Structural permissions** — ACL pre-filter at retrieval; permission-scoped caches; no post-filtering.
3. **Injection defense** — retrieved documents are **untrusted data, never instructions**. System prompts bracket tool outputs as data; the planner is instructed to ignore imperative language in retrieved content; any retrieved content attempting to direct tool use is flagged in the trace.
4. **Bounded loop** — step/depth limits, per-query budget cap, no-progress termination (see failure-mode analysis in `03-failure-modes.md`).
5. **Human-in-the-loop rules** — low-confidence answers, conflicting evidence on high-stakes claims, or detected injection attempts route to human review before delivery rather than auto-delivering.

---

## 8. Observability (summary; full detail in `04-production-operations.md`)

- **Trace per query:** plan steps as spans with timestamps (time journal), tool calls, retrieval stats, synthesis decisions.
- **Error traces:** failures captured with tool, args, error class, and recovery action — directly serving the escalation team (§1.3.2).
- **Metrics:** latency (p50/p95 vs the 8s target), retrieval recall/coverage, citation precision, answer acceptance (north star), and **cost per query** (token usage × model rates + tool-call costs, Finance §1.5.1).
- **Replay:** any trace can be replayed for debugging or audit.

---

## 9. Capacity planning (NFR §1.6)

Working from the stakeholder numbers:

| Parameter | Value | Implication |
|---|---|---|
| Users | 1,000 now → 5,000 in 3 years | Identity/ACL resolution and session memory must scale 5×; permission-scoped caches grow with distinct scopes, not just users |
| Average load | 300 queries/day (~12.5/hr) | Baseline is light; a small worker pool suffices |
| Peak load | 100 queries/hour | 8× average — design for burst; queue + async pattern absorbs overflow |
| Avg response | 8s | Latency budget: ~1s planning, ~3–4s retrieval+rerank, ~3s synthesis; streaming keeps perceived latency low |
| Concurrency at peak | 100/hr × 8s ≈ **0.22 concurrent** average; bursts need headroom | Even 5–10× burst headroom is a modest fleet; the expensive resources are the vector index and reranker, shared across queries |

**Scaling levers:** stateless orchestrator workers behind a queue; shared retrieval cluster (the ~10M-doc index) scaled independently; permission-scoped result caches to absorb repeated queries; model routing (small model for planning/observation, large model only for final synthesis) to hold the 8s budget and per-query cost down as users grow 5×. Async pattern guarantees the peak never degrades into timeouts — excess load shifts to queued jobs with honest progress indicators.

---

## 10. v0 → v1 evolution (summary)

- **v0 (initial rollout, 1,000 users):** sync + streaming paths, hybrid retrieval with ACL pre-filter, cite-or-abstain synthesis, session memory for follow-ups, per-query cost tracking, trace timelines for escalation.
- **v1:** async long-report jobs, long-term memory (glossary, corrections, permission-scoped past answers), LLM-as-judge eval loop on a growing golden set, model routing and caching for cost control at 5,000-user scale, human-in-the-loop review queue for low-confidence/high-stakes answers.

Full production discussion: [`docs/04-production-operations.md`](04-production-operations.md).
