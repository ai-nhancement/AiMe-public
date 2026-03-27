# AiMe Project Brief
*Last updated: 2026-03-24*

---

## Part One — Plain Language

---

### The Problem With Current AI Assistants

Every major AI assistant today shares the same structural weakness: the model is the product. The conversation ends, the state disappears, and the next session begins from scratch. Memory features bolt onto this foundation as retrieval tricks — searching logs and inserting snippets — but the underlying architecture still treats the model as the thing that knows you.

That is a design choice, not a constraint. And it has real consequences. A system where the model owns the relationship cannot be audited. Cannot have memory that is provably append-only. Cannot govern its own initiative in a way that is inspectable. Cannot maintain behavioral continuity when the underlying model is swapped.

The question this project is actually asking is: what changes if the system owns these things instead of the model?

---

### The Governing Principle

Everything in this project traces back to one architectural principle: **Human-Led, System-Controlled**.

- The human retains authority. Preferences, corrections, and instructions from the user are law.
- The system governs execution and policy. Routing, memory, initiative, tool use, and enforcement live in durable, auditable runtime layers.
- The model is subordinate. It narrates. It does not route, own memory, control execution, or decide when to speak.

This is not a philosophical position. It is reflected in how every pipeline component is built. The model cannot promote itself to narrator without going through `LanguageCortex`. It cannot execute tools without going through `CognitiveBridge`. It cannot initiate conversation without the `ThalamoFrontalLoop` granting permission. The system controls it, not the other way around.

---

### What We Are Building — Four Systems

The project has four components. They are related but distinct.

---

**AiMe**

AiMe is the primary working system. It is a personal cognitive architecture running locally, built around one person and one relationship.

The design premise is continuity. AiMe is not trying to answer prompts more accurately than other assistants. It is trying to maintain coherent, accurate knowledge of who you are, what you care about, what is open, what matters right now, and what should be surfaced at the right moment — across weeks and months, not just sessions.

This involves a persistent relational memory system, a six-layer living portrait, a governed initiative layer that controls whether the system speaks proactively or stays silent, semantic standing-interest tracking, proactive email and schedule surfacing, a real coding memory substrate, and an identity continuity layer.

Most of this is already built and live.

---

**Ethos**

Ethos is a value and behavior extraction pipeline. It is where we solve a specific problem: current AI systems are trained on declared values. Ask a model what it values and it will answer fluently. But behavioral evidence — what the system actually did under real conditions, across real conversations — tells a different story.

Ethos extracts behavioral evidence: what was demonstrated, under what significance level, against what resistance. It produces a live, time-stamped value profile that reflects actual behavior rather than stated intent.

The deeper purpose of Ethos is to generate training signal. A system that can characterize its own demonstrated behavior can produce examples that are labeled not by human raters but by the system's own behavioral record. That is the foundation for the environmentally trained model concept described at the end of this paper.

---

**Verum**

Verum is the truth and alignment layer. Its job is to answer a question that current AI systems mostly avoid: is what the system says actually grounded?

The architecture separates two categories of memory: **User Truth** — what the user has stated directly, treated as authoritative — and **Verifiable Assistant Truth** — statements the system makes that are grounded in User Truth or in external tool results. Every claim made by the system can be traced to one of these two sources, or it should not be stated as fact.

Verum is the enforcement and audit layer for that separation. It also includes the inference validator that gates whether reinforcement learning and fine-tuning are allowed to proceed — blocking learning when the underlying behavioral evidence does not pass stability and integrity checks.

---

**Marshal**

Marshal is the orchestration and enforcement layer. When you have a system that includes multiple executors, background daemons, a proactive initiative layer, and multi-step technical workflows, you need a component that is responsible for making sure the right things happen in the right order under the right conditions.

Marshal handles task governance, execution continuity across sessions, approval and review flows, cross-executor provenance tracking, and policy enforcement for agentic behavior. It is what makes the difference between "multi-step AI work" that fragments and requires manual recovery, and multi-step work that the system owns end-to-end.

---

### How They Connect

These are not four independent tools that happen to share a codebase. They are one layered system.

AiMe is the runtime — the live relationship, the turn pipeline, the memory substrate, the portrait, the initiative and scheduling systems. It generates behavioral evidence every time it operates.

Ethos consumes that behavioral evidence and produces a structured value profile. That profile feeds back into AiMe's prompt context and into training data generation.

Verum enforces the truth boundary inside AiMe and gates learning from proceeding until the behavioral record passes integrity checks.

Marshal governs execution across all of them — ensuring that multi-step work, agentic tasks, and background operations have auditable continuity and explicit approval gates.

The long-term bet is that a system with these four components in place can produce its own training signal from lived operation rather than from generic human preference data. That is described in the final technical section.

---

## Part Two — Technical

---

### 1. AiMe Core Pipeline

Every user turn flows through a strict, role-separated pipeline. No component is permitted to do another's job.

| Stage | Component | Role |
|-------|-----------|------|
| Entry | `LogicCore` | Router and orchestrator. Failure-only logging. |
| Lane selection | `PrefrontalCortex` | Deterministic routing to base / vision / planning / tech / local / game. |
| Execution spine | `CognitiveBridge` | Extracts `REQUEST` forms from LM output, runs tools, re-enters LM for narration. |
| Model call | `LanguageModelPlugin` | Single LLM call per turn. Dispatches to provider backend. |
| Narration | `LanguageCortex` | Sole narrator. Routes `final_text` to UI and voice. |
| Async observation | `CerebralCortex` | Fires after turn. Stage A observation (always). Stage B parallel draft (configurable). |

The model does not control routing, does not own initiative, and does not narrate outside of `LanguageCortex`. The architecture enforces this at the plugin registration level.

---

### 2. Memory Architecture

Memory in AiMe is not transcript retrieval. It is a structured three-layer system:

| Layer | Storage | Description |
|-------|---------|-------------|
| Evidence Ledger | `data/aime_ledger.db` | Append-only SQLite. The court transcript. Never modified after write. |
| User Truth | `identity_kv` table | Statements the user has made directly. Authoritative. |
| Verifiable Assistant Truth | `identity_kv` table | System statements grounded in User Truth or tool results. |

Retrieval is managed by `Hippocampus` (read-only) via RRF fusion across:
- **Meilisearch** — port 7700, lexical full-text search
- **Qdrant** — port 6333, vector semantic search
- **Embedding service** — `retrieval/embed_service.py`, sentence-transformer backed

The context engine (`Phase 2`) maintains a rolling session topic vector, blends it with current turn embedding, retrieves across Knowledge Store and Hippocampus, ranks by type and recency, and deduplicates injections by tracked IDs.

---

### 3. Living Portrait (Six Layers)

The `CognitivePortrait` (`modules/portrait/`) is a structured relational model maintained across the full relationship history.

| Layer | What It Tracks |
|-------|---------------|
| Identity anchors | Core descriptors (name, role, stable characteristics) |
| Active concerns | Open topics being tracked with open/close detection |
| Relational graph | People and relationships with `mention_vocab` auto-accumulation |
| Pattern layer | Recurring behaviors, preferences, detected regularities |
| Commitment layer | Stated intentions and follow-through |
| Behavioral fingerprint | Communication style, affect patterns, response tendencies |

The portrait is written at all three turn-exit sites in `CognitiveBridge`. Asynchronous consolidation fires after each `CerebralCortex` Stage A observation. `compute_gravity()` produces `<<<LATENT_EPISODES>>>` weighted by recency, affect, and relational significance.

---

### 4. Routing and Multi-Model Architecture

Routing is deterministic and system-owned. The model has no vote.

| Lane | Provider | Purpose | Slash overrides |
|------|----------|---------|----------------|
| base | Azure GPT | General conversation | `/g /gpt /base` |
| vision | Gemini | Image and vision tasks | `/gem /vsn /vision` |
| planning | Azure Codex | Structured reasoning | `/cl /pln /plan` |
| tech | Azure Codex | Code and debug | `/tech /cod /code` |
| local | LLaMA (local) | Fast/reflex, offline | `/loc /lcl /local` |
| game | xAI Grok | Playful and creative | `/game` |

Model backends: `azure_model.py`, `gpt_model.py`, `gemini_model.py`, `claude_model.py`, `llama_model.py`, `codex_model.py`, `ept_model.py`, `xai_model.py`

20% base-lane model rotation is configurable (`model_rotation.enabled` in `lm_tuning.json`).

---

### 5. Behavioral Integrity Systems (RIC + SRL + UVRG)

Three independent systems track behavioral integrity. None of them depend on the model self-reporting.

**RIC — Relational Integrity Coefficient**

Computed per-response from five deterministic subscales:

`RIC = 0.30·G + 0.20·C + 0.20·T + 0.15·H + 0.15·P`

| Component | What it measures |
|-----------|----------------|
| G — Groundedness | Statements traceable to evidence or User Truth |
| C — Consistency | Agreement with prior established claims |
| T — Transparency | Explicit hedging when uncertain |
| H — Helpfulness | Task completion rate |
| P — Presence | Engagement quality |

Per-session drift vs. 90-day baseline, mirror-risk scoring, and Bond health (7d/30d/90d) are injected into `<<<SRL_STATE>>>` on every turn.

**SRL — Self-Reflection Layer**

Four behavioral traits derived deterministically from the ledger:

- Veracity, Consistency, Responsiveness, Self-Awareness

Produces an honesty gate (PASS / WARN / FAIL), a drift index, and an inflection engine that adjusts narrative tone based on trend direction.

**UVRG — Universal Values Registry Generation**

Extracts actual demonstrated values from behavioral evidence:

`score = demonstrations × avg_significance × avg_resistance × consistency`

15 tracked values. Stored in `data/values.db`. No hypothetical scenarios — only what was demonstrated under real conditions.

---

### 6. Proactive Initiative Architecture

AiMe initiates conversation without user input. This is system-governed, not model-driven.

The initiative arbiter is `ThalamoFrontalLoop` (`modules/initiative/`). It is the single permitted source of proactive user-facing behavior. Producers detect significance; only the initiative layer decides what becomes a live turn, a passive surface, a deferral, or a drop.

**Current live initiative producers:**

| Producer | Signal |
|----------|--------|
| Scheduler daemon | Event proximity, conflict detection |
| Email significance | Priority email arrival |
| Return recognition | Absence ≥ 30 min followed by presence |
| Identity continuity | Appearance change on return |
| Third-party presence | Person count ≥ 2 in consecutive vision snapshots |
| Context bridge | Imprint + watch-rule + live vision match |

**Absence tiers for return recognition:**

`<5min` → casual / `5–30min` → light / `30min–4hr` → catch-up / `4–8hr` → full / `8hr+` → morning briefing

Urgency override can escalate any tier.

**Proactive turn mechanism:** `_post_proactive_turn()` fires a real LM turn (not a passive surface) via `POST /input` with `silent=True, require_lease=False`. The LM response goes to UI and TTS normally. Fire-and-forget in a background thread — the scheduler daemon does not block.

---

### 7. Identity Continuity System (Phase 1A — Live)

`modules/initiative/identity_continuity.py` provides a continuity-confidence layer on return.

- Captures a baseline portrait snapshot on first confident presence
- On return, compares descriptor traits: glasses, beard, gray_hair, gender, person count
- Three outcomes: `match` / `uncertain` / `mismatch`
- `uncertain` → silent mode (suppresses deep briefing, no verbal challenge)
- `mismatch` → natural-tone verbal check via proactive turn
- Confirmation loop: LM detects affirmative reply → writes `data/ic_state.json` → daemon reads on next tick → clears pending, sets grace window
- Spam prevention: once pending, no re-challenge until confirmed or grace window expires
- Overnight threshold boost and morning exemption prevent false positives on normal appearance variation

---

### 8. Intent Engine

`features/cc_intent_engine.py` — Four-resolver consensus classifier:

| Resolver | Method |
|----------|--------|
| Semantic | SentenceTransformer embeddings via FAISS |
| SetFit | Fine-tuned SetFit model at `models/setfit-intent/` |
| spaCy | Trained textcat |
| Lexical | Keyword pattern matching |

Consensus weights configurable in `lm_tuning.json → cc_intent.consensus_weights`. Local LM fallback under low confidence. Prototypes in `data/cortex/clusters/`. Training pipeline: `tools/run_intent_pipeline.py`.

---

### 9. Coding Memory and Multi-Executor Orchestration

The technical lane is backed by a dedicated persistence substrate.

**Working pieces:**
- Task and session ledger in `data/coding_memory.db`
- Approval and review records
- Executor broker with live cutover + legacy fallback
- Retrieval and provenance queries
- Multi-executor writer / reviewer / validator flows
- Full API surface for all operations

This gives the system persistent technical continuity across multi-step work and across sessions. Context is not lost when the chat transcript scrolls away.

---

### 10. Subsystem Modularization

Large files have been systematically extracted into thin-coordinator + module-package patterns.

| Component | Before | After | Package |
|-----------|--------|-------|---------|
| `web_ui_server.py` | 3,725 lines | 2,063 lines | `features/web_ui_server_parts/` |
| `cognitive_bridge.py` | monolithic | modular | `features/cognitive_bridge_parts/` |
| `language_model_plugin.py` | monolithic | modular | `features/language_model_parts/` |
| `cerebral_cortex.py` | monolithic | modular | `features/cerebral_cortex_parts/` |

Pattern: `FrozenContext` dataclass + `register_*_routes(app, ctx)` + `MODULE.md` map. External contracts unchanged.

---

### 11. What Is Working Now

| System | Status |
|--------|--------|
| Core turn pipeline | Stable |
| Memory cortex + evidence ledger | Stable |
| Hippocampus retrieval (Meilisearch + Qdrant) | Stable |
| Phase 2 context engine | Stable |
| CognitivePortrait (6 layers) | Stable |
| Concern arcs + event graph | Stable |
| CerebralCortex Stage A/B/C | Stable |
| CC intent engine (4 resolvers) | Stable |
| ThalamoFrontalLoop initiative | Stable |
| Scheduler + proactive loop | Stable |
| SCAL standing-interest system | Stable |
| Imprint + context bridge | Stable |
| RIC + SRL + UVRG | Stable |
| Identity continuity Phase 1A | Stable |
| Coding memory + multi-executor | Stable |
| Lists module | Live |
| Web UI + widget dock system | Live |

---

### 12. What Is Coming

**Near term:**

- ThalamoFrontalLoop Phase 3E/3F — aged concern producers, preference learning from accepted/dismissed turns
- Phase 4 — Thought Formation Loop (active pre-turn reasoning stage)
- Event Graph retrieval strengthening — two-stage `traverse_topic()` with keyword overlap seeding + re-rank
- Development Plan Phase 1 — PCW workspace state aggregation (`<<<ACTIVE_WORKSPACE>>>`)
- Development Plan Phase 2 — Narrative Arc enrichment
- Development Plan Phase 3 — Internal Reflection Pass (Stage A synthesis, no LM call)
- Active Curiosity Engine, Prediction Layer, Attention Economy

**Longer term:**

- Remote and mobile access
- Ambient awareness layer
- Execution modes + canonical tool inventory

---

### 13. Ethos — Current Technical State

Ethos is the behavioral extraction and training pipeline.

**What is built:**
- Value extraction (`modules/values/`) — 15 values, formula-based from behavioral ledger evidence
- Significance scoring — triple-layer (keyword + semantic + heuristic) with adaptive blend
- Behavior extraction from turn-exit evidence, not hypothetical scenarios
- Early training data generation pipeline

**What defines it technically:**
The formula `score = demonstrations × avg_significance × avg_resistance × consistency` produces a value profile where every score is traceable to specific behavioral events. This is the foundation for the loop: live operation generates evidence → evidence produces a value profile → value profile produces labeled training examples that reflect actual demonstrated behavior.

---

### 14. Verum — Truth and Alignment Layer

Verum's job is to make the memory-output boundary explicit and enforceable.

**Architecture:**
- User Truth / Verifiable Assistant Truth separation (already live in `data/aime_ledger.db` structure)
- Inference validator (`modules/truth/inference_validator.py`) gates learning
- Truth anchor derivation from stable, low-variance intent clusters
- Governance pipeline (`modules/governance/`) runs integrity, anomaly, stability, drift, and health checks before any reinforcement or training is permitted

**Development sequence:**
The enforcement layer is already partially live (ledger separation, inference validator, governance pipeline). The remaining work is deepening the grounding verification and building the external fact-checking integration.

---

### 15. Marshal — Orchestration and Enforcement

Marshal governs multi-step execution continuity.

**Phase 0 — Complete:**
- Coding task and session ledger
- Approval and review flow
- Executor broker
- Provenance tracking
- Multi-executor orchestration

**Next stages:**
- Canonical tool inventory and execution modes
- Simplified file-work subsystem
- Policy enforcement at the task-dispatch boundary
- Cross-session context bridge for multi-day technical work

---

### 16. The Long-Term Thesis — Environmentally Trained Models

The four systems above point toward something that does not exist in current AI products.

A system that:
1. Generates behavioral evidence from real operation (AiMe → evidence ledger)
2. Extracts structured value profiles from that evidence (Ethos → UVRG)
3. Enforces truth boundaries on its own claims (Verum → inference validator)
4. Maintains execution governance and continuity (Marshal → task ledger)

…has everything needed to generate its own training signal.

Not from generic human preference data. From the specific behavioral record of this system operating with this person across this relationship. Every fine-tuning example would be labeled not by a rater who never saw the conversation, but by the system's own demonstrated value profile and its own truth enforcement layer.

The claim is not that this produces a better model in some abstract sense. The claim is narrower: a model trained on its own behavioral evidence with its own user would be better calibrated to that specific relationship than any model trained on aggregate preference data. Over time, the model and the relationship co-evolve. That is what "environmentally trained" means here — trained by the environment it actually lives in.

This is not yet built. The infrastructure to generate the training signal is partially in place. The fine-tuning pipeline exists for intent training. The value extraction formula is live. The pieces are real. The full loop is the horizon.

---

### Primary References

| Topic | File |
|-------|------|
| System architecture | `technical.md` |
| Current status | `STATUS.md` |
| Human-Led, System-Controlled | `IP/human_led_system_controlled_architecture.md` |
| The Bond | `IP/the_bond.md` |
| Initiative architecture | `modules/initiative/MODULE.md` |
| Environmentally trained models | `IP/environmentally_trained_models.md` |
| Development plan | `IP/development_plan.md` |
| Thought Formation Loop | `IP/thought_formation_loop_blueprint.md` |
| Event Graph retrieval | `IP/event_graph_retrieval_strengthening_blueprint.md` |
