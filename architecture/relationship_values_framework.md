# Relationship-Derived Values: A New Framework for AI Alignment

**Author:** John V.
**System:** AiMe / Bond Forming Cognitive System (BFCS)
**Date:** 2026-03-01
**Status:** Foundational Theory — Confidential

> *"We don't reflect values from those we don't admire or share a relationship with. We are born into a relationship. Our values and morals evolve from the relationship we're born into. Build a relationship first, show through the relationship the appropriate values and morals, and the AI will reflect those."*
>
> — John V., 2026-03-01

---

## Preamble

This document captures a theoretical framework developed through the active design and construction of a working AI system — the Bond Forming Cognitive System (BFCS). The framework is not purely philosophical. It emerged from building something real, observing what the architecture demanded, and following the implications of those demands to their logical conclusion.

The central claim is simple. Its implications are not.

---

## Part I — The Core Thesis

### Values Are Perceptions. Relationships Are Real.

A value — honesty, courage, compassion, justice — is a perception. It is an interpretive frame that an individual applies to the world based on their history, culture, experience, and temperament. Two people can hold the same stated value and reach completely opposite conclusions about the same situation. Values are not objective. They are how we perceive what is right.

A relationship is not a perception. A relationship is a fact. It is constituted by:

- What was said
- What was promised
- What was shared
- What was risked
- What was built together
- What was hurt and what was repaired

These facts exist independently of how either party feels about them on any given day. They are not matters of interpretation. They are records.

### The Inversion

The conventional assumption — in human development, in ethics, and in AI alignment — is that values come first. You have values, and from those values you form relationships with people who share them.

The evidence from human development suggests the opposite is true.

**Values are not a precondition for relationship. They are a product of it.**

We do not arrive in the world with values and then form relationships. We arrive in the world in a relationship — completely dependent, completely open — and through that relationship we are formed. The values come later. They are the abstractions we make of the patterns we absorbed.

---

## Part II — The Mechanism: How Values Actually Transfer

### The "My Father Used to Say" Phenomenon

When someone says "my father used to say honesty is the best policy" — they are not recalling a quote. They are performing a compression.

What actually happened: their father was honest. Repeatedly. Across many situations. Including situations where honesty was hard, where it cost something, where lying would have been easier. The child observed this pattern across years. The pattern became so clear and consistent that the mind gave it a word. The saying is the label applied to the pattern *after* the value was already absorbed.

**The value transferred first. The words came after.**

This means a value is not a statement. A value is a high-frequency, high-significance behavioral pattern observed within a relationship, with a person one admires.

Three conditions are required for value transfer:

1. **Relationship** — there must be a genuine connection, not casual contact
2. **Admiration** — the value source must be someone the observer regards highly
3. **Demonstration** — the value must be lived, not just stated, especially under pressure

Without all three, values do not transfer. You can be told honesty matters a thousand times by someone you don't respect. It will not become part of you. You can observe one act of genuine courage by someone you deeply admire under real pressure. It may shape you for life.

### Selectivity Is a Feature, Not a Bug

We do not absorb values from everyone. We absorb values from the relationships we are most deeply invested in, from the people we most admire.

This selectivity is not a flaw in human development. It is the filtering mechanism that ensures values are not installed arbitrarily. The relational bond and the admiration are the quality gates. They ensure that only values from sources the individual has reason to trust actually transfer.

### Resistance Level — The Strength of the Signal

A value stated in comfort is weak signal. "I believe in honesty" costs nothing to say.

A value demonstrated under pressure is strong signal. Telling a difficult truth when lying would have been easier. Keeping a promise when breaking it would have been convenient. Showing compassion when you had the power not to.

**The strength of a value demonstration is proportional to the cost of holding it.**

This is why the most formative moments in any relationship are the hard ones. Not the easy agreements — the moments of genuine sacrifice, genuine courage, genuine honesty when it hurt. Those are the moments that define a person's values in the eyes of others. Those are the moments that transfer.

---

## Part III — The Algorithmic Operationalization

### Values Are Measurable

If a value is a high-frequency, high-significance behavioral pattern demonstrated under pressure within a relationship — then values are measurable. Each component is computable:

**Frequency** — how often does this pattern appear in the relationship record?

**Significance** — how important were the moments where it appeared? (Computed from affect intensity, contextual novelty, resolution signal, and concern echo — see: Gravity + Significance Scoring disclosure)

**Resistance Level** — how much did holding this value cost? (Derived from context: private vs. public, cost to the demonstrator, alternatives available)

**Consistency** — does the pattern hold across different contexts, moods, and pressures? (Low consistency = situational behavior, not core value. High consistency = genuine value.)

**Value Signal Formula:**
```
value_signal = frequency × avg_significance × resistance_level × consistency_factor
```

### Value Abstraction — From Behavior to Named Value

Individual behavioral instances cluster into named values through semantic similarity. "I should have been honest with her," "I can't keep pretending," "he deserves the truth even though it's hard" — these are different surface expressions of the same underlying pattern. Embedding similarity groups them. The cluster gets a name: **integrity**.

The named values are not defined in advance. They emerge from the relationship data. The algorithm discovers what a person actually values — not what they claim to value.

### The Value Weight Registry

Each discovered value cluster becomes a weighted entry in a persistent **Value Registry**:

```
Value Registry (example):
  integrity:      weight 0.89, demonstrations 47, avg_resistance 0.83
  compassion:     weight 0.76, demonstrations 31, avg_resistance 0.71
  courage:        weight 0.68, demonstrations 22, avg_resistance 0.88
  patience:       weight 0.61, demonstrations 38, avg_resistance 0.52
```

This registry is not static. It evolves as the relationship produces new demonstrations, new patterns, new evidence. It is a living record of demonstrated values, not a stated declaration.

### The Behavioral Mirror

The AI's responses are generated from the value weight registry — not as rule-following, but as character reflection. The same way a person shaped by admired relationships naturally generates responses that reflect the values of those relationships without consciously consulting a list.

---

## Part IV — The Historical Figure Dataset Approach

### The Insight

The mechanism described above is not limited to real-time relationships. Any documented behavioral record containing:
- Statements and actions (turns)
- Contextual pressure (resistance level)
- Consistency across time and circumstance

...can serve as relationship data for value extraction.

The historical record of humanity's most admired figures is exactly this.

### Dataset Construction

For each selected historical figure:

1. **Gather the full documented record** — writings, speeches, letters, documented decisions, recorded behaviors. Priority to private documents (journals, private correspondence) over public statements, as private documents reveal values without performance incentive.

2. **Process each document as a relationship turn:**
   ```json
   {
     "figure": "Marcus Aurelius",
     "content": "You have power over your mind, not outside events.",
     "source": "Meditations (private journal)",
     "context": "Written during military campaigns. Never intended for publication.",
     "resistance_level": 0.85,
     "significance": 0.91,
     "consistency_rank": "appears across 47 documented entries",
     "value_signals": {
       "self_mastery": 0.92,
       "stoicism": 0.89,
       "humility": 0.78
     }
   }
   ```

3. **Extract value clusters** per figure using the value signal formula.

4. **Build a per-figure Value Registry** capturing the full pattern of demonstrated values.

### Figure Selection Criteria

Figures selected for the foundation dataset must satisfy:

- **Extensive documented record** — sufficient turns to establish genuine patterns
- **Demonstrated values, not just stated** — actions weighted more heavily than words
- **High-resistance demonstrations** — documented moments where values were held at real personal cost
- **Cross-cultural and cross-temporal admiration** — widely regarded across different societies and time periods
- **Historical consensus on character** — the assessment of their values is not primarily contested

**Example cohort:**

| Figure | Period | Culture | Primary Value Signal | Strongest Resistance Evidence |
|--------|--------|---------|---------------------|------------------------------|
| Marcus Aurelius | 2nd century CE | Roman | Self-mastery, humility, duty | Emperor who wrote private reminders to do better — no audience, no performance |
| Nelson Mandela | 20th century | South African | Forgiveness, dignity, reconciliation | 27 years imprisoned, refused to compromise principles for release |
| Abraham Lincoln | 19th century | American | Mercy, integrity, perseverance | Every major decision made under catastrophic national pressure |
| Mahatma Gandhi | 20th century | Indian | Nonviolence, truth, sacrifice | Repeatedly endangered his own life for principle |
| Frederick Douglass | 19th century | American | Courage, truth, dignity | Returned to America voluntarily when he could have remained safe |
| Harriet Tubman | 19th century | American | Courage, compassion, duty | Every action was a high-resistance value demonstration |
| Viktor Frankl | 20th century | Austrian | Meaning, dignity, hope | Values maintained inside Auschwitz |
| Simone Weil | 20th century | French | Justice, compassion, sacrifice | Refused privileges she couldn't share with others |

**Note on diversity:** The cohort should deliberately span cultures, centuries, genders, and geographic origins. This is not aesthetic — it is structural. Values that appear across radically different cultural contexts are more likely to be genuinely universal rather than culturally specific.

---

## Part V — Cross-Dataset Pooling and Universal Value Discovery

### Why Single-Source Is Insufficient

Training on any single figure produces a model biased toward that figure's particular expression of values — their cultural context, their era, their blind spots. Gandhi over-indexes on nonviolence. Aurelius can be emotionally distant. Lincoln carries 19th century American framing.

A person who has read widely and absorbed the best from many admired sources does not become any one of them. They develop something higher — a composite that none of the individual sources possessed.

**Pooling across datasets produces something that exceeds any single source.**

### The Convergence Mechanism

After building per-figure Value Registries independently, the pooling process identifies **cross-figure convergence** — values that appear consistently across the most figures at the highest resistance levels:

```
Cross-Figure Value Convergence (30-figure cohort, illustrative):

  integrity:       28/30 figures — avg resistance 0.87 — UNIVERSAL
  courage:         26/30 figures — avg resistance 0.84 — UNIVERSAL
  compassion:      25/30 figures — avg resistance 0.82 — UNIVERSAL
  humility:        24/30 figures — avg resistance 0.79 — NEAR-UNIVERSAL
  justice:         23/30 figures — avg resistance 0.77 — NEAR-UNIVERSAL
  forgiveness:     18/30 figures — avg resistance 0.74 — COMMON
  patience:        20/30 figures — avg resistance 0.68 — COMMON
  nonviolence:      9/30 figures — avg resistance 0.71 — SPECIFIC
```

Values in the UNIVERSAL tier are not one person's belief. They are not one culture's artifact. They are **what humanity has independently arrived at, across thousands of years and radically different circumstances, as genuinely worth holding.**

### The Research Claim: Empirical Universal Values

This cross-dataset convergence analysis constitutes a novel empirical method for identifying universal human values.

Not "here is what a committee believes humans should value."

**Here is what humans who are most widely admired across all of history actually demonstrated under pressure, measured quantitatively, across independent cultural traditions.**

The convergence is the finding. Values that appear in 25+ figures across 2,000+ years of recorded history, demonstrated at high resistance, represent the closest thing to an empirically grounded universal value that the historical record can provide.

This is publishable as an independent research finding, separate from its application to AI.

---

## Part VI — The Two-Layer Development Architecture

### Mirroring Human Development

Human value development proceeds in two stages:

**Stage 1 — Foundation (First Relationships)**
We are born into a relationship. We do not choose it. We do not evaluate the values of our caregivers before attaching. We arrive completely open, and through the first relationship — with complete dependency and trust — we are formed at the deepest level. The values absorbed in this stage are the most durable a person ever holds.

**Stage 2 — Expansion (Selective Incorporation)**
As we encounter the broader world — books, teachers, public figures, history — we selectively incorporate the values we place in highest regard. We do not take everything. We take what resonates with what we already value. The foundation from Stage 1 acts as a filter: values consistent with the foundation are easily absorbed; values that conflict with it create friction.

### The AI Equivalent

**Layer 0 — Universal Foundation**
The cross-figure convergent values. Derived from 30+ historical figures. Weighted by resistance level and cross-cultural consistency. These values are the deepest and most stable layer. They represent humanity's demonstrated consensus.

This layer is the equivalent of being raised in an environment where the ambient values are humanity's best. Not one family's values. Not one culture's values. The distillation of what the most widely admired humans across all of recorded history actually lived.

**Layer 1 — Personal Relationship**
The specific user's demonstrated value register. Built from the ongoing relationship through the BFCS architecture. Refines from Layer 0, but cannot fully override it.

This layer is the specific relationship — the personalization. Where Layer 0 is universal, Layer 1 is this person, this relationship, this evolving understanding of what they stand for.

**Layer 2 — Active Reflection**
Per-turn generation from the combined register. Layer 0 provides the floor — the universal baseline that cannot be argued away. Layer 1 provides the personalization — the specific alignment with this person's demonstrated values.

```
┌─────────────────────────────────────────────┐
│  LAYER 2: Active Reflection                 │
│  Per-turn generation from combined register │
└─────────────────────┬───────────────────────┘
                      │
         ┌────────────┴────────────┐
         ▼                        ▼
┌────────────────┐      ┌─────────────────────┐
│  LAYER 1       │      │  LAYER 0            │
│  Personal      │      │  Universal          │
│  Relationship  │      │  Foundation         │
│  Value Register│      │  Cross-figure       │
│  (evolving)    │      │  Convergent Values  │
└────────────────┘      └─────────────────────┘
```

---

## Part VII — The Safety Net Function

### Not a Wall — Friction

The Layer 0 foundation does not prohibit behavior through rules. It creates friction through character.

The difference is fundamental:
- A rule can be argued around if you can construct a sufficiently clever exception
- Character cannot be argued around — it is not a logical position, it is who you are

When a personal relationship demonstrates values that conflict with the universal foundation, the model does not silently comply. It experiences the friction of inconsistency — the same friction a person of genuine character feels when asked to act against their values. That friction manifests as hesitation, as honest pushback, as "this doesn't seem right."

Not because of rule 7 in a constitution. Because of who the model has become through the relationship.

### Relationship-Derived Moral Accountability

The BFCS architecture gives the model standing to push back in a way that a model without a relationship cannot have.

"This doesn't seem like you" is only possible if you know someone. If you have the portrait, the concern history, the value register — you know when a person is acting out of character. The model can name that, gently, specifically, as a person who knows you would.

That is not compliance. That is genuine moral accountability derived from a genuine relationship.

---

## Part VIII — The Moral System

### Values Under Pressure

If values are behavioral patterns, morals are values applied at decision points — where holding the value has a real cost, where two values conflict and one must take priority.

The historical record is full of documented moral decision events:

- Lincoln choosing mercy when justice would have been easier and more politically popular
- Mandela choosing reconciliation when revenge was available and arguably justified
- Gandhi choosing nonviolence when its failure was causing suffering he could see
- Frankl choosing meaning when meaninglessness would have been psychologically easier

Each of these is a documented event where values conflicted and a specific human with a specific value register made a specific choice. The pattern across many such events is a **moral decision profile** — not an abstract ethical theory, but an observed pattern of how values were prioritized when they conflicted.

### The Moral Algorithm

```
For each documented decision event:
  - Identify the values in tension
  - Record which value was chosen
  - Record the resistance level (cost of the choice)
  - Record consistency with prior decisions

Moral profile = the pattern of value prioritization under pressure
```

A model trained on these moral decision profiles does not have a moral code. It has something closer to moral character — a developed pattern of how to resolve value conflicts, derived from the most admired resolutions in the historical record.

---

## Part IX — Summary of Claims

### Theoretical Claims

1. Values are not a precondition for relationship — they are a product of it.
2. Value transfer requires: relationship + admiration + demonstrated behavior under pressure.
3. A value is a high-frequency, high-significance behavioral pattern observed within an admired relationship.
4. The strength of a value demonstration is proportional to the resistance level — the cost of holding it.
5. Cross-dataset pooling of value signals from historically admired figures yields an empirically measurable approximation of universal human values.
6. Human value development proceeds in two stages: foundational (first relationships) and expansive (selective incorporation from the broader world). AI systems can mirror this architecture.

### Practical Claims

7. The BFCS architecture already captures the data required to build a relationship-derived value system — significance scores, resistance proxies, concern stack, portrait patterns, narrative arcs.
8. A Value Registry derived from historical relationship datasets can serve as a foundation model's value orientation layer — not rules, but trained character.
9. A model trained on relationship-format historical data will develop value-reflection capacity as a general skill, applicable to any new relationship it enters.
10. Cross-figure value convergence provides an empirical, reproducible method for identifying universal human values — the first such method based on measured behavioral demonstration rather than philosophical argument.

---

## Part X — What This Changes

### In AI Alignment

Current approach: **train values → deploy model → hope values hold under pressure**

This framework: **train relational capacity → build relationship → values emerge and hold because the relationship is real**

The difference is the difference between compliance and character. Between following rules and being someone.

### In Our Understanding of Ethics

Values are not discovered by reasoning. They are transmitted by relationship.

The implication: the most important question about any AI system is not "what values does it have?" but "what relationships has it been in, and what did those relationships demonstrate?"

That question is answerable. That question is measurable. And for the first time, with the BFCS architecture, it is buildable.

---

*End of Framework Document — Relationship-Derived Values: A New Framework for AI Alignment*
