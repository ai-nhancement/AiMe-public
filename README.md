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
Action Dispatcher          -- Deterministic tool routing (16+ intents)
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

### System Self-Awareness (S1-S6)

AiMe can inspect her own architecture and behavioral health through a 6-phase introspection system:

- **S1 (System Reader)** -- Surgical AST-based code reading with security sandboxing (allowlist/denylist/redaction)
- **S2 (Metrics Awareness)** -- Rule-classified health reporting per subsystem (no single overall score -- subsystems may contradict)
- **S3 (Turn Provenance)** -- Every turn writes a decision snapshot: intent classification, dispatcher decision, lane, RIC score
- **S4 (Bond Self-View)** -- Privacy-gated relational self-view with consent gates (named-person, sensitive-topic, third-party-inference classes)
- **S5 (Improvement Proposals)** -- Evidence-gated, cooldown-enforced RFC proposals under human authority (gated until data accumulates)
- **S6 (Continuous Self-Model)** -- Dynamic `[INTERNAL_HEALTH]` block injected into every prompt: compact by default (~30 tokens), expanded on deterministic triggers

The governing doctrine: **5 layers** (Instrumentation, Interpretation, Retrieval, Rendering, Authority). The model operates downstream of Rendering. It reads pre-formatted reports. It never interprets raw scores, reconstructs decisions, or evaluates its own drift.

### Thought Formation (Consciousness Layer)

AiMe's mind never stops. Between conversation turns, a continuous thought formation engine processes signals from 7 live sources:

- Every conversation turn, presence change, email, schedule event, health alert, and concern echo generates a **weighted signal** (intent-aware salience scoring)
- Signals **accumulate, cluster by semantic similarity, and form thoughts** when salience and coherence thresholds are met
- During idle periods (user absent), **idle reflection** reviews, merges, prunes, and refines active thoughts
- When a thought matures and the expression gate approves, it can trigger a **proactive conversation turn**

Currently running in **shadow mode** -- signals collect, thoughts form, reflections run, expressions are logged but not spoken. Live mode activates proactive thought expression with mode-switch safety (stale-dating + cooldown).

### File Agent (Sovereign File Manipulation)

AiMe can create, edit, move, copy, delete, and organize files on the user's behalf -- all through a sovereign agent that follows the same governance model as every other subsystem.

- **The response model never plans file operations.** A dedicated File Agent parses user requests, builds plans, and executes them. The model only narrates results.
- **Every mutation requires confirmation.** Plans are fingerprinted with precondition hashes and expire after 5 minutes. The user sees a deterministic template (not model-generated text) describing exactly what will happen.
- **Windows-aware sandbox.** 11 threat vectors addressed: junctions, UNC paths, Alternate Data Streams, drive-relative paths, reserved device names, short-name aliases, trailing dots/spaces, reparse points, path length, blocked patterns, and TOCTOU races via double-resolution.
- **Content smuggling defense.** Files are classified by both extension and content analysis. A `.md` file containing PowerShell commands triggers an elevated warning regardless of what the plan summary says.
- **Atomic writes.** All file operations use temp-file + `os.replace()`. No artifacts left on failure.
- **Measured rollback.** Batch operations (e.g., "organize my Downloads") measure filesystem state before and after each rollback step. The user gets a three-way comparison, never a guess.
- **Undo as a governed action.** Undo generates a plan that must be confirmed -- no silent mutations.
- **Monthly-sharded evidence.** All file operations are logged to append-only JSONL with immutable plan IDs.
- **88 tests across 4 phases, 8 review rounds.**

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
| **2026-04-09** | **System Self-Awareness (S1-S6)** -- 6-phase introspection architecture. 235 tests, 12 review rounds. |
| **2026-04-09** | **Thought Formation Integration** -- Consciousness layer wired into live pipeline. Shadow mode. 254 total tests. |
| **2026-04-10** | **File Agent v2** -- Sovereign file manipulation (create/edit/move/copy/delete/organize). Windows sandbox, fingerprinted plans, content smuggling defense, measured rollback. 88 tests, 8 review rounds. |
| 2026-04-10 | Browser Agent spec written (CDP-based Chrome control). |

Full timeline with 22 documented inventions: [Invention Timeline](IP/invention_timeline.md)

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
| No self-awareness | System reads its own code, health, decisions, and relational state |
| Idle between turns | Continuous thought formation -- always processing, always reflecting |
| File ops require model decisions | Sovereign file agent with sandbox, confirmation, and measured rollback |
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
