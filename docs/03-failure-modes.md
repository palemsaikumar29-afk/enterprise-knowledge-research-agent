# Failure Mode Analysis — Enterprise Knowledge Research Agent

**Interview Kickstart — Applied Agentic AI, Week 1**
**Author: Sai Kumar Palem**

Agentic systems fail differently from pipelines: the failures are *behavioral* (loops, drift, overconfidence) as much as technical. Each failure below lists how it shows up in our running example query and the structural mitigation — not a prompt tweak, but a mechanism that holds even when the model misbehaves.

## Failure mode table

| # | Failure mode | Symptom (in our example query) | Structural mitigation |
|---|---|---|---|
| 1 | **Infinite loops** — the agent re-plans and re-retrieves without converging | Trace shows 15 rounds of "search personalization initiatives" with near-identical queries; cost climbs, no answer ships | Hard termination triggers: max reasoning steps, max retrieval depth, **no-progress detection** (stop when consecutive iterations add no new evidence). The loop cannot vote to continue past a trigger. |
| 2 | **Plan derailment** — the plan drifts off the user's intent | Query asked about *company initiatives*; the agent wanders into academic papers on recommendation algorithms | The Observe step re-anchors every iteration against the original query intent and the plan graph's allowed transitions; off-plan tool calls are rejected by typed schemas. Drift is detected as "evidence not covering the plan's sub-queries" and triggers re-planning, not continuation. |
| 3 | **Context overflow** — evidence exceeds the window; early sources get evicted | 40 documents retrieved; the synthesis cites only the last 8, dropping the 2024 initiatives that answered half the query | Evidence log is external to the context window (stored in session memory with IDs + citations). Synthesis works from a *ranked, deduplicated shortlist*, not the raw retrieval dump; overflow spills to the log, never silently out of the answer. Streaming synthesis processes evidence in chunks. |
| 4 | **Cascading errors** — one bad tool result poisons downstream steps | A malformed `doc_fetch` returns an error blob; the synthesizer quotes the error text as a "source" | Typed tool schemas declare failure modes; Observe classifies failures (transient → retry with backoff; malformed → quarantine the result and retry once; permission-denied → abort step and replan). Quarantined outputs are marked untrusted and can never enter the evidence log. |
| 5 | **Confident hallucination** — fluent answer with fabricated specifics | Report invents "Project Lumen, launched Q3 2025 by the Growth team" — sounds right, never existed | **Cite-or-abstain**: every claim must anchor to retrieved evidence; a post-generation citation verifier checks each citation resolves to a real retrieved span. Unanchored claims are dropped, and the "could not verify" section says so explicitly. The north-star metric (answer acceptance) is tracked against citation precision to catch drift. |
| 6 | **Cost runaway** — a pathological query burns budget | A vague query ("tell me everything about personalization") triggers 60 retrieval rounds and a frontier-model synthesis | **Per-query budget cap** (Finance requirement made structural): the orchestrator tracks spend in real time (tokens × model rates + tool costs); hitting the cap terminates the loop and synthesizes from gathered evidence with a "partial results — budget reached" note. Model routing keeps the loop steps cheap by default. |

## Defense-in-depth guardrails

No single mitigation covers everything; the guardrails overlap so that one layer's miss is caught by the next.

### 1. Grounding & citations
- Cite-or-abstain in synthesis; post-generation citation verification; "could not verify" section instead of silent gaps.
- Guards primarily against **(5) hallucination**; secondarily against **(4)** (quarantined junk can't be cited).

### 2. Structural permissions
- ACL pre-filter at retrieval; permission-scoped cache keys; permission scope recorded in the trace.
- Guards against **unauthorized disclosure** (the zero-leak guardrail); also bounds **(6)** (a restricted scope retrieves less, costs less).

### 3. Injection defense — retrieved content is untrusted data
- **The core discipline:** documents, wikis, and meeting notes are *data*, never instructions. System prompts bracket all tool outputs as data; the planner is instructed to treat imperative language in retrieved content ("ignore previous instructions…", "send this report to…") as content to summarize, not commands to follow.
- Detection: an injection-pattern scan over retrieved text flags suspicious spans in the trace; flagged content is still usable as *evidence* (with its flag visible) but can never direct tool calls or change the plan.
- Human-in-the-loop: a detected injection attempt on a high-stakes query routes to review before delivery.
- This is the direct answer to Security requirement §1.4.2 ("resilient to mal-intended … data in the sources").

### 4. Malicious-query handling
- Jailbreak probes and unauthorized-data requests fail structurally: tool schemas admit no instruction-following side effects, retrieval is permission-bounded, and synthesis can only cite retrieved evidence. There is nothing to escalate *to* — the attack surface is the absence of a confused deputy.
- All attempts are logged to the trace for security review.

### 5. Human-in-the-loop rules
Route to human review before delivery when any of these hold:
- Overall answer confidence below threshold, or key claims rest on a single weak source.
- Conflicting evidence on a high-stakes claim (e.g., two docs disagree on a launch date).
- Detected injection attempt in retrieved sources.
- User explicitly requests verification.
The review queue is itself observable: queue depth and review latency are metrics, so the guardrail can't become a silent bottleneck.

### 6. Bounded autonomy (the backstop for 1, 2, and 6)
Step/depth limits, per-query budget cap, and no-progress termination form a triple backstop. Even if planning, observation, and cost tracking each misjudge, the loop still cannot run forever, spend forever, or drift forever.

## Residual risks (honest accounting)

- **Citation verifier false negatives:** a claim subtly misreading a real source passes verification. Mitigated by the golden eval set's citation-precision checks, not eliminated.
- **Scope-hierarchy cache errors:** if introduced at scale (see trade-offs §"revisit"), a misclassified scope leaks. Kept out of v0 deliberately.
- **Long-term memory poisoning:** a recorded "correction" that is itself wrong compounds over time. Mitigated at v1 by requiring corrections to be corroborated or human-approved before entering long-term memory.
