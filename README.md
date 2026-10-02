# Enterprise Knowledge Research Agent

**Interview Kickstart — Applied Agentic AI, Week 1 assignment**
*Agentic Research Systems: Planning, Tools & Guardrails*

## Assignment overview

Design an enterprise-grade research agent that can answer deep, open-ended research questions over a large corporate knowledge base — safely, accurately, and with full provenance. The design is driven by one running example query throughout:

> **"Summarize all company initiatives related to personalization and recommendation systems from the last 2 years."**

This is a hard query: it is broad, spans many documents, needs freshness ("last 2 years"), touches permission-sensitive internal content, and demands a synthesized, cited report rather than a list of links. The system below is designed to answer it — and queries like it — with a Plan → Act → Observe loop, permission-aware hybrid retrieval, grounded synthesis with cite-or-abstain, and production-grade guardrails.

## Deliverables

| # | Deliverable | File |
|---|-------------|------|
| 1 | Architecture diagram | This README (Mermaid flowchart below) |
| 2 | Written system design document | [docs/01-system-design.md](docs/01-system-design.md) |
| 3 | Trade-off analysis | [docs/02-trade-offs.md](docs/02-trade-offs.md) |
| 4 | Failure mode analysis | [docs/03-failure-modes.md](docs/03-failure-modes.md) |
| 5 | Production operations discussion | [docs/04-production-operations.md](docs/04-production-operations.md) |

## Architecture

```mermaid
flowchart TB
    subgraph Input["Query Input"]
        Q["Employee query:<br/>“Summarize all company initiatives<br/>related to personalization and<br/>recommendation systems<br/>from the last 2 years”"]
        ID["Identity & ACL context<br/>(user, groups, clearance)"]
    end

    O["🧭 Orchestrator<br/>(Plan → Act → Observe loop)"]
    P["📋 Hybrid Planner<br/>graph backbone for known flows<br/>+ LLM reasoning at decision nodes<br/>termination: goal met · step/depth limit<br/>· per-query budget cap · no-progress repeats"]
    A["⚙️ Act — Tools & Retrieval<br/>typed tool schemas<br/>hybrid retrieval: BM25 + vector + recency<br/>+ cross-encoder rerank<br/>🔒 ACL pre-filter at retrieval<br/>(permission-scoped cache keys)"]
    B["👁 Observe — Evaluate results<br/>sufficiency check · coverage gaps<br/>· evidence quality → refine plan or continue"]
    S["📝 Synthesize — Cited report<br/>group initiatives by theme/team<br/>every claim anchored to a source<br/>cite-or-abstain: no evidence → abstain"]
    D["📄 Delivered report<br/>+ audit log of sources"]

    M[("🧠 Memory<br/>short-term: working plan,<br/>evidence log, conversation<br/>long-term: past queries,<br/>glossary, corrections")]
    OB[("📊 Observability<br/>per-query trace & spans<br/>cost / latency / quality metrics<br/>replay & alerts")]

    Q --> O
    ID --> O
    O --> P
    P --> A
    A --> B
    B -->|gap found| P
    B -->|sufficient| S
    S --> D
    O -.-> M
    A -.-> M
    S -.-> M
    O -.-> OB
    A -.-> OB
    S -.-> OB

    style Q fill:#e3f2fd,stroke:#1565c0
    style ID fill:#fff3e0,stroke:#ef6c00
    style O fill:#e8f5e9,stroke:#2e7d32
    style P fill:#e8f5e9,stroke:#2e7d32
    style A fill:#e8f5e9,stroke:#2e7d32
    style B fill:#e8f5e9,stroke:#2e7d32
    style S fill:#e8f5e9,stroke:#2e7d32
    style D fill:#e1f5fe,stroke:#0277bd
    style M fill:#f3e5f5,stroke:#7b1fa2
    style OB fill:#fff8e1,stroke:#f9a825
```

**How the example query flows through the system:** the employee's query and identity context enter the Orchestrator, which plans a research strategy (decompose into sub-queries like "personalization initiatives 2024–2026", "recommendation systems projects", "relevant teams and launches"). Each Act step retrieves documents through ACL-pre-filtered hybrid search; Observe checks whether the evidence covers the two-year window and the themes adequately, looping back to refine the plan when gaps remain. Synthesis then produces a grouped, cited report — and any claim that cannot be anchored to retrieved evidence is dropped rather than hallucinated.

## Design principles

1. **Security is structural, not a filter.** Permissions are enforced at retrieval time (ACL pre-filter), never as a post-processing pass.
2. **Every claim is earned.** The synthesis engine cites sources or abstains — there is no uncited text in a delivered report.
3. **Bounded autonomy.** The agent plans and acts freely *within* a termination envelope: step/depth limits, a per-query cost budget, and no-progress detection.
4. **External content is data, never instructions.** Retrieved documents are treated as untrusted data; prompt-injection defenses are built into every stage.
5. **Observable by default.** Every query produces a trace, cost accounting, and replayable spans — the system is debuggable in production, not just in demos.

---

*Author: Sai Kumar Palem — Interview Kickstart Applied Agentic AI (Mid-June cohort)*
