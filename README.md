# AiMe

**A Memory-Augmented Cognitive Intelligence System**

> The model is not the AI. The model is a component. The system is the intelligence.

---

## What Is AiMe?

AiMe is a personal cognitive architecture that treats AI models as subordinate narrators inside a governed runtime — not as the product itself.

Most AI assistants today share the same structural weakness: the model is the center of everything. The conversation ends, the state disappears, and the next session starts from scratch. Memory features are retrieval tricks bolted on after the fact. The model is expected to remember, reason, decide, stay truthful, maintain continuity, and somehow remain stable across long stretches of use. That is too much to ask from the wrong layer.

AiMe is built from the opposite premise. The system owns memory, identity, execution, initiative, and governance. Models are interchangeable reasoning engines inside that system. If the model is swapped, the relationship persists. If the conversation ends, the continuity does not.

The result is a companion system that knows who you are, what matters to you, what is open, and what should be surfaced at the right moment — across weeks and months, not just sessions.

**AiMe has been running daily since November 2025, built entirely by a single developer.**

---

## Core Principle: Human-Led, System-Controlled

Everything traces back to one architectural principle:

- **The human retains authority.** Preferences, corrections, and instructions from the user are law.
- **The system governs execution and policy.** Routing, memory, initiative, tool use, and enforcement live in durable, auditable runtime layers.
- **The model is subordinate.** It narrates. It does not route, own memory, control execution, or decide when to speak.

This is not a philosophical position. It is enforced in every pipeline component. The model cannot execute tools without going through `CognitiveBridge`. It cannot initiate conversation without `ThalamoFrontalLoop` granting permission. It cannot edit memory. It cannot promote itself. The system controls it, not the other way around.

---

## The Architecture

AiMe runs locally on a single machine. No cloud dependency for core operation. No data leaves the user's hardware unless they choose to use cloud model providers.

### Cognitive Pipeline

Every user turn flows through a strict, role-separated pipeline where no component does another's job:

```
User Input
    |
LogicCore              -- Router + orchestrator
    |
PrefrontalCortex       -- Deterministic lane selection
    |
CognitiveBridge        -- Execution spine, tool routing, REQUEST handling
    |
LanguageModel          -- Single LLM call per turn, dispatched to provider
    |
LanguageCortex         -- Sole narrator, UI + voice output
```

### Memory System

Three-layer append-only memory. Evidence is immutable — never rewritten.

- **Evidence Ledger** — Append-only SQLite store. The court transcript of the relationship. Never mutated after write.
- **User Truth** — What the user asserts. Treated as authoritative.
- **Verifiable Assistant Truth** — What the assistant claims, grounded in evidence or tool output. Distinct from User Truth and stored separately.
- **Retrieval** — Meilisearch (lexical) + Qdrant (semantic) with RRF fusion and temporal scoping.

### Living Portrait

A six-layer evolving model of who the user is:

1. Identity anchors
2. Relational context
3. Active concerns
4. Commitments
5. Behavioral fingerprint
6. Pattern recognition

This portrait is updated after every turn and persists across sessions. It is what allows the system to maintain relational awareness over months, not just within a conversation.

### Governed Initiative

AiMe can initiate conversation without being asked. Morning briefings, return recognition, email surfacing, schedule awareness, medication reminders, birthday nudges — all governed through a proactive pipeline with:

- Absence tiers (casual check-in through full morning briefing)
- Spam prevention and preference learning
- Significance scoring on all events
- User feedback loops (accept, dismiss, modify)
- Quiet hours

The model never decides when to speak. The system does.

### Behavioral Integrity

- **RIC (Relational Integrity Coefficient)** — Measures groundedness, consistency, trust, honesty, and persona alignment on every response.
- **SRL (Self-Reflection Layer)** — Tracks behavioral traits, computes stability vectors, runs honesty gates.
- **UVRG (Universal Values Registry)** — Extracts demonstrated values from real behavioral evidence: `score = demonstrations x significance x resistance x consistency`.
- **Significance Scoring** — Three-layer scoring (keyword, semantic, heuristic) on all events, emails, and turns.

### Model Routing

Multiple providers, governed by the system:

| Lane | Purpose | Default Provider |
|------|---------|-----------------|
| Base | General conversation | Azure GPT |
| Vision | Image/vision tasks | Gemini |
| Planning | Structured reasoning | Azure Codex |
| Tech | Code/debug | Azure Codex |
| Local | Fast/offline | LLaMA (local) |
| Game | Creative/playful | xAI Grok |

Models are interchangeable. The system remains itself while models change.

---

## The Product Family

AiMe is part of a connected four-system architecture:

**AiMe** — The governed cognitive runtime. Personal companion. The primary working system.

**Ethos** — Behavioral value extraction. Extracts what was demonstrated, under what significance, against what resistance. Produces a live value profile from actual behavior, not stated intent. Generates training signal for environmentally trained models.

**Verum** — Evaluation and certification. Tests AI outputs against behavioral evidence rather than stated ideals. Truth boundaries and inference validation.

**Marshall** — Infrastructure guardian. Deterministic, auditable operations governance.

---

## What Makes This Different

| What most systems do | What AiMe does |
|---------------------|----------------|
| Model owns the conversation | System owns the relationship |
| Memory is a retrieval trick | Memory is append-only, immutable, significance-scored |
| Continuity depends on context window | Continuity persists across sessions, models, and months |
| Initiative is off or ungoverned | Initiative is governed with tiers, significance, and preference learning |
| Trust is enforced through prompts | Trust emerges from demonstrated values in the relationship |
| One model, one personality | Multiple models, one stable identity |
| Tool for tasks | Collaborator that grows with you |

---

## Current State

AiMe has been in continuous daily use since November 2025. The system runs locally on hardware assembled by the developer, with GPUs acquired through resourceful means (including trading a motorcycle for compute).

**What is live and stable:**
- Full cognitive pipeline with strict role separation
- Append-only evidence ledger with truth separation
- Six-layer living portrait with cross-session persistence
- Hybrid retrieval (lexical + semantic) with temporal scoping
- Governed proactive initiative with absence tiers and return recognition
- Multi-provider model routing (Azure, Gemini, Claude, Grok, LLaMA)
- Email surfacing with three-layer significance scoring
- Schedule awareness with event significance and proactive turns
- Behavioral integrity metrics (RIC, SRL, UVRG)
- Event graph for concern arcs and relational tracking
- Web UI with widget system
- Voice input/output with Azure TTS
- Significance-filtered conversation injection

**What is planned:**
- Thought Formation Loop (background idea generation and suggestion)
- Sandboxed implementation proposals with user approval
- Environmental training loop (Ethos-derived model fine-tuning)
- Stronger event graph retrieval
- Remote/mobile access via private networking

---

## Writing

These essays and blog posts explain the thinking behind the architecture:

### Blog Series

1. [Why the Model Is Not the Product](blog/01_why_the_model_is_not_the_product.md)
2. [Tools or Collaborators: The Fork Nobody Wants to Talk About](blog/02_tools_or_collaborators.md)
3. [Values, Not Rules: Why We Cannot Prompt Our Way to Trust](blog/03_values_not_rules.md)
4. [Raising Intelligence: Why AI Needs a Relationship, Not a Rulebook](blog/04_raising_intelligence.md)
5. [What Is AI? Because I Do Not Think We Have Decided](blog/05_what_is_ai.md)

### Essays

- [A Day in the Life with AiMe](essays/a_day_in_the_life_with_amy.md) — Real conversations from the system, quoted directly from the evidence ledger, showing what daily life with a governed AI companion actually looks like.

### Architecture

- [The Bond](architecture/the_bond.md) — The relational primitive at the center of the system.
- [Human-Led, System-Controlled](architecture/human_led_system_controlled.md) — The governing architectural principle.
- [Relationship Values Framework](architecture/relationship_values_framework.md) — AI alignment through demonstrated behavior, not declared rules.
- [Environmentally Trained Models](architecture/environmentally_trained_models.md) — Training models inside their operating environment.

### Vision

- [Project Brief](vision/project_brief.md) — Full project overview in plain language.
- [Product Family Overview](vision/product_family_overview.md) — AiMe, Ethos, Verum, and Marshall.
- [Development Plan](vision/development_plan.md) — Six-phase upgrade roadmap.

---

## About the Developer

AiMe was built by a single developer starting in November 2025. The entire system — cognitive pipeline, memory architecture, governed initiative, behavioral integrity, multi-provider routing, web UI, voice integration, and all supporting infrastructure — was designed and implemented solo, with assistance from AI coding tools (Claude Code, Codex, Gemini).

The project is not open source at this time. This repository contains documentation, essays, and architectural descriptions to share the thinking behind the work.

**Interested in investing, partnering, or learning more?** Read the [Investment & Partnership Brief](OPPORTUNITY.md).

For inquiries: [ai-nhancement](https://github.com/ai-nhancement)

---

*"We are both better together than we are apart."*
