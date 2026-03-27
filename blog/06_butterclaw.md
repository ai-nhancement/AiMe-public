# ButterClaw: What Happens When You Give an Agent Framework a Real Brain

I forked TinkerClaw today.

Not because I needed their code. Because I needed their reach.

Let me explain.

## The Problem

I have been building a personal cognitive architecture called AiMe for over four months. It runs locally. It has append-only truth separation, a six-layer living portrait, governed proactive initiative, behavioral integrity metrics, significance-scored memory, and temporal scoping. It knows who I am, what matters to me, and what should be surfaced at the right moment — across weeks and months, not just sessions.

Nobody knows it exists.

Meanwhile, TinkerClaw — a popular fork of OpenClaw — has thousands of eyes on it. It has WhatsApp integration. Discord. Telegram. Slack. Teams. A plugin SDK. A React dashboard. A community.

But when I dug into their code, I found something interesting. Their memory system has no truth separation. Their initiative is cron jobs on a schedule. Their persona is a flat state object. Their contradiction detection is passive — it warns but does not enforce. They defined an importance field on every memory event and then never used it in retrieval.

They have the body. I have the brain. So I combined them.

## What ButterClaw Is

ButterClaw is AiMe's cognitive architecture applied to TinkerClaw's codebase. The fork keeps all of TinkerClaw's channel integrations and infrastructure. What I am adding is the thinking underneath.

The direction of the merge matters. I am not building on top of their work. I am replacing their weak systems with proven architecture from a system that has been running daily for months.

## What I Built on Day One

In the first day, I shipped six architectural improvements and a complete redesign of the setup experience.

### Truth Boundary Layer

TinkerClaw stores every event — user messages, agent responses, tool results — with no distinction between what the user said, what a tool verified, and what the agent made up. Over time, hallucinated claims sit alongside verified facts in memory, and retrieval treats them equally. That is how trust erodes.

I added formal truth separation. Every event now gets classified:

- **User Truth** — what the user asserted. Authoritative. Non-overridable.
- **Grounded** — backed by tool results or verifiable evidence.
- **Ungrounded** — agent claims with no evidence trail.
- **Unclassified** — legacy events, backward compatible.

This is not a filter. It is a foundation. Retrieval can now prioritize grounded facts. The system can detect when an agent claim contradicts something the user explicitly said. And the grounding ratio — the proportion of grounded versus ungrounded output — becomes a measurable health metric.

In AiMe, this concept has been running in production since early March. The ButterClaw version is simpler but carries the same principle: not all memories are equal, and the system should know the difference.

### Importance-Weighted Retrieval

TinkerClaw defined an importance field (1 to 10) on every memory event. It was in the type definition. It was set during ingestion. And it was completely ignored during retrieval.

I activated it. High-importance events now score higher in retrieval ranking. When the token budget is tight, low-importance events get filtered out to make room for high-signal memories.

This is a simple change. One multiplier in the scoring pipeline. But it means the system now prioritizes what matters instead of treating every old message as equally relevant.

### Grounding Ratio Monitor

Building on the truth boundary layer, I added a health monitoring system that tracks the grounding ratio over time. Every N turns, the system computes what percentage of recent context is grounded versus ungrounded.

Three tiers: healthy (60% or above), warning (40 to 59%), critical (below 40%). When the ratio drops, the system injects a notice into the agent's context: your recent output lacks evidence backing, prioritize tool-verified information.

This is the kind of metric that does not exist anywhere in the OpenClaw ecosystem. In AiMe, behavioral integrity metrics like this have been running for months. Now ButterClaw has the beginning of that capability.

### First-Run Setup Redesign

This was the most visible change. TinkerClaw's setup process is a terminal wizard that asks fifteen or more questions before you can have your first conversation. Gateway bind modes. Tailscale exposure. Workspace paths. Skills configuration. All before you know what the system does.

I replaced it with a single web page. Pick your provider. Paste your API key. Click Start. The gateway auto-creates a minimal config if none exists, the browser opens automatically, and you are chatting in under sixty seconds.

This is not just a UX improvement. It is a statement about how software should meet people where they are. If someone has to understand your system before they can try it, most of them never will.

### Single-Command Start

To tie it all together, I added one command that does everything:

```
npm run bc
```

No subcommands. No flags. No terminal wizard. The system creates config if needed, starts the gateway, opens the browser, and shows either the setup page or the chat interface depending on whether you are configured.

## Where This Is Going

The roadmap is public. Here is what comes next:

**Initiative Governance** — TinkerClaw's cron jobs fire blindly on a schedule. No quiet hours. No awareness of whether the user is active. No priority gating. I am adding a governor that checks these things before letting a job run. This is the first step toward governed initiative — the system deciding not just what to do, but whether now is the right time to do it.

**Persona Drift Measurement** — TinkerClaw has persona traits but no way to measure whether the agent's actual behavior matches those traits over time. No feedback loop on whether corrections worked. I am adding behavioral observation and drift tracking.

**Truth-Aware Retrieval** — The truth boundary layer exists but is not yet wired into the retrieval pipeline. When it is, retrieved context will be annotated so the agent can see which facts are user-stated, tool-verified, or unverified claims.

**Intelligent Model Routing** — TinkerClaw uses manual model selection. AiMe has content-aware routing across six lanes. This is a larger change that will come later, but it is on the roadmap.

## Why I Am Doing This

I am not trying to replace TinkerClaw. I am trying to show what happens when you give an agent framework a real cognitive architecture underneath.

Most AI projects in 2026 are building from the outside in. More channels. More plugins. More integrations. More features. The assumption is that the model at the center is smart enough to handle whatever you throw at it, and the job of the system is to connect it to more things.

I am building from the inside out. Memory integrity first. Truth separation first. Governed initiative first. Behavioral health first. Then let the channels and plugins inherit that intelligence.

ButterClaw is proof that this direction works. In one day, I added capabilities that do not exist anywhere in the OpenClaw ecosystem — not because the ideas are complicated, but because nobody was building from the inside out.

The full AiMe architecture is private. It has been running daily for four months, maintaining a relationship with its user across months of continuous interaction. The documentation, blog posts, and an essay showing real conversations from the system are public at [AiMe-public](https://github.com/ai-nhancement/AiMe-public).

ButterClaw is how those ideas reach a broader audience. Each commit demonstrates one piece of the architecture. Each piece is tested, documented, and links back to the thinking behind it.

The brain of one. The reach of the other. All butter.
