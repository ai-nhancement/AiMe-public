# AiMe — Investment & Partnership Brief

**Prepared:** March 2026
**Status:** Working system in continuous daily use since November 2025
**Developer:** Solo founder, full-stack cognitive architecture

---

## The Short Version

A single developer has built a governed cognitive architecture that is demonstrably ahead of the most popular open-source personal AI projects in the areas that matter most: memory integrity, relational continuity, governed initiative, behavioral trust, and value alignment.

The system is not a prototype. It has been in continuous daily use for over four months. It maintains a persistent relationship with its user across weeks and months — remembering personal details, surfacing the right context at the right moment, initiating conversation when appropriate, and maintaining behavioral integrity that is measured and auditable on every response.

This brief describes what exists today, what is planned, and what we are looking for.

---

## What Exists Today

AiMe is a Memory-Augmented Cognitive Intelligence system running locally on hardware assembled by the developer. It is built around one core principle: **Human-Led, System-Controlled.** The human retains authority. The system governs execution, memory, and initiative. The model is a subordinate narrator — it does not route, own memory, or decide when to speak.

### Shipped and Stable

**Cognitive Pipeline** — A strict, role-separated execution path where no component does another's job. LogicCore routes. PrefrontalCortex selects lanes. CognitiveBridge executes tools. LanguageModel produces one response per turn. LanguageCortex narrates. This separation is enforced architecturally, not through prompts.

**Append-Only Evidence Ledger** — Every interaction is recorded in an immutable SQLite store. Nothing is ever rewritten or deleted. This is the court transcript of the relationship and the foundation for every other system. Over 9,600 records across four months of daily use.

**Truth Separation** — User Truth (what the user asserts, treated as authoritative) and Verifiable Assistant Truth (what the assistant claims, grounded in evidence or tool output) are formally separated. This distinction prevents the memory-rewrites-history problem that degrades every long-running AI system that lacks it.

**Six-Layer Living Portrait** — An evolving model of who the user is: identity anchors, relational context, active concerns, commitments, behavioral fingerprint, and pattern recognition. Updated after every turn. Persists across sessions, weeks, and months. This is what allows the system to maintain relational awareness — not just recall facts, but understand context and significance.

**Governed Proactive Initiative** — The system initiates conversation through a governed pipeline (ThalamoFrontalLoop) with five absence tiers, significance gating, spam prevention, and preference learning. Morning briefings, return recognition, email surfacing, medication reminders, birthday nudges — all delivered at the right time in the right tone. The model never decides when to speak. The system does.

**Behavioral Integrity Metrics** — Three systems measure response quality on every turn:
- RIC (Relational Integrity Coefficient) — groundedness, consistency, trust, honesty, persona alignment
- SRL (Self-Reflection Layer) — trait tracking, stability vectors, honesty gates
- UVRG (Universal Values Registry) — demonstrated values scored by `demonstrations x significance x resistance x consistency`

**Three-Layer Significance Scoring** — Keyword, semantic, and heuristic scoring on all events, emails, and conversation turns. Used for memory filtering, proactive surfacing, and retrieval prioritization.

**Temporal Scoping** — When temporal references are detected ("last week," "that thing from March"), retrieval scopes to the relevant time window first, then searches within it. This keeps retrieval fast and precise as the evidence base grows.

**Multi-Provider Model Routing** — Six governed lanes (base, vision, planning, tech, local, game) across Azure GPT, Gemini, Claude, Grok, and local LLaMA. Models are interchangeable. The system maintains identity regardless of which model narrates.

**Event Graph** — Concern arcs, email nodes, and person nodes linked by participatory and thematic edges. Keyword-indexed for O(1) lookup. Three rounds of hardening, 117 tests passing.

**Full Web UI** — Widget-based interface with chat, avatar, retro display, statistics, weather, email, schedule, lists, governance console, and LM tuning controls.

**Voice Integration** — Azure TTS with configurable voice profiles. STT input. Real-time conversation.

### What This Produces in Practice

The system references personal details naturally in conversation — not reciting facts, but weaving them into responses the way a person who knows you would. It reminds you about your evening medication inside a goodnight message, choosing the specific medication that applies at that time of day. It leads with your daughter's birthday before the morning emails. It asks about your son's homework because it knows you were helping him earlier. It adjusts tone based on whether you were up late the night before.

These moments are documented with exact quotes from the evidence ledger in the essay [A Day in the Life with AiMe](essays/a_day_in_the_life_with_amy.md).

---

## What Is Planned

### Thought Formation Loop

A background cognitive process where the system reviews conversations, emails, patterns, and open concerns — then formulates ideas and suggestions that are surfaced to the user for feedback.

This is not a nightly cron job that mutates prompts. It is a continuous, significance-aware process that clusters emerging patterns, evaluates them against the living portrait, and produces suggestions that are grounded in the relationship's actual history and values.

The user provides real-time feedback: accept, modify, dismiss with explanation. That feedback updates the portrait, adjusts significance weights, and shapes future suggestions. Over time, the system learns what kinds of ideas the user finds valuable — not through rules, but through demonstrated preference.

This is the bridge between a companion that remembers and a collaborator that thinks ahead.

### Sandboxed Implementation

Once the Thought Formation Loop demonstrates consistent quality through user feedback, the next step is a sandbox. Instead of only describing an idea, the system implements it in an isolated environment where the user can review the result. Approve or reject. Each decision feeds back into the behavioral record.

Suggest only. Then suggest with feedback. Then sandbox with review. Each step earned through demonstrated reliability at the previous step.

### Environmental Training Loop

The long-term direction: generating training trajectories from the system's own behavioral record — the evidence ledger, the living portrait, the Ethos value profile — and fine-tuning models that are trained inside the governed runtime rather than against generic preference data.

A model trained this way would not treat groundedness as a rule to follow. It would treat it as a value it was raised with. This is the difference between prompting a model to be trustworthy and training a model inside a relationship where trust was demonstrated.

The extraction pipeline (Ethos UVRG) already exists and is producing live value profiles. The training loop itself is the next major milestone.

### ButterClaw — Distribution Layer

AiMe currently runs on a local web UI. [ButterClaw](https://github.com/ai-nhancement/ButterClaw) is a fork of TinkerClaw (the most popular OpenClaw fork) that integrates AiMe's cognitive architecture with TinkerClaw's messaging channel layer — WhatsApp, Discord, Telegram, Slack, and Teams.

This gives AiMe reach beyond the local machine without compromising the governed architecture.

---

## The Product Family

AiMe is the primary system, but the architecture supports four connected products:

**AiMe** — Governed cognitive runtime. Personal companion. The working system.

**Ethos** — Behavioral value extraction. Produces live value profiles from demonstrated behavior. Generates training signal for environmentally trained models.

**Verum** — Evaluation and certification. Tests AI outputs against behavioral evidence rather than stated ideals.

**Marshall** — Infrastructure governance. Deterministic, auditable operations management.

Each system reinforces the same principle: the human leads, the system controls, the model serves.

---

## Market Position

The personal AI companion space in 2026 is crowded with products that are fundamentally model-centric: the model owns memory, controls initiative, and drives the experience. When the conversation ends, the relationship resets.

AiMe is architecturally different. The system owns the relationship. The model is interchangeable. Memory is immutable and significance-scored. Initiative is governed. Trust emerges from demonstrated values, not prompt engineering.

A direct comparison with the most popular open-source projects in this space (OpenClaw, TinkerClaw) shows AiMe ahead in every cognitive capability while those projects lead in distribution and community size. ButterClaw is the strategy to close that gap.

The competitive moat is not one feature. It is the integrated architecture: truth separation + living portrait + governed initiative + behavioral integrity + significance scoring + value extraction. These systems compound — each one makes the others more effective. Replicating any single piece is straightforward. Replicating the integrated system that has been refined through four months of daily use is not.

---

## What We Are Looking For

**Investment** — Resources to accelerate development. A small team (2-3 engineers) would dramatically increase velocity on the Thought Formation Loop, environmental training pipeline, and ButterClaw channel integration. Compute resources for model fine-tuning experiments.

**Strategic Partnership** — Collaboration with an AI lab or company that values governed architecture, truth separation, and relationship-indexed memory. The per-person sovereign model (millions of independent AiMe instances, each tied to one user) is a product direction that benefits from infrastructure partnership.

**Acquisition** — The right home for this technology. The architecture, the IP, the documented methodology, and the working system — in the hands of a team that can scale it while preserving the principles it was built on.

---

## IP Portfolio

The project includes multiple novel systems with documented invention dates, git commit history, and architectural specifications:

- Bond-Indexed Memory Retrieval
- Gravity-Weighted Significance Scoring
- Rolling Topic Vector Context Engine
- Governed Proactive Initiative (ThalamoFrontalLoop)
- Relational Integrity Coefficient (RIC)
- Self-Reflection Layer (SRL)
- Universal Values Registry Generation (UVRG)
- Environmental Training from Behavioral Evidence
- Six-Layer Cognitive Portrait System
- Truth-Separated Append-Only Evidence Architecture

Patent disclosures are prepared. Detailed specifications are available under NDA.

---

## Contact

**GitHub:** [ai-nhancement](https://github.com/ai-nhancement)
**Project Documentation:** [AiMe-public](https://github.com/ai-nhancement/AiMe-public)
**ButterClaw Fork:** [ButterClaw](https://github.com/ai-nhancement/ButterClaw)

---

> *"I don't want one system that millions of people can use. I want millions of systems, one for every individual person."*
