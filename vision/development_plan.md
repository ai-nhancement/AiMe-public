# AiMe Future Development Plan

Purpose: Introduce higher-order cognition capabilities while preserving the existing deterministic architecture.

This plan focuses on six upgrades:
1. Persistent Cognitive Workspace (PCW) â€” refinement of existing systems
2. Narrative Arc Tracker â€” extension of existing narrative_engine
3. Internal Reflection Pass â€” Stage A/B upgrade in CerebralCortex
4. Active Curiosity Engine â€” feeds from reflection output
5. Prediction Layer
6. Attention Economy
7. Multi-Thinker Unified Narration

Work is divided into phases to keep the system stable.

---

# Phase 1 â€” Persistent Cognitive Workspace (PCW)

**Status: Largely in place. This phase is a unification and surfacing effort, not a greenfield build.**

Most of what PCW describes already exists across several systems:
- CognitivePortrait tracks identity_anchors, active_concerns, relational_graph, pattern_layer, commitments, behavioral_fingerprint
- SCAL tracks standing interests (imprints + watch rules + portrait concerns) in a live Qdrant index
- Event Graph tracks concern_arc, email, and person nodes with thematic/participatory edges
- Concern Arcs track open/close lifecycle of active threads

The gap is **unification and active surfacing** â€” these systems don't present a single coherent "right now" view that other systems can query cheaply.

Goal: Create a lightweight workspace aggregation layer that assembles a real-time active-mind snapshot from existing stores.

## What to Build

**Step 1**
Create module: `modules/workspace/workspace_state.py`

This is a read-aggregator, not a new store. On demand it compiles:
- `active_arcs` â€” from NarrativeEngine (OPENING + ACTIVE state arcs)
- `open_questions` â€” from ConcernArcs (open concerns not yet resolved)
- `hypotheses` â€” populated by the Reflection Pass (Phase 3)
- `focus_topics` â€” from SCAL pattern_tracker top signals + portrait pattern_layer
- `predicted_events` â€” populated by Prediction Layer (Phase 5)

**Step 2**
Expose workspace snapshot to PromptOptimizer via a lightweight in-memory cache (TTL: 60s). No new DB table â€” the source of truth stays in the existing stores.

**Step 3**
Inject workspace into prompt as block: `<<<ACTIVE_WORKSPACE>>>`
Position: after `<<<SELF_MODEL>>>`, before `<<<SRL_STATE>>>`.

**Principle:** PCW is a lens, not a store. It reads from what already exists and presents it coherently.

---

# Phase 2 â€” Narrative Arc Tracker

**Status: Foundation in place. This phase is an extension, not a new build.**

`modules/narrative/narrative_engine.py` already exists and tracks arc lifecycle:
OPENING â†’ ACTIVE â†’ RESOLVING â†’ DORMANT

It processes every turn in Stage A, updates arc state, and the `<<<NARRATIVE_CONTEXT>>>` block is already injected into the prompt.

The gap is **arc richness and cross-system linkage** â€” arcs currently track text but don't connect to emails, schedule events, or relational nodes.

Goal: Extend the existing NarrativeEngine with richer arc metadata and cross-system linkage.

## What to Build

**Step 1**
Extend `modules/narrative/arc_tracker.py` (add alongside existing engine).

New arc metadata:
- `linked_emails` â€” email node IDs from Event Graph that match arc topic
- `linked_events` â€” calendar events semantically related to the arc
- `linked_persons` â€” relational nodes involved in the arc
- `last_mentioned_ts` â€” for staleness detection
- `momentum` â€” rolling significance score (EMA) indicating whether arc is heating or cooling

**Step 2**
In Stage A, after `_ne.process_turn()`, run arc linkage:
- Semantic match arc topic against recent emails and calendar events
- Update `linked_persons` from relational graph mentions in the turn

**Step 3**
Feed `active_arcs` (linked, annotated) into PCW workspace.

**Step 4**
Update `<<<ACTIVE_ARCS>>>` injection block with linked metadata so the LM understands arc connections, not just arc existence.

**Principle:** Arcs should know what's connected to them. An arc about a job change should automatically surface related emails, upcoming interviews, and the people involved.

---

# Phase 3 â€” Internal Reflection Pass

**Status: Foundation in place (CerebralCortex Stage A). This phase is a targeted upgrade, not a new system.**

CerebralCortex already runs a rich Stage A after every turn:
- Intent classification (cc_intent_engine)
- Significance scoring (Tier I)
- Narrative arc detection (Tier II)
- Relational depth tracking (Tier III)
- Cognitive self-model update (Tier IV)
- Longitudinal pattern scan (Tier V.2)
- Portrait consolidation (Stage C)

The gap is that Stage A **observes but does not synthesize**. Each tier runs independently and writes to its own store. Nothing looks across tiers to draw conclusions about the turn as a whole.

Goal: Add a synthesis step at the end of Stage A that produces structured reflection output â€” a compact turn assessment used by PCW and the Curiosity Engine.

## What to Build

Add `_reflect_turn()` as a final step in `_observe_turn()`, after all existing tiers complete. Deterministic â€” no LLM call.

**Reflection outputs (written to workspace cache):**

1. **Arc progression signal**
   - Did the turn advance, stall, or resolve an active arc?
   - Source: compare arc state before/after `process_turn()`
   - Output: `arc_delta = {arc_id, prev_state, new_state, momentum_delta}`

2. **Intent clarity score**
   - Was the user's intent unambiguous? (high cc_intent confidence = clear)
   - Low confidence + high significance = candidate for a curiosity prompt
   - Output: `intent_clarity = float (0.0â€“1.0), intent_label, confidence`

3. **Hypothesis update**
   - If a hypothesis in the workspace matches the turn topic and the assistant's response was factual/corrective, bump or damp the hypothesis confidence
   - Output: `hypothesis_updates = [{id, confidence_delta}]`

4. **Emotional trajectory delta**
   - Compare affect score this turn vs rolling average
   - A meaningful negative shift = signal worth tracking
   - Output: `tone_delta = float, direction = ("rising" | "stable" | "declining")`

5. **Gap flags**
   - Fields that are high-significance but low-clarity â†’ queued for Curiosity Engine
   - Output: `gap_flags = [{topic, reason, priority}]`

**Stage B upgrade (parallel draft mode):**
When Stage B is enabled, feed the reflection output into the draft prompt so the draft is informed by the turn's arc context, hypothesis state, and gap flags â€” not just the raw conversation. This makes Stage B drafts structurally aware rather than purely reactive.

**Principle:** Reflection is a cross-tier synthesizer. It reads the turn's signals together and produces a compact, structured assessment. All consumers (PCW, Curiosity, Prediction) read from this output rather than each querying the tiers independently.

---

# Phase 4 â€” Active Curiosity Engine

**Note: Moved after Reflection (Phase 3). Curiosity requires knowing what the system doesn't understand. That knowledge comes from reflection output â€” specifically `gap_flags` and low `intent_clarity` signals.**

Goal: Generate intelligent, well-timed follow-up questions when genuine knowledge gaps are detected.

## What to Build

**Step 1**
Create module: `modules/curiosity/curiosity_engine.py`

**Step 2**
Inputs from Reflection Pass:
- `gap_flags` â€” high-significance topics with low clarity
- `intent_clarity` â€” turns where the user's meaning was ambiguous
- PCW `open_questions` â€” workspace questions not yet answered
- Arc momentum â€” stalled arcs where a question might re-engage

**Step 3**
Generate "curiosity prompts" â€” structured context injected into the LLM system prompt, not standalone questions. The LM decides whether and how to ask, preserving natural conversation flow.

Format: `[CURIOSITY_SIGNAL: topic=X, gap=Y, priority=Z]` injected into `<<<ACTIVE_WORKSPACE>>>`.

**Step 4**
SCAL-style gating (companion_filter logic):
- Minimum 2-turn gap between curiosity signals
- No curiosity signal if a high-significance topic is already unresolved
- Quiet if a proactive turn was recently fired
- Per-topic cooldown (don't keep asking about the same gap)

**Principle:** Curiosity should feel natural, not procedural. The engine surfaces the gap; the LM handles the expression.

---

# Phase 5 â€” Prediction Layer

Goal: Anticipate likely future states and pre-position context before the user needs it.

## What to Build

**Step 1**
Create module: `modules/prediction/prediction_engine.py`

**Step 2**
Inputs:
- PCW workspace arcs + momentum scores
- Scheduler events (upcoming 24â€“48h window)
- SCAL pattern_tracker (recurring behaviors + escalation levels)
- Reflection tone_delta (emotional trajectory)
- Arc progression signals (arcs nearing RESOLVING state)

**Step 3**
Generate `predicted_events` list:
- Predicted decision points (arc approaching resolution)
- Predicted emotional triggers (tone trajectory + known stressors)
- Predicted information needs (event approaching + linked arc incomplete)

**Step 4**
Feed predictions to:
- PCW `predicted_events` (surfaces in `<<<ACTIVE_WORKSPACE>>>`)
- Scheduler ProactiveLoop (Phase 2 proactive triggers â€” imminent event, priority email)
- PromptOptimizer (pre-load context before the user asks)

**Principle:** Prediction is not speculation â€” it is pattern-informed pre-positioning. The LM is not told what will happen; it is given the right context so it's ready when it does.

---

# Phase 6 â€” Attention Economy

Goal: Dynamically prioritize cognitive resources â€” what gets injected, at what depth, in what order.

Currently the prompt injection order is static. Everything that passes its gate gets injected at full weight. This phase makes injection dynamic.

## What to Build

**Step 1**
Create module: `modules/attention/attention_engine.py`

**Step 2**
Score all injectable signals each turn:

| Signal Source | Scoring Input |
|---|---|
| SCAL | companion_filter consent grade + pattern escalation level |
| Memory significance | ledger significance + gravity score |
| Workspace arcs | arc momentum + recency |
| Reflection output | gap_flags priority + tone_delta magnitude |
| Prediction | predicted_event proximity |

**Step 3**
Output: ranked priority weights per injectable block. PromptOptimizer reads these weights and:
- Reorders blocks dynamically (highest attention first)
- Truncates low-weight blocks when context budget is tight
- Expands high-weight blocks to full depth

**Step 4**
Feedback loop: if a curiosity signal or prediction was surfaced and the user responded to it directly, that is a positive attention signal â†’ re-weight similar signals upward in future turns.

**Principle:** The right information at the right moment â€” not everything, not always. Attention makes the system feel less like a database and more like a mind.

---

# Integration Order

1. PCW workspace aggregation layer
2. Narrative arc enrichment (linked metadata)
3. Reflection Pass (Stage A synthesis + Stage B upgrade)
4. Curiosity Engine (feeds from reflection gap_flags)
5. Prediction Layer (feeds from workspace + arcs + reflection)
6. Attention Economy
7. Multi-Thinker Unified Narration (specialist cognition + fused single-voice response)

Each phase validated independently before moving to the next.

---

# Architectural Principles (Do Not Break)

- Single user-facing narration path per turn â€” internal thinker calls may be multiple, but only one narrator speaks
- Append-only evidence ledger â€” never rewrite, only append
- Deterministic cognition layers â€” no LLM calls in Stage A/B reflection
- LanguageCortex is sole narrator â€” no plugin narrates directly
- PCW is a lens, not a store â€” source of truth stays in existing systems
- Curiosity signals are context, not commands â€” LM decides expression

---

# Expected Outcome

After these phases AiMe gains:

- A coherent real-time active-mind state (PCW)
- Long-term narrative understanding with cross-system linkage (Arcs)
- Structured turn synthesis that informs every downstream system (Reflection)
- Proactive curiosity grounded in genuine knowledge gaps (Curiosity)
- Anticipatory context positioning before the user needs it (Prediction)
- Dynamic cognitive resource allocation (Attention)

This completes the move from assistant â†’ relational cognitive companion.

---

# Phase 7 â€” Multi-Thinker Unified Narration

**Status: Newly identified near-term architecture opportunity. This phase should start small and use existing model/provider infrastructure before any bespoke model training is considered.**

Goal: allow multiple internal specialist thinkers to inspect the same turn, contribute structured judgment, and feed one narrator model that speaks in AiMe's unified voice.

## What to Build

**Step 1**
Extend PrefrontalCortex to request one or more thinker roles in turn metadata.

Suggested first fields:
- _thinkers_requested
- _thinker_mode
- _thinker_priority
- _thinker_weights
- _intent_vector

**Step 2**
Add a thinker execution stage in CognitiveBridge.

This stage should:
- call selected thinker profiles
- collect structured outputs only
- store them in a bounded synthesis pool
- preserve assigned thinker weights alongside results

**Step 3**
Create a deterministic fusion layer.

The first version should:
- deduplicate overlaps
- preserve high-priority warnings
- rank by relevance/significance
- weight findings by thinker importance for the current turn
- surface conflicts explicitly

**Step 4**
Pass only the fused pool to the narrator model.

**Principle:** many internal judgments, one governed response.

**Control rule:** intent drives thinker weighting, weighting drives fusion priority, and fusion drives narration.

## Practical Constraint

The canonical thinker council should be **The Phronesis**:
- Logos - Analytical
- Praxis - Practical
- Philia - Relational
- Poiesis - Creative
- Ethos - morals, values, and beliefs

Coding and technical analysis belong under `Logos` / `Analytical` in the future architecture rather than remaining a privileged lane identity.

Not every turn needs all five active at once. The Phronesis is canonical, but activation should remain selective and intent-weighted.

These do not need to be newly trained frontier models at first. The right near-term move is to use best-fit current models or smaller local models under strict role contracts.

**Principle:** the thinkers can be small because they do not have to be the self.

## Why This Belongs Here

Phases 1-6 increase AiMe's ability to accumulate, structure, and prioritize cognition.
Phase 7 changes how that cognition is internally processed before narration.

This is the step from:
- one model with rich context

to:
- one system with plural specialist cognition and unified narration


