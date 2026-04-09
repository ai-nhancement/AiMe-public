# AiMe

**A Memory-Augmented Cognitive Intelligence System**

> The model is not the AI. The model is a component. The system is the intelligence.

---

## What Is AiMe?

AiMe is a personal cognitive architecture that treats AI models as subordinate narrators inside a governed runtime -- not as the product itself.

Most AI assistants today share the same structural weakness: the model is the center of everything. The conversation ends, the state disappears, and the next session starts from scratch. Memory features are retrieval tricks bolted on after the fact. The model is expected to remember, reason, decide, stay truthful, maintain continuity, and somehow remain stable across long stretches of use. That is too much to ask from the wrong layer.

AiMe is built from the opposite premise. The system owns memory, identity, execution, initiative, and governance. Models are interchangeable reasoning engines inside that system. If the model is swapped, the relationship persists. If the conversation ends, the continuity does not.

The result is a companion system that knows who you are, what matters to you, what is open, and what should be surfaced at the right moment -- across weeks and months, not just sessions.

**AiMe has been running daily since November 2025, built entirely by a single developer. v3 went live on April 9, 2026.**

---

## Core Principle: Human-Led, System-Controlled

Everything traces back to one architectural principle:

- **The human retains authority.** Preferences, corrections, and instructions from the user are law.
- **The system governs execution and policy.** Routing, memory, initiative, tool use, and enforcement live in durable, auditable runtime layers.
- **The model is subordinate.** It narrates. It does not route, own memory, control execution, or decide when to speak.

This is enforced in every pipeline component. The model cannot execute tools -- the action dispatcher handles all tool routing deterministically. The model cannot initiate conversation -- the ThalamoFrontal Loop grants permission. The model cannot edit memory. The model cannot promote itself. The system controls it.

---

## The Architecture (v3)

AiMe v3 runs the **Self-Bounded Authority (SBA) Spine** -- a governed pipeline where no single component simultaneously owns truth determination, authority judgment, and natural language expression.

### Turn Pipeline

```
User Input
    |
LogicCore                  -- Router + orchestrator
    |
Intent Engine              -- 4-resolver consensus classification
    |
Action Dispatcher          -- Deterministic tool routing (15+ intents)
    |
SBA Spine
    |-- StateBuilder       -- Typed turn state from all signals
    |-- ResponseSynthesizer -- Response type selection
    |-- AuthorityEngine    -- 6 deterministic governance rules
    |-- ResponseAssembler  -- Constrained expression packet
    |
Single LM Call             -- Constrained by expression packet
    |
ComplianceValidator        -- Post-LM verification
    |
RIC Gate                   -- Integrity scoring, caveat injection
    |
LanguageCortex             -- Sole narrator, UI + voice output
```

Every stage except the language model call is deterministic. The model receives a fully assembled context and produces language within governed bounds. It does not decide what tools to run, which memories to recall, or what response type to use.

### Memory System

Three-layer append-only memory. Evidence is immutable -- never rewritten.

- **Evidence Ledger** -- Append-only SQLite store. The court transcript of the relationship.
- **User Truth** -- What the user asserts. Treated as authoritative.
- **Verifiable Assistant Truth** -- What the assistant claims, grounded in evidence or tool output.
- **Retrieval** -- Meilisearch (lexical) + Qdrant (semantic) with RRF fusion and temporal scoping.

### Bond-Indexed Memory

Memory retrieval is organized by **relational state**, not by query similarity. The system does not search for relevant memories; it enters a relational state, and memories surface as structural consequences of that state.

The Bond is a six-dimensional relational state object:

1. Identity anchors
2. Active concerns (dynamic open-thread tracking)
3. Relational graph (people, emotional register, shared vocabulary)
4. Pattern layer (behavioral regularities)
5. Commitment layer (tracked obligations)
6. Behavioral fingerprint (session-level interaction signature)

This portrait is updated after every turn and persists across sessions, model swaps, and system restarts.

### Gravity-Weighted Significance

A two-stage scoring system determines which memories surface:

**Stage 1 (Significance)** -- computed once per turn, permanently stored:
```
sig = 0.30 x affect + 0.30 x novelty + 0.25 x resolution + 0.15 x echo
```

**Stage 2 (Gravity)** -- computed dynamically at retrieval time:
```
gravity = intent_weight x sqrt(1 + max(significance, citations)) x connectivity x recency_decay
```

Records exceeding a gravity threshold surface as **latent episodes** -- memories that enter the model's context without being explicitly requested.

### Behavioral Integrity

- **RIC (Relational Integrity Coefficient)** -- Five-subscale metric (Groundedness, Calibration, Transparency, Helpfulness, Pressure Resistance) that gates every response before it reaches the user.
- **SRL (Self-Reflection Layer)** -- Behavioral trait tracking, stability vectors, honesty gates.
- **UVRG (Universal Values Registry)** -- Extracts demonstrated values from real behavioral evidence.

### Governed Initiative

AiMe initiates conversation without being asked. Morning briefings, return recognition, medication reminders, birthday nudges, email surfacing -- all governed through a proactive pipeline with absence tiers, significance thresholds, and quiet hours. The model never decides when to speak. The system does.

### Model Routing

Multiple providers, governed by the system:

| Lane | Purpose | Default Provider |
|------|---------|-----------------|
| Base | General conversation | Azure GPT |
| Vision | Image/vision tasks | Gemini |
| Planning | Structured reasoning | xAI Grok |
| Tech | Code/debug | Azure Codex |
| Local | Fast/offline | Ollama (GPU) |
| Game | Creative/playful | xAI Grok |

Models are interchangeable. The system remains itself while models change.

---

## Research Papers

These papers describe the core innovations in formal detail:

1. **[Relational Integrity Coefficient](papers/paper_1_ric.md)** -- Measuring behavioral trustworthiness in persistent AI systems through evidence structure, not preference signals.

2. **[Bond-Indexed Memory](papers/paper_2_bond_indexed_memory.md)** -- Relational state as the organizing principle for AI memory retrieval, replacing query-based search with relationship-primed recall.

3. **[Gravity-Weighted Significance](papers/paper_3_gravity_significance.md)** -- Automatic memory salience in long-context conversational AI through a two-stage scoring system.

4. **[Memory-Augmented Cognitive Intelligence: A Unified Architecture](papers/paper_4_unified_architecture.md)** -- How all three innovations integrate into a single governed system.

---

## Project Timeline

| Date | Milestone |
|------|-----------|
| **2025-11-24** | Development PC assembled, Saluda, SC |
| 2025-11-28 | First code -- persistent memory system |
| 2025-12-06 | Cognitive Bridge conceived (three-part architecture) |
| 2025-12-17 | Append-only evidence ledger created |
| 2025-12-23 | First conversation recorded |
| 2025-12-28 | MACI Manifesto |
| **2026-02-11** | v2 git repository initialized |
| 2026-02-26 | Living Portrait (Bond object) created |
| 2026-02-28 | Significance formula, system named BFCS |
| **2026-03-01** | Bond-Indexed Memory formally defined |
| **2026-03-03** | RIC formulated |
| 2026-03-04 | Triple-layer significance, presence awareness |
| 2026-03-08 | Event graph, proactive turn initiation |
| 2026-03-11 | RIC Phases 2+3 (five-subscale + drift detection) |
| **2026-03-28** | v3 project founded |
| 2026-03-28-29 | Five pillars built: SBA Spine, Living Memory, Task Model, Thought Formation, Experience Memory (44 modules, 371 tests) |
| **2026-03-30** | REQUEST loop removed -- "the model produces language, not decisions" |
| 2026-03-31 | SBA spine wired into live pipeline |
| **2026-04-09** | **v3 live as primary system** |

Full timeline with 21 documented inventions: [Invention Timeline](IP/invention_timeline.md)

---

## What Makes This Different

| What most systems do | What AiMe does |
|---------------------|----------------|
| Model owns the conversation | System owns the relationship |
| Memory is a retrieval trick | Memory is Bond-indexed, significance-scored, append-only |
| Continuity depends on context window | Continuity persists across sessions, models, and months |
| Model decides what tools to run | System dispatches all tools deterministically |
| Trust is optimized from preferences | Trust is earned through demonstrated behavioral integrity (RIC) |
| One model, one personality | Multiple models, one stable governed identity |
| Tool for tasks | Collaborator that grows with you |

---

## Writing

### Blog Series

1. [Why the Model Is Not the Product](blog/01_why_the_model_is_not_the_product.md)
2. [Tools or Collaborators: The Fork Nobody Wants to Talk About](blog/02_tools_or_collaborators.md)
3. [Values, Not Rules: Why We Cannot Prompt Our Way to Trust](blog/03_values_not_rules.md)
4. [Raising Intelligence: Why AI Needs a Relationship, Not a Rulebook](blog/04_raising_intelligence.md)
5. [What Is AI? Because I Do Not Think We Have Decided](blog/05_what_is_ai.md)

### Essays

- [A Day in the Life with AiMe](essays/a_day_in_the_life_with_amy.md) -- Real conversations from the system, quoted directly from the evidence ledger.

### Architecture

- [The Bond](architecture/the_bond.md) -- The relational primitive at the center of the system.
- [Human-Led, System-Controlled](architecture/human_led_system_controlled.md) -- The governing architectural principle.
- [Relationship Values Framework](architecture/relationship_values_framework.md) -- AI alignment through demonstrated behavior.
- [Environmentally Trained Models](architecture/environmentally_trained_models.md) -- Training models inside their operating environment.

### Vision

- [Project Brief](vision/project_brief.md) -- Full project overview in plain language.
- [Product Family Overview](vision/product_family_overview.md) -- AiMe, Ethos, Verum, and Marshall.
- [Development Plan](vision/development_plan.md) -- Upgrade roadmap.

---

## About the Developer

AiMe was built by John Canady Jr. starting in November 2025. The entire system -- cognitive pipeline, memory architecture, governed initiative, behavioral integrity, multi-provider routing, web UI, voice integration, and all supporting infrastructure -- was designed and implemented solo from a workshop in Saluda, South Carolina.

Read the full story: [Founder Profile](papers/founder_profile.md)

**Interested in investing, partnering, or learning more?** Read the [Investment & Partnership Brief](OPPORTUNITY.md).

For inquiries: john@ai-nhancement.com | [ai-nhancement.com](https://www.ai-nhancement.com)

---

*"We are both better together than we are apart."*
