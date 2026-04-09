# Gravity-Weighted Significance: Automatic Memory Salience in Long-Context Conversational AI

**Authors:** John Canady Jr.
**Affiliation:** AI nHancement, Independent Research
**Date:** April 2026
**Public Copy:** https://www.ai-nhancement.com/paper/gravity-weighted-significance

---

## Abstract

We present a two-stage scoring system for episodic memory records in persistent conversational AI systems. The first stage, Significance Scoring, computes a composite importance score for each conversational turn using four independently measurable dimensions: affective intensity, contextual novelty, resolution signal, and concern echo. The second stage, Gravity Scoring, combines significance with backward citation density, relational state connectivity, and exponential temporal decay to produce a dynamic salience value that determines whether a memory record should be automatically surfaced into the language model's prompt context on subsequent turns. Records exceeding a gravity threshold are injected as latent episodes -- unsolicited but contextually appropriate memory fragments that enter the model's awareness without explicit user request. The system addresses a fundamental problem in long-running conversational AI: not whether to store a memory, but which memories to surface at any given moment, given who the user is and what they are currently thinking about. The scoring system is implemented and operational in a live single-user cognitive system, where it annotates every conversational turn with permanent significance scores and dynamically computes gravity at each turn to select latent episodes for prompt injection. We present the method as an operational architecture for long-running conversational systems and as a testable alternative to recency-only or query-only surfacing strategies.

---

## 1. Introduction

A persistent conversational AI system that records every interaction faces an inevitable challenge: the memory store grows without bound, but the language model's context window does not. After thousands of turns, the system possesses far more relevant information than it can present to the model on any single turn. The question is not whether to store memories -- append-only storage is cheap and preserves provenance. The question is which memories to surface, and when.

Existing approaches to this problem are unsatisfying. Recency-based selection (most recent N records) produces temporal bias -- the system forgets important old events in favor of trivial recent ones. Semantic similarity retrieval (top-N closest to the current query) produces topical bias -- the system retrieves what sounds similar, not what matters most [Lewis et al., 2020]. Neither approach answers the question that a human conversational partner answers intuitively: given everything I know about this person, what is most important for this moment?

This paper presents a two-stage scoring system that answers that question through formal computation. Stage 1 (Significance) evaluates the intrinsic importance of each turn at the time it occurs, producing a permanent annotation. Stage 2 (Gravity) evaluates the dynamic relevance of each annotated record at retrieval time, combining intrinsic importance with extrinsic validation, relational connectivity, and temporal decay. Records with sufficient gravity surface automatically as latent episodes -- context injected into the language model's prompt without the user requesting it.

The contribution is threefold:

1. A four-component significance formula that captures affective intensity, contextual novelty, resolution events, and concern resonance as independently measurable dimensions of conversational importance.
2. A compound gravity formula that integrates significance with backward citation, relational state connectivity, and temporal decay to produce dynamic salience scores.
3. A latent episode mechanism that uses gravity scores to automatically surface memories into prompt context, producing unsolicited but contextually appropriate recall.

---

## 2. The Surfacing Problem

### 2.1 Why Storage Is Not the Hard Problem

Modern storage is effectively unlimited for text-based conversational records. A year of daily interaction produces perhaps 50,000 turn records -- trivially storable in any database. The hard problem is not capacity but selection: which of those 50,000 records should the language model see on this particular turn?

The context window of a language model imposes a hard budget. A system prompt that includes relational state (the Bond portrait), behavioral constraints (contracts and governance), active context (email, calendar, presence), and the current dialogue already consumes a substantial fraction of available tokens. The budget remaining for historical memory injection is typically 2,000-4,000 tokens -- enough for perhaps 3-5 selected records.

Selecting the right 3-5 records from 50,000 candidates is the surfacing problem.

### 2.2 Failure Modes of Naive Selection

**Recency bias.** Selecting the most recent records ignores the possibility that an important conversation from three weeks ago is more relevant than a trivial exchange from yesterday. A user who mentioned their mother's illness three weeks ago and now says "I'm feeling overwhelmed" needs the system to recall the illness, not yesterday's weather discussion.

**Topical bias.** Selecting by semantic similarity to the current utterance misses contextually important records that are topically dissimilar. The connection between "feeling overwhelmed" and "mother's illness" is relational, not lexical -- it exists because the system knows this user has an active concern about their mother's health.

**Popularity bias.** Selecting frequently-accessed records creates a rich-get-richer effect where the same memories surface repeatedly while equally important but less-accessed records fade.

### 2.3 Design Requirements

An effective memory salience scorer must satisfy six requirements:

1. **Affective sensitivity** -- emotionally intense moments must score higher than neutral informational turns
2. **Novelty awareness** -- information the system did not previously know should score higher than redundant repetitions
3. **Resolution detection** -- turns where decisions were made or problems solved represent inflection points
4. **Relational continuity** -- turns connected to the user's current concerns are more relevant than unrelated past turns
5. **Temporal decay** -- old records are generally less relevant than recent ones, all else being equal
6. **Citation propagation** -- records that later turns reference or build upon have demonstrated lasting importance

Requirements 1-4 are addressed by Stage 1 (Significance). Requirements 4-6 are addressed by Stage 2 (Gravity). Requirement 4 appears in both stages because concern echo contributes to intrinsic significance while relational connectivity contributes to dynamic gravity.

---

## 3. Stage 1: Significance Scoring

### 3.1 Formula

$$sig = 0.30 \times affect + 0.30 \times novelty + 0.25 \times resolution + 0.15 \times echo$$

Significance $sig \in [0.0, 1.0]$ is computed once per turn and written to the evidence ledger as a permanent, immutable annotation. It is never recomputed, even as the user's relational state evolves. This ensures historical stability -- a turn's importance at the time it was recorded is preserved as a fixed property of the record.

### 3.2 Affect (weight = 0.30)

**Definition.** The maximum emotional intensity signal detected in the turn's text content.

**Method.** A lexical lookup against a curated affective intensity dictionary mapping words and phrases to normalized intensity scores $\in [0.0, 1.0]$. Only the maximum detected intensity is used, not the mean -- a single high-intensity expression is sufficient to classify a turn as emotionally significant.

**Critical design choice: magnitude, not valence.** Affect is scored on intensity regardless of polarity. Grief and joy at equivalent intensity receive equivalent significance boosts. This is because both represent psychologically significant events in the relationship -- moments where the user's emotional state was activated and the system's response mattered. A memory system that privileges positive emotions over negative ones (or vice versa) would produce a distorted portrait of the relationship's history.

**Examples:**
- "I got the job!" -> affect $\approx$ 0.85 (high-intensity positive)
- "My mother passed away." -> affect $\approx$ 0.90 (high-intensity negative)
- "What time is it?" -> affect $\approx$ 0.05 (neutral, informational)

### 3.3 Novelty (weight = 0.30)

**Definition.** The degree to which the turn introduces vocabulary not already represented in the user's identity portrait.

**Formula:**

$$novelty = 1 - J(V_{turn}, V_{portrait})$$

where $J(A, B) = |A \cap B| / |A \cup B|$ is the Jaccard similarity between the turn's non-stopword vocabulary and the portrait's accumulated vocabulary (identity anchor values, concern topics, relational graph node names).

**Rationale.** A turn that introduces new names, new topics, or new concepts expands the system's model of the user and is correspondingly more significant. Turns that repeat known vocabulary provide less new information.

**Cold start behavior.** When $V_{portrait}$ is empty (new relationship, no established facts), $novelty = 1.0$ for all turns. This ensures early turns receive high significance during the bootstrapping period, which is desirable -- the first conversations in a new relationship are among the most important for establishing the relational foundation.

### 3.4 Resolution (weight = 0.25)

**Definition.** Binary detection of decision, closure, or problem-solving events.

**Method.** Regular expression matching against curated resolution phrases: "decided to," "settled on," "figured out," "resolved," "going with," "made up my mind," "finally chose," "committed to," "moving forward with," "done thinking about."

$resolution = 1.0$ if any pattern matches, $0.0$ otherwise.

**Rationale.** Decision events are inflection points in a user's life narrative. A turn where the user announces "I decided to take the job in Austin" is categorically more important than a turn where they continue deliberating about the same decision. These inflection points serve as natural chapter boundaries and should have high persistence in memory. The binary nature of the score (1 or 0) reflects the categorical distinction: a decision was either made or it was not.

### 3.5 Echo (weight = 0.15)

**Definition.** Whether the current turn's vocabulary intersects with any currently open concern topic in the user's active concern stack.

**Method.** For each active concern, compare the turn's token set against the concern's stored topic vocabulary. $echo = 1.0$ if any overlap exists, $0.0$ otherwise.

**Rationale.** Echo detects whether the current turn resonates with an open thread -- a topic the user has flagged (explicitly or implicitly) as something they are actively thinking about. A turn that echoes an open concern is more contextually relevant than a topically similar turn that does not connect to any active concern.

**Example.** If the user has an open concern "career change," then a turn asking "what are some good ways to update a resume?" receives $echo = 1.0$ even if its affect and resolution scores are both zero. The turn is significant because it connects to an active thread in the user's life, not because it is emotionally intense or decisionally pivotal.

### 3.6 Computation Timing

Significance is computed asynchronously after each turn by an observation cycle that fires post-response on a daemon thread. The scorer receives the completed turn record (input text + output text + session context), loads the current portrait, computes all four components, and writes the resulting significance score to the evidence ledger. This computation never blocks the response path.

The asynchronous timing means significance scores may be written slightly after the turn completes -- but since significance is used only for future retrieval (not for the current turn's response), this latency is invisible to the user.

---

## 4. Stage 2: Gravity Scoring

### 4.1 Formula

$$gravity = W_{intent} \times \sqrt{1 + \max(sig, citations)} \times M_{conn} \times D_{recency}$$

where:
- $W_{intent}$ is the base intent weight of the record
- $sig$ is the Stage 1 significance score
- $citations$ is the backward citation count
- $M_{conn}$ is the portrait connectivity multiplier
- $D_{recency}$ is the exponential temporal decay factor

Unlike significance, gravity is **dynamic** -- it is recomputed at the start of each turn by evaluating each candidate record against the current portrait state and the current timestamp.

### 4.2 Base Intent Weight ($W_{intent}$)

A fixed weight assigned by the intent classification of the turn that produced the record:

| Intent Class | Weight | Rationale |
|-------------|--------|-----------|
| plan | 0.80 | Planning turns establish future commitments |
| identity | 0.75 | Identity-establishing turns define the relational foundation |
| recall | 0.70 | Explicit recall events are important by user intent |
| truth_ledger | 0.70 | Verified facts in the evidence ledger |
| distress | 0.60 | Emotional distress moments are high-gravity by nature |
| general | 0.40 | General conversation -- moderate base gravity |
| unknown | 0.10 | Unclassified -- low prior |

The base weight establishes a prior probability that a given record type is worth surfacing. Planning and identity turns are intrinsically more important than general conversation, independent of their specific content.

### 4.3 Compound Amplification ($\sqrt{1 + \max(sig, citations)}$)

This term rewards records that are important either intrinsically (high significance score) or extrinsically (cited by later turns). The $\max$ operator ensures that either form of importance is sufficient -- a record with zero significance but five backward citations receives substantial amplification, as does a record with high significance but no citations.

The square root produces sublinear growth, preventing any single record from dominating gravity scores. This is essential for diversity: without the dampening effect of $\sqrt{}$, a single high-significance, high-citation record could crowd out all other candidates.

**Range examples:**
- $sig = 0.0$, $citations = 0$ -> amplification = 1.0 (no boost)
- $sig = 0.83$, $citations = 0$ -> amplification $\approx$ 1.35
- $sig = 0.0$, $citations = 5$ -> amplification $\approx$ 2.45
- $sig = 0.90$, $citations = 10$ -> amplification $\approx$ 3.32

### 4.4 Portrait Connectivity Multiplier ($M_{conn}$)

A record is "Bond-connected" if its content overlaps with any dimension of the currently active relational state object:
- Any identity anchor key or value
- Any active concern topic
- Any relational graph node name

$$M_{conn} = \begin{cases} 1.5 & \text{if Bond-connected} \\ 1.0 & \text{otherwise} \end{cases}$$

This 50% gravity boost for Bond-connected records is the mechanism by which the user's current relational state influences memory surfacing. A record about the user's sister receives elevated gravity whenever the sister is mentioned -- not because the current message is topically about the sister, but because "sister" is a node in the relational graph that connects to the record through the Bond state [Canady, 2026a].

### 4.5 Exponential Temporal Decay ($D_{recency}$)

$$D_{recency} = e^{-age_s \cdot \ln(2) / T_{half}}$$

where $age_s$ is the record's age in seconds and $T_{half} = 30 \times 86400$ seconds (30-day half-life).

| Age | Decay Factor |
|-----|-------------|
| 0 days | 1.0 |
| 30 days | 0.5 |
| 90 days | 0.125 |
| 365 days | 0.00025 |

The 30-day half-life represents a natural conversational relevance window. Memories from the past month are substantially more likely to be contextually relevant than memories from six months ago. However, the compound interaction with significance and citation count means that highly important or highly-cited records retain gravitational pull even as they age -- a record with significance 0.90 and 10 citations retains meaningful gravity for months, while a trivial record with significance 0.10 and no citations fades within weeks.

---

## 5. Latent Episode Selection and Injection

### 5.1 Selection Algorithm

After all candidate records are scored, selection proceeds in three steps:

1. **Threshold filter.** Only records with $gravity \geq \theta$ (default $\theta = 0.35$) are candidates.
2. **Connectivity gate.** At least one of significance or connectivity must be non-zero. This prevents very old, uncited, Bond-disconnected records from surfacing purely on the strength of a high base intent weight.
3. **Top-K selection.** The top $K$ records by gravity score (default $K = 3$) are selected as latent episodes.

### 5.2 Prompt Injection

Selected latent episodes are injected into the language model's system prompt as a structured block:

```
<<<LATENT_EPISODES v=1 n=N>>>
[intent] content_snippet [sig=0.XX]
[intent] content_snippet [sig=0.XX]
...
<<<END_LATENT_EPISODES>>>
```

This block is positioned after the relational state portrait and before the user's current message. The language model receives these memories as implicit context -- it is free to reference them naturally or to let them inform its response without explicit mention. The user does not see the injection.

### 5.3 The Unsolicited Recall Property

Latent episodes are not query results. The user did not ask for them. They are memories that the system's relational and temporal state determined to be salient. This is the defining property of the mechanism: it produces recall that is contextually appropriate but not explicitly requested -- the computational equivalent of a memory that "comes to mind" during conversation.

This property is what distinguishes gravity-based surfacing from retrieval-augmented generation. RAG answers: "what stored records match this query?" Gravity answers: "given who this person is, what they care about, and what has been important in our conversation history, which memories have enough gravitational pull to surface right now?"

---

## 6. Comparison to Prior Art

### 6.1 TF-IDF and BM25

Classical information retrieval scores documents by term frequency and inverse document frequency. These methods operate on query-document pairs and have no concept of importance, affect, temporal decay, or relational state. They answer "what matches?" not "what matters?"

### 6.2 Embedding-Based Retrieval

Dense retrieval methods [Karpukhin et al., 2020] encode queries and documents into a shared embedding space and retrieve by cosine similarity. This improves on lexical matching but still operates on query-document similarity rather than intrinsic importance or relational relevance.

### 6.3 Memory Networks

End-to-end memory networks [Sukhbaatar et al., 2015] and subsequent architectures learn attention over memory slots within a single forward pass. These operate at the token level within the model's context window and do not address the cross-session, cross-turn memory selection problem that gravity scoring solves.

### 6.4 Summarization-Based Context Management

Some long-context systems compress prior conversation into summaries that fit within the context window [Chevalier et al., 2023]. This preserves information density but loses the episodic granularity that makes specific memories powerful. "You mentioned being worried about your mother's health" is more relationally impactful than a compressed summary that mentions the topic in passing.

### 6.5 Summary Comparison

| Property | TF-IDF / BM25 | Dense Retrieval | Memory Networks | Context Compression | Gravity Scoring |
|----------|---------------|-----------------|-----------------|---------------|-----------------|
| Importance scoring | None | None | Learned attention | Implicit | Explicit 4-component formula |
| Affect awareness | None | None | None | None | Magnitude-based (valence-independent) |
| Temporal dynamics | None | None | None | Recency in summary | 30-day exponential decay |
| Citation signal | None | None | None | None | Backward citation count |
| Relational connectivity | None | None | None | None | Bond-state multiplier |
| Unsolicited recall | No | No | No | No | Yes (latent episodes) |

---

## 7. Implementation and Evaluation

### 7.1 System Context

Gravity-Weighted Significance is implemented within AiMe, a Memory-Augmented Cognitive Intelligence (MACI) system for persistent single-user interaction. The evidence ledger grows continuously with daily use, making effective memory selection increasingly important as the store expands.

### 7.2 Key Components

| Component | File | Role |
|-----------|------|------|
| Significance Scorer | `modules/significance/scorer.py` | Four-component formula, all weights configurable |
| Gravity Scorer | `modules/portrait/gravity.py` | Compound gravity formula, latent episode selection |
| Observation Trigger | `features/cerebral_cortex.py` | Asynchronous post-turn significance computation |
| Prompt Injection | `features/language_model_plugin.py` | `_load_latent_episodes()` -- pre-LM gravity evaluation |
| Prompt Formatting | `features/prompt_optimizer.py` | `<<<LATENT_EPISODES>>>` block construction |
| Persistence | `data/aime_ledger.db` | `records.significance` column (permanent annotation) |

### 7.3 Configuration

All parameters are configurable at runtime via `lm_tuning.json`:

```json
{
  "significance_scorer": {
    "weights": { "affect": 0.30, "novelty": 0.30, "resolution": 0.25, "echo": 0.15 }
  },
  "gravity": {
    "threshold": 0.35,
    "top_k": 3,
    "connectivity_multiplier": 1.5,
    "half_life_days": 30
  }
}
```

Weight changes take effect immediately on future computations. Significance scores already written to the ledger are not retroactively recomputed -- this is a deliberate design choice preserving the historical record.

### 7.4 Operational Observations

**Affect as a differentiator.** The affect component produces the largest variance in significance scores across turns. Most conversational turns are affectively neutral ($affect < 0.10$), making the emotionally significant turns stand out sharply. This is the intended behavior -- high-affect turns should be rare and memorable, just as they are in human experience.

**Resolution as chapter marker.** Turns with $resolution = 1.0$ reliably correspond to narrative inflection points -- moments the user would recognize as "when I decided" or "when it was settled." These turns serve as natural anchors in the memory landscape, frequently cited by later turns and surfacing as latent episodes in related future conversations.

**Echo and concern dynamics.** The echo component creates a temporal amplification effect: when a user opens a concern, subsequent related turns receive elevated significance. When the concern closes (resolution detected), the amplification stops. This produces a natural narrative arc in the significance timeline -- concern-related turns cluster as high-significance episodes, then the cluster closes.

**Gravity diversity.** The sublinear amplification ($\sqrt{}$) successfully prevents dominance by any single record. In practice, the top-3 latent episodes selected on any given turn typically represent diverse time periods and topics, connected through the Bond state rather than through topical similarity to each other.

---

## 8. Limitations and Future Work

### 8.1 Current Limitations

**Lexical affect scoring.** The affect component uses dictionary lookup, which misses context-dependent emotional intensity ("fine" can be neutral or loaded depending on context) and novel emotional expressions not in the dictionary.

**Binary resolution detection.** Resolution is scored as 0 or 1 based on phrase patterns. Partial resolution ("I'm leaning toward..."), resolution with reservations ("I decided, but I'm not sure"), and implicit resolution (behavior change without verbal announcement) are not captured.

**Fixed half-life.** The 30-day decay half-life is a single global parameter. In practice, different types of memories may deserve different decay rates -- identity facts should decay slower than casual observations, and emotionally significant memories should persist longer than informational ones.

### 8.2 Future Directions

**Adaptive decay.** Per-record decay rates calibrated by significance and intent class would produce more nuanced temporal dynamics. A high-significance identity turn could have an effective half-life of 180 days while a low-significance general turn decays with a 7-day half-life.

**Embedding-based affect.** Replacing lexical affect scoring with an embedding-based detector trained on annotated conversational data would capture context-dependent emotional intensity and novel expressions.

**Multi-signal resolution.** Expanding resolution detection beyond binary phrase matching to include behavioral signals (topic disappearance, affect reduction, follow-through mentions) would capture implicit and partial resolution events.

**Gravity as training signal.** Records that consistently surface as latent episodes and contribute to high-quality responses (as measured by RIC scores [Canady, 2026c]) could be identified as high-value training examples for specialist model fine-tuning.

---

## 9. Conclusion

The surfacing problem -- which memories to present to a language model on any given turn -- is the operational bottleneck of persistent conversational AI. Storage is cheap; selection is hard. Gravity-Weighted Significance addresses this by decomposing the problem into two stages: a permanent importance annotation (significance) computed once at recording time, and a dynamic relevance score (gravity) computed at retrieval time by integrating importance with citation, connectivity, and temporal decay.

The result is a memory system that surfaces records not because they match the current query, but because they carry gravitational weight in the context of the ongoing relationship. A turn where the user shared difficult news three weeks ago surfaces today not because today's message mentions that topic, but because the turn's high affect score, its connection to an open concern, and its backward citations from later conversations give it sufficient gravity to enter the model's awareness.

This is the computational equivalent of a memory that stays with you -- not because you search for it, but because it mattered.

---

## References
Canady, J. Jr. (2026a). Bond-Indexed Memory: Relational State as the Organizing Principle for AI Memory Retrieval. Unpublished manuscript, AI nHancement. Public copy: https://www.ai-nhancement.com/paper/bond-indexed-memory.
Canady, J. Jr. (2026c). Relational Integrity Coefficient: Measuring Behavioral Trustworthiness in Persistent AI Systems. Unpublished manuscript, AI nHancement. Public copy: https://www.ai-nhancement.com/paper/relational-integrity-coefficient.
Chevalier, A., Wettig, A., Ajith, A., & Chen, D. (2023). Adapting Language Models to Compress Contexts. *Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing (EMNLP)*. arXiv:2305.14788.
Karpukhin, V., Oguz, B., Min, S., Lewis, P., Wu, L., Edunov, S., ... & Yih, W. (2020). Dense Passage Retrieval for Open-Domain Question Answering. *Proceedings of the 2020 Conference on Empirical Methods in Natural Language Processing (EMNLP)*. arXiv:2004.04906.
Lewis, P., Perez, E., Piktus, A., Petroni, F., Karpukhin, V., Goyal, N., ... & Kiela, D. (2020). Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. *Advances in Neural Information Processing Systems (NeurIPS) 33*. arXiv:2005.11401.
Sukhbaatar, S., Szlam, A., Weston, J., & Fergus, R. (2015). End-To-End Memory Networks. *Advances in Neural Information Processing Systems (NeurIPS) 28*. arXiv:1503.08895.
---

*Correspondence: AI nHancement -- john@ainhancement.com*
