# The Bond
## From Retrieval to Relationship-State in Cognitive Systems

> "I stopped building retrieval and started building relationship state.
> So context isn't searched — it's entered."

---

## 1. Executive Summary

The Bond is the foundational relational primitive of AiMe.

It represents a shift from retrieval-centric cognition to relationship-state cognition.
Instead of searching memory by topic, the system activates memory through relational state.

Memory remains a database.

But the index is no longer topic.

The index is **bond**.

---

## 2. The Core Realization

Most AI systems are built around queries.

Queries are atomic:
- They begin.
- They resolve.
- They reset.

Human relationships are not atomic.

They are cumulative:
- Built over time.
- Weighted by significance.
- Shaped by emotion.
- Reinforced through shared narrative.
- Adjusted through trust and correction.

The goal was not to build a better interface.
The goal was to build the relational layer between a human and a cognitive system.

That layer is the **Bond**.

---

## 3. The Reframe

### Traditional Model

```
store → retrieve → respond
```

### Bond-Forming Model

```
enter relationship state → memory surfaces as a consequence → respond
```

Retrieval becomes secondary.

Primary operation:
> Enter the bond state.

---

## 4. Memory as a Relationship-Indexed Field

Humans do not retrieve memories from labeled slots.

We do not think:
> "Pull memory from slot A."

Instead:

- When we are around a person, shared memories surface.
- When we return to a place, associated experiences reappear.
- When we re-enter a project, unresolved threads become salient.

Memory is activated by relationship context.

Therefore:

> Memory is a relationship-indexed field.
> The index is not topic — it is bond.

---

## 5. Formal Definition: The Bond

**Bond**

A persistent, evolving, bidirectional relational state between a human and a cognitive system, strengthened through significance, continuity, reinforcement, and shared narrative.

A bond contains:

- **Identity Cues**
  Who or what the bond represents.

- **Salience Weighting**
  What matters most within the bond.

- **Affective Register**
  Emotional tone tracked longitudinally.

- **Open Threads**
  Active narrative arcs and commitments.

- **Shared History**
  Significance-weighted episodes.

- **Trust Calibration**
  Reinforcement and correction patterns.

- **Temporal Awareness**
  Decay, change, and validity windows.

A bond is not a memory record.

It is a structured relational state.

---

## 6. Architectural Implications

The shift to bond-centric cognition changes the system primitive.

### Before

- Stateless session memory
- Uniform fact weighting
- Topic-indexed retrieval
- Single-turn lookup
- Retrieval-first cognition

### After

- Persistent relationship portraits
- Significance-weighted gravity
- Narrative arc lifecycle tracking
- Commitment and deadline awareness
- Relational salience graph
- Cognitive self-model
- Temporal truth decay
- Trajectory-aware context blending
- Relationship-indexed activation

The governing question becomes:

> "Within this relationship, what matters now?"

---

## 7. Human Analogy (Relatable Explanation)

When you are around someone you've known for years:

You do not manually retrieve who they are.

You do not pull structured data.

The relationship itself activates shared history.

This system operates the same way.

Context is not searched.

It is entered.

---

## 8. Relationship State Activation Model

### Step 1 — Detect Active Bond(s)
Based on:
- Mentioned people
- Narrative arcs
- Environmental cues
- Ongoing commitments
- Session trajectory

### Step 2 — Enter Bond State
Activate:
- High-salience memories
- Active arcs
- Recent significance-weighted events
- Trust calibration signals

### Step 3 — Surface Memory
Memories surface as a consequence of bond activation.

Retrieval is guided by relationship gravity, not just semantic similarity.

---

## 9. The Long Memory Plan (Technical Implementation)

The Bond abstraction was made possible through the Long Memory Plan implementation, which introduced:

- Embedding unification
- Significance scoring
- Narrative arc engine
- Commitment tracking
- Relational salience graph
- Cognitive self-model
- Temporal truth system
- Predictive preloading
- Behavioral health monitoring
- Durability and archival infrastructure
- Session-stateful trajectory context engine

---

### Full Implementation Specification

## Cognitive Architecture Evolution — The Long Memory Plan

> *"Close the gap between a system that records and one that understands."*
>
> The Long Memory Plan was a structured program to evolve AiMe from a stateless session companion into a continuously-learning cognitive entity. It was conceived, designed, and fully implemented by a solo developer working with a single AI system — a real-world proof of The Bond principle that humans and AI together can accomplish what neither could alone. What would have taken a committee of engineers 12+ weeks was completed in a fraction of that time.

### The Core Problem It Solved

Before the Long Memory Plan, AiMe had three problems:

1. **No longitudinal coherence** — each conversation started cold regardless of history
2. **No importance hierarchy** — a trivial question and a profound personal revelation were stored identically
3. **No self-model** — AiMe had no understanding of its own knowledge gaps, interaction patterns, or relationship with the user

The plan addressed all three through seven tiers of capability, plus five foundational fixes.

---

### Foundation Work (Pre-Tier)

**F1 — Embedding Unification** (`tools/migrate_knowledge_store.py`, commit `74962bbb`)

The KnowledgeStore was originally indexed with MiniLM 384-dimensional embeddings while all other retrieval paths used BGE-base 768-dimensional embeddings. This meant knowledge objects existed in a different vector space from conversation memory — semantic search would find related conversations but miss related knowledge, and vice versa. F1 replaced all KnowledgeStore embeddings with BGE-base via a migration script that loads `BAAI/bge-base-en-v1.5` directly (bypassing the EmbeddingManager which would have loaded the SetFit checkpoint) and upserts all 41 existing objects into the unified `knowledge_objects_v2` Qdrant collection.

**F2 — Portrait Eviction** (commit `a2850910`)

The CognitivePortrait `identity_anchors` layer had no eviction policy — facts accumulated without bound. F2 added importance-weighted eviction: when the anchor count exceeds the cap, the lowest-importance anchors are evicted first, preserving high-signal facts regardless of recency.

**F3 — Temporal Truth** (`modules/temporal/temporal_index.py`)

Facts don't stay true forever. A job title from two years ago may be wrong today. F3 introduced fact validity windows and decay thresholds: each fact type carries a half-life (`long: 540d`, `medium: 365d`, `short: 180d`, `project: 60d`, `momentary: 1d`, `permanent: null`). When a fact's confidence decays below `stale_threshold: 0.25`, the prompt qualifies it with "(may be outdated)". Below `warn_threshold: 0.5`, it reads "(possibly outdated)". Facts are never deleted — AiMe knows to verify before asserting.

**F4 — Bootstrap Deadlock Fix** (commit `a2850910`)

A cold-start condition could block the first learning cycles: an empty anchor store would classify every candidate as CONTRADICTED (nothing to agree with), preventing the accumulation of reinforcement data needed to build anchors. F4 added a bootstrap protection mode that relaxes the minimum anchor similarity threshold when internal anchor count is below 50, allowing the first cycles to populate the anchor store.

**F5 — Guardian Attestation Coupling** (commit `a2850910`)

Truth admission (via `truth_admitter_plugin.py`) is now gated on the Guardian reaching NORMAL state (~3 minutes after boot). This prevents candidates from being admitted during the system's unstable early state when governance hasn't yet confirmed system integrity.

---

### Tier I — Significance Layer (`modules/significance/scorer.py`, commit `e5fb6c03`)

**The Problem:** Every turn was treated identically — a casual greeting and a conversation about a life decision both entered the ledger with the same weight. This made the gravity system ineffective at distinguishing what mattered.

**The Solution:** A per-turn significance scorer that computes a composite score from four signal dimensions:

```
significance = 0.30 × affect + 0.30 × novelty + 0.25 × resolution + 0.15 × echo
```

- **Affect**: Emotional signal weight. Detected via vocabulary pattern matching against positive and negative affect word sets. High-affect turns score higher.
- **Novelty**: New information density. Measured by comparing turn content against the last 10 stored records' vocabulary. High overlap = low novelty.
- **Resolution**: Decision/completion signal. Regex patterns detect "I decided", "we agreed", "it's done" etc. Resolution events are high-value.
- **Echo**: Reinforcement of prior concerns or arcs. Turns that reference open concerns or active narrative arcs score higher — they are continuations of ongoing threads.

Significance is written to `records.significance` in Stage A. The gravity system uses it: `sqrt(1 + max(significance, citations))`. High-significance facts surface more readily as latent episodes in future turns. Portrait anchors created from high-significance turns receive an `anchor_importance_boost` (default 2×) at insertion.

---

### Tier II — Narrative Engine + Commitment Tracking (`modules/narrative/narrative_engine.py`, commit `24996611`)

**The Problem:** AiMe could not track recurring themes across conversations. A project discussed across five separate sessions was invisible as a coherent arc. Commitments made in conversation were never tracked.

**The Solution, Part A — Narrative Arc Engine:**

An arc lifecycle manager that detects recurring topics and tracks their progression:
```
OPENING → ACTIVE → RESOLVING → DORMANT
```

Arc detection uses vocabulary-overlap coefficient (intersection / min-set-size, threshold 0.15) rather than embedding similarity — this avoids embedding calls in the Stage A hot path while still being robust to rephrasing. A new arc opens when a turn matches no existing arc and significance ≥ 0.20. It escalates to ACTIVE after 2+ matching turns. Resolution regex (`resolved`, `done`, `figured out`, etc.) marks RESOLVING. Arcs unseen for 30 days become DORMANT.

Active arcs inject a `<<<NARRATIVE_CONTEXT v=1>>>` block into the system prompt: `"[14d, ACTIVE] project discussion"`. This gives the LM visibility into the ongoing story of the relationship without requiring explicit recall.

**The Solution, Part B — Commitment Tracking:**

Commitment entries in the portrait now include deadline extraction via `_DEADLINE_PATTERNS` (regex for "by Friday", "before the 15th", "next week", etc.). Commitments are annotated as `due in 3 days`, `overdue by 2 days` when injected into `<<<PORTRAIT>>>`. `get_pending_commitments()` returns items due within 7 days.

---

### Tier III — Relational Depth (`modules/portrait/cognitive_portrait.py`, commit `ca1a0e09`)

**The Problem:** The `relational_graph` stored simple {person: value} pairs. AiMe knew your sister's name but had no sense of how often she came up, what emotional register surrounded her, or what topics connected to her.

**The Solution:** Full RelationalNode format for every person in the graph:

```python
{
  "name": "Sarah",
  "mention_count": 23,
  "emotional_register": 0.72,   # EMA, α=0.3 per mention
  "context_vocabulary": ["work", "college", "Boston"],
  "arc_ids": ["arc_042"],
  "salience": 0.91               # log(count+1) × (1+affect) × recency_90d
}
```

Stage A in CerebralCortex detects person names in each turn and calls `portrait.mention_person()`, which updates the mention count, applies a 0.3-weighted EMA to the emotional register (positive/negative vocabulary of the surrounding sentence), and adds context words. The `[PEOPLE]` block in `<<<PORTRAIT>>>` is sorted by salience — the most relationship-relevant people appear first. After 5+ mentions, the top 3 context words appear alongside the name.

---

### Tier IV — Cognitive Self-Model (`modules/portrait/self_model.py`, commit `162e2e1b`)

**The Problem:** AiMe had no introspective model of itself — no awareness of what it didn't know, where it made characteristic errors, or what interaction patterns worked well.

**The Solution:** A `CognitiveSelfModel` stored in `identity_kv` (owner `self_model:{session_id}`) with three living layers:

- **Knowledge Frontiers:** Topics where AiMe has given zero-result responses 3+ times. AiMe knows what it doesn't know.
- **Characteristic Errors:** Correction patterns detected via `_CORRECTION_RE` ("that's wrong", "actually", "no,"). Per-topic error counts.
- **Effective Patterns:** Positive reinforcement via `_POSITIVE_RE` ("exactly", "that's perfect", "yes!"). What works gets remembered.

Additional state: `trust_level` EMA (updated by correction/positive signals), `interaction_depth` EMA (complexity of conversations over time).

Injected as `<<<SELF_MODEL v=1>>>` in the system prompt:
```
[FRONTIERS] quantum_computing, tax_law
[WATCH] tend to over-explain code examples
[EFFECTIVE] concise bullet points, direct answers
```

---

### Tier V — Predictive Preloading (`modules/portrait/predictive_context.py`, gated `enabled: false`)

**What it does:** Scans aged concerns (open > 14 days), narrative momentum signals, and longitudinal day-of-week patterns to proactively inject anticipatory context before reading user input. Creates `[ATTENTION]` signal lines for dormant-but-relevant concerns. All signals are read-only. The longitudinal scanner builds weekly topic patterns by scanning the ledger and storing them in the self-model.

**Status:** Fully implemented, disabled by default. Enable via `predictive_preloading.enabled: true` in `lm_tuning.json`.

---

### Tier VI — Behavioral Health Monitor (`modules/portrait/behavioral_health.py`, gated via `lm_tuning.json`)

**What it does:** Computes a composite behavioral health score from five indicators:

- **Recall Precision:** Rapid re-query rate (consecutive recall turns within 120s indicate poor precision)
- **Concern Resolution Rate:** % of concerns that close vs stagnate (>30 days open = stagnant)
- **Portrait Freshness:** High-significance turn density in the last 30 days (target: 20 turns ≥ 0.2 sig)
- **Narrative Ghost Rate:** Arc density of "ghost arcs" (unseen >7 days AND older than 21 days)
- **Topic Diversity:** Variety of intent classes across recent sessions

`warn_threshold: 0.45`, `healthy_threshold: 0.60`. These are indicators only — nothing blocks learning.

---

### Tier VII — Durability (`lm_tuning.json → durability`, tooling)

**What it does:** Long-term operation stability:

- **Ledger Archival:** `tools/archive_old_records.py` soft-archives records older than `archive_months: 6`. Hot retrieval excludes archived records — keeps Qdrant and Meilisearch lean without data loss.
- **Weekly SQLite Backups:** `backup_keep: 4` retains 4 rolling weekly backups of `aime_ledger.db`.
- **Embed Model Tagging:** All Qdrant payloads carry `embed_model: "bge-base-en-v1.5"` for forward-compatibility if the embedding model is ever changed.

---

### Phase 2 — Context Engine (`modules/context/`, commit `c3793344`)

**The Problem:** Per-turn KnowledgeStore lookup was stateless — each turn independently retrieved the 5 most semantically similar facts. This worked for isolated queries but failed to track conversation *trajectory*. A session that started discussing "work stress" and evolved into "career change" would surface facts about stress on turn 10 even though the conversation had moved on. There was also no way to surface relevant *past conversations* alongside knowledge facts.

**The Solution:** A session-stateful retrieval engine built on a rolling topic vector.

**Architecture:**
```
[Turn N arrives]
     ↓
ConversationStateManager.get_or_create(session_id)
     ↓ → topic_vector from turns 1..N-1
ContextEngine.get_context_pack(session_id, current_emb, user_text, meta)
     ├── query_vector = normalize(0.6 × topic_vector + 0.4 × current_emb)
     ├── KnowledgeStore.retrieve(query_vector, top_k=8)
     ├── Hippocampus.recall(query_embedding=query_vector, top_k=3)  → past turns
     └── deduplicate vs injected_ids → rank → top 6
     ↓
<<<CONTEXT_MEMORY v=2 n=N>>> in system prompt
     ↓ (daemon thread, post-response, non-blocking)
ConversationStateManager.update(session_id, turn_embedding)
     topic_vector[N] = normalize(0.7 × topic[N-1] + 0.3 × turn_emb[N])
```

**Rolling Vector Math (α=0.7):**

| Turn | Contribution to current query vector |
|------|--------------------------------------|
| N (current, via blend) | 40% |
| N-1 (last turn) | 42% |
| N-2 | 29% |
| N-3 | 20% |
| N-4+ | diminishing, combined ~24% |

The result is a query vector that represents the conversation's direction — not just the current message, but where the conversation has been heading.

**Deduplication:** `injected_ids` is a rolling deque of 60 entries (per session). Items already injected in the last ~60 turns are excluded from new retrievals. This prevents the same fact from appearing in every prompt across a long session.

**Type Priority Ranking:**
```
FACT (0) > NOTE (1) > past_turn (2) > TOOL_RESULT (3)
```
Within each type, items are ranked by similarity score descending.

**Format in Prompt:**
```
<<<CONTEXT_MEMORY v=2 n=4>>>
- [FACT] User's sister is Sarah (family)
- [NOTE] Prefers concise bullet points over long explanations
- [PAST] User asked about career change anxiety → AiMe responded with reframing...
- [TOOL_RESULT] Weather: 72°F, clear (weather, tool_result)
<<<END_CONTEXT_MEMORY>>>
```

**Fallback:** `_get_engine_context()` in `language_model_plugin.py` wraps the engine call and falls back to Phase 1 `_lookup_knowledge_context()` on any exception. Zero regression guaranteed.

**State update timing:** The topic vector update fires **after** the response is sent (daemon thread), not before. Context for turn N uses the trajectory from turns 1..N-1 blended with the current message. This is the correct behavior — the model is always one turn ahead of where the vector thinks the conversation is, which creates productive tension toward the current topic while maintaining contextual memory.

---

### What The Long Memory Plan Achieved

In architectural terms:

| Before | After |
|--------|-------|
| Stateless session memory | Persistent, evolving per-user portrait |
| Uniform fact weighting | Significance-weighted storage and retrieval |
| Single-turn knowledge lookup | Session-trajectory-aware context engine |
| No narrative awareness | Active arc tracking across sessions |
| No relational depth | Salience-ranked relationship graph |
| No self-knowledge | Frontier/error/pattern self-model |
| Flat embedding space (mixed dims) | Unified 768d BGE vector space across all stores |
| Time-invariant facts | Temporal decay with soft qualification |

In human terms: **AiMe went from a system that remembered things to a system that understands context.**

---

## 10. Result

The system moved from:

> Remembering information

To:

> Maintaining relational continuity.

It is now a:

# Bond-Forming Cognitive System

---

## 11. One-Sentence Definition

> A bond-forming cognitive system maintains a persistent relationship-state such that memory is activated through continuity rather than searched through retrieval.

---

## 12. Future Extensions (Placeholder)

- Formal Bond schema definition
- Bond strength metrics
- Rupture and repair modeling
- Multi-bond interaction modeling
- Environmental bond modeling
- Cross-bond inference rules
- Bond health metrics
- Compounding relational velocity measurement
