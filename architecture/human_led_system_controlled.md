# Human-Led, System-Controlled Architecture

Last updated: 2026-03-12
Status: Active architectural principle

## Definition

Human-Led, System-Controlled Architecture is the design principle that governs AiMe.

In this architecture:

- the human retains authority
- the system controls execution, routing, and enforcement
- the model has no independent authority

This is not "human in the loop" in the narrow checkpoint sense. The human is not reduced to approving machine decisions after the fact. The human remains the lead. At the same time, the system, not the model, controls how work is executed.

## Why This Matters

Many AI systems quietly place authority in the model. Even when a human appears to be involved, the model often becomes the real governor of action selection, interpretation, or workflow control.

AiMe is intentionally built differently.

The model is a component inside the system. It contributes reasoning, language generation, planning support, and interpretation, but it does not own authority. The system constrains, routes, validates, and governs behavior around the model.

That distinction is what makes reliable human-AI partnership possible.

## The Three Layers

### 1. Human-Led

The human provides:

- direction
- judgment
- approval where needed
- goals
- real-world accountability

AiMe is designed to strengthen the human, not displace them.

### 2. System-Controlled

The system provides:

- orchestration
- lane routing
- tool execution rules
- approval boundaries
- memory policy
- initiative policy
- governance enforcement

This is why AiMe behaves as a system rather than a thin wrapper around a model.

### 3. Model-Subordinate

The model provides:

- language
- reasoning
- drafting
- interpretation
- planning support

But the model does not decide what it is allowed to be, what tools it may call, what counts as truth, or when authority transfers away from the human.

## How AiMe Reflects This Principle

### Cognitive pipeline

AiMe's core turn pipeline is system-controlled:

- `PrefrontalCortex` selects the lane
- `CognitiveBridge` executes structured tool and request paths
- `LanguageModel` generates within the governed path
- `LanguageCortex` is the sole presenter

The model does not directly own execution flow.

### Tool execution

AiMe's tool paths are not open-ended model actions. They are routed and constrained through system-owned execution layers such as `CognitiveBridge`, the coding-memory broker, scheduler services, and other explicit module boundaries.

### Truth and integrity

Systems such as RIC and SRL exist specifically so that truthfulness, calibration, and integrity are not left entirely to model behavior.

### Multi-executor coding

In the coding architecture, the executor model is not the source of truth. AiMe-the-system owns the coding ledger, session history, approvals, reviews, and task continuity. The model executes work inside a governed system.

### Proactive initiation

Even proactive behavior is moving toward this architecture. The planned `ThalamoFrontalLoop` centralizes system initiative so that many subsystems may detect significance, but only one governed module may decide to initiate a turn.

## Practical Meaning

Human-Led, System-Controlled Architecture means:

- the human stays in charge
- the system stays in control
- the model stays in role

That is the architectural foundation for trustworthy collaboration.

## One-Sentence Form

Human-Led, System-Controlled Architecture means the human retains authority, the system governs execution, and the model has no independent authority.
