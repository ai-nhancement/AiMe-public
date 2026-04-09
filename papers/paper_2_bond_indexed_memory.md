# Bond-Indexed Memory: Relational State as the Organizing Principle for AI Memory Retrieval

**Authors:** John Canady Jr.
**Affiliation:** AI nHancement, Independent Research
**Date:** April 2026
**Public Copy:** https://www.ai-nhancement.com/paper/bond-indexed-memory

---

## Abstract

We present Bond-Indexed Memory, a novel memory retrieval architecture for persistent conversational AI systems in which the primary organizing index for stored episodic records is not a topical keyword or semantic query vector, but a persistent relational state object called the Bond. The central innovation is an inversion of the conventional retrieve-then-respond paradigm: the system does not query memory for a response; it enters a relational state, and memories surface as a structural consequence of that state activation. This produces a fundamentally different class of memory access -- one that mirrors the associative, relationship-primed recall that characterizes human episodic memory. We describe the formal architecture of the Bond object (a six-dimensional relational state representation), the state activation mechanism that replaces query-based retrieval, and the portrait-connected traversal method that scores memory records by their connectivity to the active relational context rather than their semantic similarity to the current utterance. The system is implemented and operational in a live single-user cognitive AI system, where it constructs prompt context from memory without explicit user request across multiple language model backends. We present the paper as an architectural case study for persistent single-user systems rather than as a benchmark claim over general-purpose retrieval pipelines. We argue that for persistent AI relationships, relational state can be a more appropriate memory index than topical similarity, and that this architectural choice can produce more contextually appropriate conversational recall.

---

## 1. Introduction

Memory-augmented language model systems have converged on a common architecture: Retrieval-Augmented Generation (RAG). The user issues a message, the message is encoded as a vector, a similarity search retrieves the top-N most semantically similar records from a stored index, and those records are injected into the language model's prompt context [Lewis et al., 2020; Borgeaud et al., 2022]. This approach is effective for knowledge-grounded question answering, where the user's query is a reliable proxy for what information is needed.

It is not effective for persistent conversational relationships.

Consider a user who says to their AI assistant: "I'm thinking about this weekend." In a RAG system, this message is encoded and matched against stored records about weekends -- past weekend plans, weather forecasts, calendar events. But the user's actual cognitive context may be entirely different. They may be anxious about a family visit they mentioned three weeks ago. They may be excited about a project milestone that concludes Friday. They may be worried about their dog's upcoming surgery, mentioned once in passing two months prior.

A human conversational partner who knows this user well would not retrieve "records about weekends." They would activate their internal model of this person -- their knowledge of who they are, what they care about, what they are currently worried about -- and relevant memories would arise naturally within that activated relational context. The weekend mention would trigger associations not by topical similarity but by relational connectivity: "this person has been worried about the vet appointment, and the surgery is scheduled for Saturday."

This paper describes a system that implements the formal equivalent of this process. The primary contribution is the Bond-Indexed Memory architecture, in which memory retrieval is organized by a persistent relational state object rather than by query-based semantic search.

---

## 2. Limitations of Query-Based Retrieval

### 2.1 Topical Bias

Standard RAG retrieval is biased toward surface-level semantic similarity between the current message and stored content. The encoding of "I'm thinking about this weekend" will match records containing weekend-related vocabulary, not records about anxiety, family relationships, or veterinary appointments -- even when those are the most contextually relevant records for this particular user at this particular moment.

This is not a failure of the embedding model. It is a failure of the retrieval premise: the assumption that the current utterance is a sufficient specification of what information is needed. In persistent relationships, the current utterance is often a thin surface over a deep relational context that the retrieval system cannot access through similarity search alone.

### 2.2 Context Blindness

Query-based retrieval maintains no persistent model of the user's psychological or relational state between turns. Each query is evaluated in isolation against the raw embedding index. The system does not know that the user has an open concern about their job, that they mentioned their sister is visiting next month, or that their affect has been elevated for the past several sessions. Without this context, retrieval operates on the text of the message rather than the meaning of the message within the relationship.

### 2.3 Stateless Retrieval

RAG systems do not remember what has already been surfaced. The same records are retrieved on successive turns if the user's messages remain topically similar. This produces a "stuck" quality -- the system keeps presenting the same information because the same embeddings keep matching. There is no mechanism for novelty, progression, or contextual evolution in what the memory system surfaces.

### 2.4 The Core Realization

The present work is based on the following insight:

> In human memory, the primary act of meaningful recall is not issuing a query. It is entering a relationship context -- activating a persistent internal model of another person. Memories then surface as structural consequences of that activated state.

A person does not think "search my memories for facts about my sister" before having a conversation with their sister. They enter a relational state -- an awareness of their sister as a person with history, shared experiences, emotional significance, and current concerns -- and relevant memories arise automatically and appropriately within that activated state.

Bond-Indexed Memory implements the formal equivalent of this process for a conversational AI system.

---

## 3. The Bond Object

### 3.1 Definition

The Bond is a persistent, evolving data structure representing the AI system's internal model of its relationship with the user. It is not a user profile, a preference store, or a conversation summary. It is a relational state object -- a structured representation of who the user is, what they care about, who matters to them, and what patterns characterize their interaction with the system.

### 3.2 Six Dimensions

The Bond comprises six named dimensions, each independently maintained and updated:

| Dimension | Type | Content |
|-----------|------|---------|
| **Identity Anchors** | Key-value store | Named facts about the user: name, profession, preferences, personal history. Each anchor carries a value, timestamp, confidence score, and importance weight. |
| **Active Concerns** | Dynamic stack | Open threads -- topics the user is actively thinking about, worried about, or has not yet resolved. Concerns open and close dynamically based on linguistic signals. |
| **Relational Graph** | Named graph | People the user has referenced. Each node tracks mention count, emotional register (exponential moving average), relationship arc, and shared vocabulary. |
| **Pattern Layer** | Statistical model | Behavioral regularities: response timing, topic recurrence, message-length distribution. |
| **Commitment Layer** | Tracked obligations | Promises or obligations the user or system has stated, with optional deadlines and follow-up signals. |
| **Behavioral Fingerprint** | Session signature | Turn density, average significance, correction frequency, affective tone distribution. |

### 3.3 Persistence Properties

The Bond persists independently of which language model backend is active. Swapping the underlying model (from GPT to Claude to Gemini to a local model) does not reset or alter the Bond state. This is a critical design property: the relational context is a property of the relationship, not of the model. A person's knowledge of their friend does not reset when they switch phones.

The Bond is stored in a key-value persistence layer within the system's append-only evidence ledger, keyed by session identifier. It is loaded once at session start, updated incrementally after each turn via asynchronous write-paths, and preserved across system restarts.

### 3.4 The Concern Stack

The Active Concerns dimension implements dynamic open-thread tracking that is central to the retrieval mechanism:

**Opening signals** (detected by lexical pattern matching): "thinking about," "worried about," "not sure whether," "I keep coming back to," "can't decide on."

**Closing signals** (take priority): "decided," "settled on," "figured out," "resolved," "moving on from."

Each concern record contains: topic string, first-raised timestamp, update history, and embedding vector. Concern deduplication uses cosine similarity (threshold > 0.60) to prevent re-opening the same concern expressed with different surface phrasing.

The concern stack is the most dynamic dimension of the Bond and the most influential on memory retrieval. An open concern creates a persistent attentional bias -- records connected to open concerns receive elevated retrieval scores regardless of their topical similarity to the current utterance.

---

## 4. State Activation Mechanism

### 4.1 The Architectural Inversion

| Conventional System | Bond-Indexed System |
|---------------------|---------------------|
| Store -> Query -> Retrieve -> Respond | Enter Bond State -> Surface -> Respond |
| Query is the primary act | State entry is the primary act |
| Records matched to query text | Records matched to Bond state |
| Retrieval is per-turn, stateless | Bond is persistent, stateful |
| Memory is a database to search | Memory is a relationship-indexed field to enter |

### 4.2 Three-Step Activation Process

State activation occurs at the beginning of each turn, before the language model is called:

**Step 1 -- Bond Loading.** The current Bond object is loaded from the persistence layer into working memory. This is a single key-value lookup -- O(1) in the absence of eviction. The full six-dimensional state is available for the remainder of the turn.

**Step 2 -- Portrait-Connected Traversal.** Memory records in the evidence ledger are scored not by raw semantic similarity to the current user message, but by their connectivity to the loaded Bond object. A record is considered "Bond-connected" if its content overlaps with any of:
- Any identity anchor key or value
- Any active concern topic
- Any relational graph node name

Bond-connected records receive a connectivity multiplier (default 1.5x) in the gravity scoring function. This means a record about the user's sister -- even if the current message is about weekend plans -- will receive elevated retrieval weight because "sister" is a node in the relational graph.

**Step 3 -- Latent Episode Surfacing.** Records that score above a gravity threshold (default 0.35) are selected as **latent episodes** -- contextually salient memories that surface into the language model's prompt context without being explicitly requested by the user. These are injected as a structured block in the system prompt, positioned after the Bond portrait and before the user's current message.

### 4.3 Latent Episodes as Unsolicited Recall

The term "latent episode" is chosen deliberately. These are not query results -- the user did not ask for them. They are memories that the system's relational state determined to be relevant. The language model receives them as implicit context, free to reference them naturally or ignore them, just as a human might recall a relevant memory during conversation without being asked about it.

The prompt injection format:

```
<<<LATENT_EPISODES v=1 n=N>>>
[intent_type] record_content_snippet [sig=0.XX]
[intent_type] record_content_snippet [sig=0.XX]
...
<<<END_LATENT_EPISODES>>>
```

This block is invisible to the user and the model is not instructed to reference it explicitly. It primes the model's context with relationship-relevant memories before generation begins.

---

## 5. Bond Update Pipeline

### 5.1 Asynchronous, Post-Turn Updates

The Bond is read-only during inference -- no writes occur during or after the language model call itself. All updates are asynchronous and post-response, executed on daemon threads:

1. **Portrait update** -- writes new facts from the current turn's fact extraction into the appropriate Bond dimensions
2. **Concern update** -- opens or closes concern records based on detected linguistic signals
3. **Consolidation** -- periodic re-processing of past evidence records against the evolving Bond, advancing a watermark to prevent redundant reprocessing

### 5.2 Fact Routing

Extracted facts from a user turn are routed into Bond dimensions by type:

- **Relational facts** (sister/brother/wife/partner/friend/colleague + name) -> Relational Graph
- **Identity facts** (all others) -> Identity Anchors
- **High-significance facts** (significance $\geq$ 0.65) -> receive an importance weight boost (+0.2)

Significance scoring -- which determines how important a turn is for memory persistence -- is described in a companion paper [Canady, 2026b].

### 5.3 Mention Tracking

The relational graph accumulates context automatically. When the user mentions a person ("Karra had a school event"), the system:
1. Creates or updates the relational node for "Karra"
2. Increments the mention count
3. Updates the emotional register (exponential moving average of affect detected in mentions)
4. Accumulates context keywords from the mention context ("school," "event")

Over time, each relational node develops a rich context vocabulary that influences retrieval. When the user later mentions "school" in an unrelated context, the retrieval system connects it to Karra through the accumulated vocabulary -- not through semantic similarity of the current message, but through the relational graph's accumulated context.

---

## 6. Comparison to Prior Art

### 6.1 Retrieval-Augmented Generation (RAG)

RAG [Lewis et al., 2020] retrieves by encoding the current query and performing similarity search. Bond-Indexed Memory retrieves by activating a persistent relational state. RAG is stateless across turns; Bond-Indexed Memory accumulates relational context. RAG deduplicates by result ranking; Bond-Indexed Memory deduplicates through a rolling window that tracks what has already been surfaced.

### 6.2 Memory-Augmented Transformers

MemTRM [Wu et al., 2022] and similar architectures augment the transformer's attention mechanism with external memory banks. These operate at the token level within a single forward pass. Bond-Indexed Memory operates at the turn level across an entire conversation history, injecting selected records into the prompt context before the forward pass begins.

### 6.3 Personalization Systems

User profile systems [Zhang et al., 2018] maintain static attribute stores (age, location, preferences) for response personalization. The Bond is not a profile -- it is a dynamic relational state that evolves with every turn, tracks open concerns, maintains a social graph, and influences memory retrieval through state activation rather than attribute matching.

### 6.4 Summary Comparison

| Property | RAG | Memory-Augmented Transformers | User Profiles | Bond-Indexed Memory |
|----------|-----|-------------------------------|---------------|---------------------|
| Index type | Embedding similarity | Token-level attention | Attribute matching | Relational state |
| Persistence | None (per-query) | Within context window | Static store | Evolving per-turn |
| Concern tracking | None | None | None | Dynamic open-thread stack |
| Person graph | None | None | Possible (flat) | Named graph with salience |
| Model independence | N/A | Model-specific | N/A | Persists across model swaps |
| Unsolicited recall | None | None | None | Latent episodes |

---

## 7. Implementation and Evaluation

### 7.1 System Context

Bond-Indexed Memory is implemented within AiMe, a Memory-Augmented Cognitive Intelligence (MACI) system for persistent single-user interaction. The evidence ledger contains tens of thousands of records accumulated across months of daily interaction. The system operates across multiple language model backends (Azure GPT, Anthropic Claude, Google Gemini, xAI Grok, local models via Ollama), with the Bond persisting identically across all of them.

### 7.2 Key Implementation Components

| Component | File | Role |
|-----------|------|------|
| Bond object | `modules/portrait/cognitive_portrait.py` | Six-dimensional relational state dataclass |
| Concern tracking | `modules/portrait/concerns.py` | Open/close signal detection, deduplication |
| Gravity scoring | `modules/portrait/gravity.py` | Portrait-connectivity scoring, latent episode selection |
| Turn-exit hooks | `features/cognitive_bridge.py` | Asynchronous Bond updates (portrait, concerns) |
| Consolidation | `features/cerebral_cortex.py` | Periodic re-processing with watermark |
| Prompt injection | `features/prompt_optimizer.py` | `<<<LATENT_EPISODES>>>` block construction |
| Persistence | `data/aime_ledger.db` | `identity_kv` table (Bond storage) |

### 7.3 Qualitative Observations

The most striking property of Bond-Indexed Memory in live operation is the naturalness of recall. When the user mentions a person, the system surfaces relevant memories about that person -- not because the user asked, but because the relational graph connected the mention to stored episodes. When the user expresses concern about a topic, memories related to that concern surface in subsequent turns even when the user's explicit messages are about something else.

Users do not notice the retrieval mechanism. They notice that the system "remembers" and "understands" -- that it brings up relevant context without being asked, connects current topics to past conversations, and maintains awareness of ongoing concerns. This is precisely the design goal: memory retrieval that feels like relationship, not search.

### 7.4 Governance Properties

The Bond architecture includes strict governance constraints:

- The Bond is **read-only during inference** -- no writes occur during or after the language model call
- All writes are asynchronous and post-response only
- The Bond contains no model weights, no gradient signals, and no training data -- it is a pure relational state record
- The system does not use Bond state for reinforcement learning or model fine-tuning
- Memory surfacing is deterministic given the current Bond state -- the same Bond + evidence store produces the same latent episodes

---

## 8. Limitations and Future Work

### 8.1 Current Limitations

**Single-user scope.** The Bond is designed for one-to-one persistent relationships. Multi-user or group conversation support would require a federation of Bond objects with interaction rules between them.

**Linguistic signal detection.** Concern opening and closing rely on lexical pattern matching. Subtle concern signals -- tone shifts, topic avoidance, increased message brevity -- are not detected by the current system. A more sophisticated signal detector would improve concern tracking fidelity.

**Cold start.** A new Bond has empty dimensions. Until sufficient interaction data accumulates, retrieval falls back toward conventional similarity-based methods. The bootstrapping period (approximately 20-30 substantive turns) is a practical limitation for new relationships.

### 8.2 Future Directions

**Concern inference.** Moving beyond lexical detection to infer concerns from behavioral patterns -- message timing changes, topic avoidance, affect shifts -- would enable the system to detect concerns the user has not explicitly stated.

**Multi-modal Bond dimensions.** Incorporating visual context (shared images, environmental awareness via camera) into the Bond's relational graph would enrich the state activation mechanism with non-verbal relational signals.

**Federated specialist retrieval.** In a multi-agent architecture, different specialist agents could traverse the Bond from different perspectives -- a Memory Agent prioritizing episodic recall, an Experience Agent prioritizing affective significance -- and their results aggregated before prompt injection.

---

## 9. Conclusion

Bond-Indexed Memory replaces the question "what is semantically similar to this query?" with "within this relationship, what matters now?" The architectural inversion -- from query-based retrieval to state-activated surfacing -- produces a class of memory access that more closely mirrors human episodic recall in relational contexts.

The practical consequence is a conversational AI system that remembers in a way that feels relational rather than mechanical. It connects a mention of "this weekend" to a veterinary appointment not because the words are similar, but because the user's active concern about their dog's health is part of the relational state, and the appointment is connected to that concern in the evidence store.

For persistent AI systems designed to maintain long-term relationships, we argue that relational state is the correct organizing principle for memory. The Bond is not a feature added to a retrieval system -- it is the retrieval system's index.

---

## References
Borgeaud, S., Mensch, A., Hoffmann, J., Cai, T., Rutherford, E., Millican, K., ... & Sifre, L. (2022). Improving Language Models by Retrieving from Trillions of Tokens. *Proceedings of the 39th International Conference on Machine Learning (ICML)*. arXiv:2112.04426.
Canady, J. Jr. (2026b). Gravity-Weighted Significance: Automatic Memory Salience in Long-Context Conversational AI. Unpublished manuscript, AI nHancement. Public copy: https://www.ai-nhancement.com/paper/gravity-weighted-significance.
Lewis, P., Perez, E., Piktus, A., Petroni, F., Karpukhin, V., Goyal, N., ... & Kiela, D. (2020). Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. *Advances in Neural Information Processing Systems (NeurIPS) 33*. arXiv:2005.11401.
Wu, Y., Rabe, M. N., Hutchins, D., & Szegedy, C. (2022). Memorizing Transformers. *International Conference on Learning Representations (ICLR)*. arXiv:2203.08913.
Zhang, S., Dinan, E., Urbanek, J., Szlam, A., Kiela, D., & Weston, J. (2018). Personalizing Dialogue Agents: I have a dog, do you have pets too? *Proceedings of the 56th Annual Meeting of the Association for Computational Linguistics (ACL)*. arXiv:1801.07243.
---

*Correspondence: AI nHancement -- john@ainhancement.com*
