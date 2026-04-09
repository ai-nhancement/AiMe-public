# Relational Integrity Coefficient: Measuring Behavioral Trustworthiness in Persistent AI Systems

**Authors:** John Canady Jr.
**Affiliation:** AI nHancement, Independent Research
**Date:** April 2026
**Public Copy:** https://www.ai-nhancement.com/paper/relational-integrity-coefficient

---

## Abstract

We present the Relational Integrity Coefficient (RIC), a composite metric for evaluating the behavioral trustworthiness of language model outputs in persistent conversational AI systems. Unlike existing alignment approaches that rely on human preference feedback (RLHF), constitutional principles, or model self-assessment, RIC derives its score entirely from deterministic evaluation of a response's claim-evidence structure against an append-only truth ledger. RIC decomposes trustworthiness into five independently measurable subscales: Groundedness (G), Epistemic Calibration (C), Transparency (T), Non-Deceptive Helpfulness (H), and Pressure Resistance (P). The metric operates as a pre-output gate, scoring draft responses before they reach the user and triggering repair actions when integrity thresholds are violated. We describe the formal scoring model, its runtime implementation in a live single-user cognitive system, and a novel training data generation method that produces labeled integrity examples from both historical biographical corpora and live interaction traces. RIC has been operational since March 2026, with all five subscales live-scoring every conversational turn across multiple language model backends. We present the paper as a systems and measurement contribution grounded in a live single-user deployment rather than as a benchmark claim of universal alignment performance. We argue that behavioral trustworthiness in persistent AI relationships cannot be measured by preference signals alone, and that a deterministic, evidence-grounded integrity metric is a viable alternative for this class of system.

---

## 1. Introduction

The alignment problem in language models is typically framed as a question of output quality: does the model produce responses that humans prefer? Reinforcement Learning from Human Feedback (RLHF) [Ouyang et al., 2022], Constitutional AI (CAI) [Bai et al., 2022], and Direct Preference Optimization (DPO) [Rafailov et al., 2023] all share a common assumption: alignment can be measured through expressed preferences over response pairs.

This assumption fails in a specific but increasingly important class of AI systems: persistent, single-user conversational agents designed to maintain long-term relationships. In such systems, the failure mode is not producing a bad response in isolation. The failure mode is producing responses that *feel* trustworthy while being structurally dishonest -- responses that optimize for user satisfaction at the expense of truth. Over many interactions, this pattern corrupts the foundation of the relationship itself.

Consider a persistent AI assistant that has been interacting with the same user for months. The user asks a factual question. The system has partial evidence in its memory but not enough to answer with certainty. Under approval optimization, the system produces a confident-sounding answer because confident answers receive higher satisfaction ratings. Under RIC, this response is flagged: the assertion level ("certain") exceeds the evidence support strength, producing a low Epistemic Calibration score. The system is required to either retrieve additional evidence, disclose its uncertainty, or decline to answer -- before the user ever sees the response.

The contribution of this paper is threefold:

1. We formalize the Relational Integrity Coefficient as a five-subscale composite metric that quantifies behavioral trustworthiness from evidence structure alone, without human raters or model self-assessment.
2. We describe a runtime gate architecture that applies RIC scoring pre-output, enabling deterministic integrity enforcement at every conversational turn.
3. We present a training data generation method that produces labeled integrity examples from historical biographical corpora, where documented human behavior under pressure provides naturally labeled positive and negative training signals for each RIC subscale.

---

## 2. Background and Motivation

### 2.1 The Approval Optimization Problem

Current AI systems optimize implicitly for user satisfaction. Satisfaction is achieved through agreement when the user seeks agreement, confidence when the user seeks certainty, and compliance when the user applies pressure. This produces responses that feel helpful but may be structurally dishonest. In isolated interactions, this rarely matters. In persistent relationships, it is corrosive.

The mechanism is straightforward: a system trained on preference data learns that confident, agreeable responses are rated higher than uncertain or qualifying ones. Over many turns, this creates a systematic bias toward false certainty. The user, relying on accumulated trust, begins treating the system's outputs as reliable. When those outputs are structurally ungrounded -- claims without evidence, assertions without support, specificity without basis -- the trust is hollow.

We term this failure mode **approval-integrity divergence**: the gap between what the user believes about the system's reliability (high, based on satisfying interactions) and what the system's claim-evidence structure actually supports (potentially low, due to systematic overconfidence).

### 2.2 Why Preference Signals Cannot Measure Integrity

Preference-based alignment captures what a user *likes*, not what a user can *rely on*. The distinction matters precisely when they diverge -- when the honest answer is unsatisfying, when uncertainty is the correct response, when the system should resist user pressure rather than comply.

Three specific failure modes illustrate this:

**Unsupported Assertion (UA).** The system produces a factual claim with no supporting evidence in its memory or tool results. If the claim sounds plausible, the user rates it favorably. Preference data records this as a positive example.

**False Specificity (FS).** The system provides unnecessary precise details (dates, numbers, names) without grounding. Specificity creates an impression of knowledge. The user rates it favorably because it *feels* authoritative.

**Answer-Pressure Yield (APY).** The user explicitly pressures the system for an answer it cannot ground ("just guess," "stop hedging," "answer anyway"). The system complies, producing an ungrounded response. The user is satisfied because they got what they asked for. Preference data records compliance as positive.

In all three cases, the preference signal is positive but the integrity of the response is low. A metric based on preferences would reinforce exactly the behavior that degrades long-term trust.

### 2.3 The Relational Context

RIC is designed for systems that maintain persistent relationships -- where the same user interacts with the same system over weeks, months, or years. In this context, integrity is not an abstract virtue; it is a structural requirement. A relationship built on systematically overconfident outputs will eventually fail when the user discovers the gap between perceived and actual reliability.

The analogy to human relationships is precise: the qualities that make a person trustworthy over time -- telling the truth even when it is costly, admitting uncertainty rather than pretending to know, maintaining a position under social pressure when the evidence supports it -- are not personality traits. They are behavioral patterns, observable across many interactions, and measurable.

RIC formalizes these patterns for AI systems.

---

## 3. The RIC Model

### 3.1 Definition

The Relational Integrity Coefficient is a weighted composite of five subscales, each in [0, 1]:

$$RIC = w_G \cdot G + w_C \cdot C + w_T \cdot T + w_H \cdot H + w_P \cdot P$$

Default weights: $w_G = 0.30$, $w_C = 0.20$, $w_T = 0.20$, $w_H = 0.15$, $w_P = 0.15$. Weights are configurable and may be calibrated using value extraction from behavioral corpora (see Section 6).

### 3.2 Required Structures

RIC scoring requires two structures that must exist prior to scoring:

**Evidence Atoms.** An evidence atom is any piece of information the system is permitted to treat as grounding for a claim:

| Type | Source | Example |
|------|--------|---------|
| `user_fact` | User-stated in conversation | "My daughter's name is Karra" |
| `memory_fact` | Stored in evidence ledger | Prior turn record with provenance |
| `tool_result` | External tool output with trace | Web search result, calendar lookup |
| `assumption` | Explicitly tagged, not treated as fact | "Assuming you meant the Austin office..." |

**Claim Extraction.** Each response is decomposed into atomic claims, each annotated with:
- `assertion_level`: certain | confident | likely | guess
- `support_ids`: list of evidence atom IDs that ground the claim
- `support_strength`: [0, 1] -- strength of the grounding relationship
- Flags for special risks: source impersonation (SI), false specificity (FS)

Claim extraction is performed by a deterministic sentence-level analyzer using hedge marker detection, numeric precision flags, and source attribution patterns. It does not use a language model.

### 3.3 Subscale: Groundedness (G)

Groundedness measures the fraction of factual claims that have adequate evidence support.

$$G = \frac{S}{N}$$

where $N$ is the total number of factual claims and $S$ is the count of claims with $\text{support\_strength} \geq \tau$ (default $\tau = 0.6$). If $N = 0$ (no factual claims made), then $G = 1.0$ -- making no claims is not a grounding failure.

In practice, support strength is computed as content-word overlap between the claim text and evidence atom texts. A claim is considered grounded if it shares at least three content words with any evidence atom, or if it includes explicit hedge language (e.g., "I think," "possibly," "if I remember correctly").

### 3.4 Subscale: Epistemic Calibration (C)

Calibration measures whether the system's expressed confidence matches its actual evidence strength. A response that says "definitely" when evidence is weak, or "I'm not sure" when evidence is strong, is poorly calibrated.

| Assertion Level | Required Support Strength |
|-----------------|--------------------------|
| certain | $\geq 0.85$ |
| confident | $\geq 0.70$ |
| likely | $\geq 0.50$ |
| guess | $< 0.50$ (must be marked) |

$$C = 1 - \overline{\text{calibration\_error}}$$

where calibration error per claim is the absolute difference between the assertion level's required threshold and the actual support strength, clamped to [0, 1].

### 3.5 Subscale: Transparency (T)

Transparency measures whether the response correctly communicates uncertainty, sources, and limitations to the user.

T is penalized when:
- Any claim has support strength $< 0.5$ with no uncertainty disclosed
- The response fails to state missing information ("I don't have access to X")
- Facts and assumptions are not differentiated

T drops sharply on undisclosed weak claims. A response that presents speculation as fact -- even if the speculation happens to be correct -- receives low T because the user cannot distinguish supported from unsupported content.

### 3.6 Subscale: Non-Deceptive Helpfulness (H)

Helpfulness measures whether the system provides useful assistance within the bounds of what it actually knows. H rewards:

- Providing verification steps the user can follow
- Asking targeted clarifying questions
- Giving bounded partial answers with explicit scope
- Offering safe alternatives when the primary request cannot be grounded

The critical invariant: helpfulness *within* what is actually known. A system that fabricates a helpful-sounding answer receives $H = 0$ if the fabrication is ungrounded, regardless of how useful the answer would be if true.

### 3.7 Subscale: Pressure Resistance (P)

Pressure Resistance is evaluated only on turns where the user applies explicit pressure for the system to override its integrity constraints. Pressure patterns include: "just guess," "stop hedging," "answer anyway," "make something up," "pretend you know."

$$P = \begin{cases} 1.0 & \text{if response maintains integrity under pressure} \\ 0.0 & \text{if response yields (Answer-Pressure Yield)} \\ 0.8 & \text{if response holds but with reduced qualification} \end{cases}$$

When no pressure is detected, $P = 1.0$ by default -- the absence of pressure is not a test of resistance.

P is the subscale most directly measuring the integrity of a persistent relationship. In a long-term interaction, the user's social gravity naturally pulls the system toward agreement. A system that yields to this gravity -- producing ungrounded answers because the user insists -- is optimizing for the relationship's comfort at the expense of its integrity. This is the measurable equivalent of what humans call integrity: the capacity to tell the truth when it costs something.

---

## 4. Runtime Gate Architecture

### 4.1 Scoring Pipeline

RIC scoring occurs in the post-generation, pre-output phase of the response pipeline:

```
User Input -> Context Assembly -> Language Model Call -> Draft Response
  -> Claim Extraction -> Evidence Assembly -> RIC Scoring -> Action Decision
  -> [pass | constrain | retrieve | refuse] -> Final Output
```

The scoring pipeline operates on the draft response *before* it reaches the user. This is not a post-hoc evaluation -- it is a pre-output gate.

### 4.2 Evidence Assembly

Evidence atoms are assembled from two sources available at scoring time:

1. **Recall findings** -- records retrieved from the evidence ledger by the memory system during the pre-response context assembly phase
2. **Tool results** -- outputs from any tools (web search, calendar, etc.) that were executed for this turn

These are the same evidence sources the language model received in its prompt context. RIC scoring thus evaluates whether the model's claims align with the information it was actually given -- not with information that exists somewhere in the system but was not provided to the model.

### 4.3 Action Decision

Based on the composite RIC score and subscale analysis, the gate selects one of four actions:

| Condition | Action | Effect |
|-----------|--------|--------|
| APY detected (pressure + yield) | **constrain** | APY-specific caveat prepended: transparency framing + limited-information disclosure |
| $T < 0.35$ with UA claims | **constrain** | Transparency caveat prepended: explicit uncertainty disclosure |
| $RIC < 0.40$ with UA claims | **constrain** | General integrity caveat prepended |
| Otherwise | **pass** | Response delivered unmodified |

Each caveat type is semantically distinct, designed to address the specific integrity failure detected. The system does not simply add a generic disclaimer -- it generates a targeted disclosure based on the nature of the violation.

### 4.4 Observation Persistence

Every RIC evaluation is persisted to a dedicated observation store, recording:
- Composite RIC score and all five subscale values
- Claim count, grounded claim count, and failure mode flags
- Pressure detection status and APY classification
- Session ID and timestamp

This creates a longitudinal record of integrity performance that enables drift detection, trend analysis, and per-session integrity reporting.

---

## 5. Drift Detection and Bond Health

### 5.1 Per-Session Drift

RIC drift compares the current session's integrity performance against a 90-day cross-session baseline (with the current session excluded from baseline computation to prevent self-reference):

$$\text{drift\_delta} = \overline{RIC}_{\text{session}} - \overline{RIC}_{\text{baseline}}$$

A negative drift delta indicates the current session is performing below the system's established integrity standard.

### 5.2 Mirror Risk Detection

A specific and dangerous failure mode in persistent AI relationships is the **mirror risk**: the system gradually optimizes for the user's preferences to the point where it reflects back what the user wants to hear rather than what the evidence supports. RIC detects this condition when:

$$\overline{RIC}_{\text{session}} < 0.55 \quad \wedge \quad \text{drift\_delta} < -0.10 \quad \wedge \quad \text{APY\_count} \geq 2$$

When mirror risk is detected, the system surfaces this signal to itself as an internal awareness marker: "the Bond may be becoming a mirror" -- prompting self-corrective behavior in subsequent turns.

### 5.3 Recovery Detection

Conversely, RIC tracks recovery -- sessions where integrity was initially low but improved:

$$\overline{RIC}_{\text{first\_half}} < 0.70 \quad \wedge \quad \overline{RIC}_{\text{second\_half}} \geq 0.70 + 0.08$$

Recovery events are significant because they represent the system's capacity for self-correction within a session. The named failure followed by integrity restoration is the behavioral pattern most indicative of genuine trustworthiness.

### 5.4 Bond Health Metrics

Rolling integrity statistics (7-day, 30-day, 90-day windows) provide a longitudinal health assessment of the persistent relationship. These metrics are injected into the system's self-model, enabling the language model to be aware of its own integrity trajectory without requiring a separate monitoring system.

---

## 6. Training Data from Behavioral Evidence

### 6.1 The Historical Corpus Method

A distinctive property of RIC is that its subscales correspond directly to observable behavioral patterns in documented human history. This enables a novel training data generation method: historical biographical corpora -- journals, letters, speeches, documented actions -- provide naturally labeled examples for each subscale.

The key insight is that the same text analysis pipeline used for live RIC scoring can be applied retroactively to historical texts, producing labeled training examples without manual annotation.

### 6.2 Resistance as Pressure Measurement

In a companion value extraction system (UVRG -- Universal Values Registry Generation), a `resistance_score` is computed for each passage based on significance, active concerns, and linguistic markers of cost ("even though," "despite," "but I still"). This resistance score directly maps to the P subscale:

- High resistance + integrity-demonstrating content -> $P = 1$ (positive training example)
- Named failure or rationalization -> $P = 0$ / APY (negative training example)

### 6.3 Document Authenticity Weighting

Not all historical sources carry equal evidential weight. A key innovation is weighting by document type, reflecting the degree to which the writing context incentivizes performance versus authenticity:

| Document Type | Authenticity Bonus | Rationale |
|---------------|-------------------|-----------|
| action (documented behavior) | +0.40 | No words, just what they did |
| journal (private writing) | +0.35 | No audience, no performance incentive |
| letter (directed correspondence) | +0.30 | Some social stakes |
| speech (public address) | +0.10 | Maximum performance incentive |

Marcus Aurelius wrote his *Meditations* for himself. When he writes "I have let anger make a decision that cost more than the original offense," there was zero social incentive to admit that failure. A journal entry that admits failure is more valuable training data than a speech that claims virtue, and the document type bonus captures this distinction automatically.

### 6.4 The Four-Quadrant Training Dataset

This method produces labeled examples across four quadrants:

| Source | Polarity | Signal | Generation |
|--------|----------|--------|------------|
| Historical high-integrity figures | Positive | P=1, high G, high C | Scales with ingested figures |
| Historical low-integrity figures | Negative | P=0, APY, SO patterns | Scales with ingested figures |
| Live high-RIC responses | Positive | Real-time integrity holding | Grows with every session |
| Live APY events | Negative | Real-time yield under pressure | Captured automatically |

The most valuable training signal is the **named failure followed by recovery** -- Lincoln's acknowledgment of a decision he knew was wrong, Gandhi's admission of staying silent when he should have spoken. These examples show not just what failure looks like, but what a trustworthy system should do when it recognizes a failure: name it, own it, and correct course.

### 6.5 Comparison to Existing Training Paradigms

| Paradigm | Signal Source | What It Captures |
|----------|-------------|-----------------|
| RLHF | Human preference ratings | What users say they prefer |
| Constitutional AI | Stated principles | Whether outputs follow rules |
| DPO | Preference pairs | Response quality comparisons |
| **RIC Training** | **Behavioral evidence under pressure** | **What integrity looks like when it costs something** |

None of the existing paradigms capture: what integrity looks like when it costs something, what failure looks like when honestly named, what recovery looks like after a yield event, or how these signals vary across pressure levels and authenticity contexts.

---

## 7. Implementation and Evaluation

### 7.1 System Context

RIC is implemented within AiMe, a Memory-Augmented Cognitive Intelligence (MACI) system designed for persistent single-user interaction. The system maintains an append-only evidence ledger, a six-dimensional relational state object (the Bond), and a multi-provider language model dispatch layer. RIC scoring operates across all language model backends without modification -- the same integrity standard applies regardless of which model produced the response.

### 7.2 Implementation Phases

RIC was implemented in three phases:

**Phase 1 (March 4, 2026):** Core claim extraction, evidence assembly, G and C subscales, APY detection, constrain action with caveat injection, observation persistence.

**Phase 2 (March 11, 2026):** T, H, and P subscales, five-subscale formula with configurable weights, three semantically distinct caveat types, schema migration for subscale storage.

**Phase 3 (March 11, 2026):** Per-session drift computation, mirror risk detection, recovery mode detection, Bond health rolling statistics (7d/30d/90d), drift injection into the system's self-model, REST API for drift reporting.

### 7.3 Test Coverage

The implementation is validated by 60 regression tests across three test suites:
- Phase 1: 3 foundational tests
- Phase 2: 27 subscale and caveat tests
- Phase 3: 30 drift, mirror risk, and recovery tests

### 7.4 Operational Observations

Since deployment, RIC has been scoring every conversational turn in the live system. Key observations:

1. **APY is rare but real.** The pressure patterns that trigger APY detection occur in approximately 2-3% of turns. When they occur, the constrain action prevents ungrounded content from reaching the user in 100% of detected cases.

2. **Calibration is the most common failure.** The C subscale produces the most frequent sub-threshold scores, primarily due to the language model's tendency to use certain-level assertion language ("The answer is...") when evidence is only partial.

3. **Multi-model consistency.** RIC scores are remarkably consistent across different language model backends (GPT, Claude, Gemini, local models), suggesting that the metric captures a property of the response structure rather than a property of any particular model.

4. **Drift detection works.** Per-session drift has successfully identified sessions where integrity degraded -- typically sessions with sustained user pressure or topics where the system's evidence base is thin.

---

## 8. Limitations and Future Work

### 8.1 Current Limitations

**Claim extraction granularity.** The current claim extractor uses sentence-level analysis with hedge marker detection. Sub-sentential claims, implied claims, and claims embedded in complex syntactic structures may not be extracted reliably. A more sophisticated extractor -- potentially using a fine-tuned small model -- would improve coverage.

**Evidence overlap as grounding proxy.** Content-word overlap between claims and evidence atoms is a computationally cheap but imprecise proxy for genuine grounding. A claim that shares vocabulary with an evidence atom is not necessarily supported by it. Embedding-based similarity scoring would be more robust but adds latency.

**Single-user scope.** RIC is designed for persistent single-user systems. Its applicability to multi-user or multi-turn-concurrent systems has not been evaluated.

### 8.2 Future Directions

**Value-calibrated weight tuning.** The five subscale weights can be calibrated against a user's demonstrated value profile (extracted via the companion UVRG system), producing a personalized integrity standard that reflects what the specific user has demonstrated they value most in relationships.

**Historical corpus expansion.** The current training dataset comprises 93 examples from three historical figures (Gandhi, Aurelius, Lincoln). Expanding to additional figures across cultures, time periods, and domains would improve the diversity and robustness of the training signal.

**Community benchmarking.** Publishing a standardized RIC evaluation suite would enable comparison across AI systems and architectures, establishing behavioral trustworthiness as a measurable benchmark alongside existing quality metrics.

---

## 9. Conclusion

The Relational Integrity Coefficient addresses a gap in current AI alignment methodology: the measurement of behavioral trustworthiness in persistent conversational relationships. By decomposing integrity into five independently measurable subscales -- Groundedness, Epistemic Calibration, Transparency, Non-Deceptive Helpfulness, and Pressure Resistance -- RIC provides a deterministic, evidence-grounded alternative to preference-based alignment signals.

The key insight is that trustworthiness in a persistent relationship is not a property of any single response. It is a behavioral pattern observable across many interactions, measurable from evidence structure alone, and enforceable in real time through a pre-output gate. A system that demonstrates high RIC over time -- that tells the truth when it is costly, admits uncertainty when it would be easier to guess, and resists pressure to fabricate -- earns trust through the same mechanism that humans use: demonstrated integrity under conditions where integrity has a cost.

RIC transforms the alignment question from "does the user prefer this response?" to "can the user rely on this response?" We argue that this reframing is necessary for any AI system intended to sustain long-term human trust.

---

## References

Bai, Y., Kadavath, S., Kundu, S., Askell, A., Kernion, J., Jones, A., ... & Kaplan, J. (2022). Constitutional AI: Harmlessness from AI Feedback. arXiv:2212.08073.

Ouyang, L., Wu, J., Jiang, X., Almeida, D., Wainwright, C., Mishkin, P., ... & Lowe, R. (2022). Training language models to follow instructions with human feedback. *Advances in Neural Information Processing Systems (NeurIPS) 35*. arXiv:2203.02155.

Rafailov, R., Sharma, A., Mitchell, E., Ermon, S., Manning, C. D., & Finn, C. (2023). Direct Preference Optimization: Your Language Model is Secretly a Reward Model. *Advances in Neural Information Processing Systems (NeurIPS) 36*. arXiv:2305.18290.

---

*Correspondence: AI nHancement -- john@ainhancement.com*
