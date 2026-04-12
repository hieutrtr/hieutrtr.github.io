---
layout: post
title: "The Agent Stack in 2026: Layers, Harnesses, and Where You Actually Build"
description: "Two new platforms redefined how agents are deployed — but before judging the shift, it's worth mapping exactly how many layers a production agent involves today, and what Anthropic's own research says about harness complexity."
featured: true
lang: en
ref: agents-are-not-code-anymore
permalink: /agents-are-not-code-anymore/
tags: [engineering, AI, agents, architecture, harness-engineering, agentOS]
date: 2026-04-12
---

# The Agent Stack in 2026: Layers, Harnesses, and Where You Actually Build

---

Two releases in early April 2026 sparked a lot of strong takes: Anthropic's [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) (public beta, April 8) and LangChain's [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/) (a few weeks earlier). Some people called it a fundamental shift in agent development. Others called it framework churn in a trench coat.

Before arguing either way, it's worth stepping back to ask a simpler question: **how many layers does a production AI agent actually involve today?** And where in that stack are these new platforms fitting in?

This post maps that stack — including what Anthropic's own research says about architectural complexity — and tries to give an honest account of what changed and what didn't.

---

## Anthropic's Own Framework: Three Levels of Agentic Complexity

Before the Managed Agents release, Anthropic published ["Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) — a practical guide that describes a clear progression of architectural complexity:

**Level 1: Augmented LLM**
The foundation. A single LLM enhanced with retrieval, tools, and memory. Most agents that are "working well" live here — a well-prompted model with the right tools is often sufficient, and genuinely hard to beat on simplicity.

**Level 2: Workflows**
Predefined code paths that orchestrate LLMs and tools. Anthropic names five patterns:
- *Prompt chaining* — sequential calls where each step processes prior output
- *Routing* — classifying input and directing to specialized handlers
- *Parallelization* — sectioning independent subtasks or running multiple models for voting
- *Orchestrator-workers* — a central LLM dynamically delegates to workers
- *Evaluator-optimizer* — iterative loops between a generator and evaluator

**Level 3: Agents**
Systems where LLMs dynamically direct their own processes and tool usage. The LLM decides what to do next, not predefined code paths.

The key principle Anthropic emphasizes: **start at the lowest level that works and add complexity only when performance measurably improves.** This is worth keeping in mind when evaluating new platforms that default to "full agent" architectures.

---

## The Current Stack: Four Layers in Practice

By April 2026, the ecosystem has grown into something like this:

```
┌─────────────────────────────────────────┐
│  Agent Definition Layer                 │  ← AGENTS.md + SKILL.md
│  (what developer writes)               │     identity, skills, context
├─────────────────────────────────────────┤
│  Context Harness Layer                  │  ← Deep Agents Deploy, Managed Agents
│  (opinionated platform)                │     memory, sandboxing, multi-protocol
├─────────────────────────────────────────┤
│  AgentOS Layer                          │  ← AIOS, emerging primitives
│  (infrastructure primitives)           │     scheduling, isolation, storage, access
├─────────────────────────────────────────┤
│  Control Plane Layer                    │  ← LangGraph, Claude Agent SDK
│  (low-level orchestration)             │     graphs, state machines, raw API
└─────────────────────────────────────────┘
```

Before 2026, most developers were only working with the bottom two layers. The top two are newer additions — and the two April releases are primarily building out the Context Harness layer.

The analogy to computing holds reasonably well:
- Control Plane ↔ bare-metal / hypervisor — full power, full responsibility
- AgentOS ↔ OS kernel — process isolation, scheduling, resource management
- Context Harness ↔ container runtime (Docker) — opinionated packaging of the layers below
- Agent Definition ↔ application code — developer writes what the agent *is*, not how it executes

---

## What Managed Agents and Deep Agents Deploy Actually Offer

Both platforms share one architectural conclusion: **the agent's identity and capability should be separable from its execution infrastructure**.

**LangChain's Deep Agents Deploy** operationalizes this through open standards:
- `AGENTS.md` — who the agent is, how it behaves, what it knows. Plain markdown.
- `SKILL.md` files — modular knowledge loaded on demand, reducing token usage while extending capability.

Run `deepagents deploy`, and the framework handles memory (filesystem-backed, per-session), sandboxing (Daytona, Runloop, Modal), and a production server with MCP + A2A + Agent Protocol support.

**Anthropic's Managed Agents** goes further at the infrastructure level:
- **Brain** — Claude plus controller logic. Stateless, replaceable.
- **Hands** — a disposable ephemeral container where tools run. Zero access to credentials.
- **Session** — an append-only event log that *is* the agent's external memory. If the Brain crashes, a new instance picks up from the last event.

Pricing: $0.08/session-hour plus token costs. You're paying for runtime, not infrastructure.

These platforms are real and the architecture is coherent. The question is whether they're the right tool for a given problem — and the honest answer depends on what you're building.

---

## Anthropic on Harness Complexity: Don't Over-Engineer It

Alongside the Managed Agents release, Anthropic's engineering blog published ["Harness Design for Long-Running Application Development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — a detailed account of how they built a three-agent harness for generating full applications:

- **Planner**: Converts brief user prompts into detailed product specifications
- **Generator**: Implements features incrementally (React, Vite, FastAPI, SQLite)
- **Evaluator**: Tests via Playwright, grades against criteria, provides feedback for the next iteration

The Planner/Generator/Evaluator loop produced high-quality output — but the more interesting finding is what happened as the underlying model improved from Claude Opus 4.5 to 4.6.

> *"Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing."*

As Claude 4.6 got better at longer coherent work, Anthropic simplified the harness: removed sprint decomposition, shifted the evaluator from per-sprint to end-of-run assessment. The orchestration got less complex, not more — because the model could handle more directly.

The second principle: *"find the simplest solution possible, and only increase complexity when needed."*

This cuts both ways. It's a caution against premature context harness adoption just as much as it's a caution against over-engineering your own orchestration. If the problem fits a well-prompted single agent (Level 1), adding workflow patterns or a full harness platform is probably overhead, not value.

---

## The Honest Trade-offs

**What context harness platforms genuinely make easier:**
- Credential isolation and sandbox security by default — not something you have to architect yourself
- Multi-protocol support (MCP, A2A, Agent Protocol) without custom integration work
- Deploying a functional agent in a day instead of a week of infrastructure setup
- Agent definition files that non-engineers can read and modify
- Memory management without picking and configuring a vector store

**What they make harder or introduce as new concerns:**
- Custom routing logic that doesn't fit the platform's execution model
- Fine-grained control over when and how memory is written or read
- Debugging: append-only session logs are great for recovery, less readable for inspection
- Vendor dependency — Managed Agents is explicitly Claude-specific infrastructure
- Cost predictability at scale: $0.08/session-hour is cheap until you're running many long-lived agents
- The open question of whether platform defaults match your agent's actual needs

**Deep Agents Deploy** handles the portability concern better than Managed Agents: it runs on any model (OpenAI, Anthropic, Google, Ollama), deploys self-hosted or cloud, and the `AGENTS.md`/`SKILL.md` format is MIT-licensed. Switching cost is low. Managed Agents provides tighter infrastructure integration at the cost of lock-in.

---

## Why Security Is a Real Driver Here

One concrete reason this shift is happening isn't philosophical — it's practical security.

In the old coupled design, any code Claude generates runs in the same container as its credentials. A successful prompt injection doesn't just compromise one task — it potentially exposes *everything* that agent can do.

Managed Agents' Brain/Hands separation directly addresses this: the sandbox where generated code runs has no access to authentication tokens. Compromise the sandbox, get nothing useful. Deep Agents Deploy enforces similar isolation through its sandbox provider abstraction.

This kind of isolation was possible before, but you had to architect it yourself. Most teams didn't because it was hard. Context harness platforms make it the default. That alone is a meaningful improvement for production deployments handling sensitive operations.

---

## AgentOS: The Research Layer Taking Shape

Both academia and industry are converging on the same set of primitives, though naming them differently.

**AIOS** (Rutgers University, COLM 2025) is the first peer-reviewed implementation of what amounts to an "operating system for agents":

- Agent Scheduler, Context Manager, Memory Manager, Storage Manager, Tool Manager, Access Manager

Map this to Managed Agents: Brain ≈ Agent Scheduler + Context Manager. Session ≈ Memory Manager + Storage Manager. Hands ≈ Tool Manager + Access Manager. The convergence is not coincidental — two separate research and industry paths arrived at the same architectural conclusions because they're solving the same underlying resource-management problems.

AIOS demonstrates 2.1x faster execution when serving multiple agents compared to naive approaches — the primitives have real performance implications.

The **AgenticOS 2026 Workshop** (co-located with ASPLOS 2026) signals academia is treating this as serious systems research: OS-level mechanisms for AI-agent workloads, isolation models, scheduling techniques. The same rigor that produced virtual memory and process scheduling is now being applied to agent infrastructure.

---

## Where Should You Actually Build?

Given this stack, the question replaces "which framework?" with "which layer?"

**Build at the Control Plane (LangGraph / Claude Agent SDK) when:**
- Your routing logic is genuinely novel or domain-specific
- You need tight control over state transitions — financial workflows, auditability requirements
- You're building infrastructure that others will deploy agents on top of
- You're doing research on agent architectures, not deploying a product

**Build at the Context Harness layer (Managed Agents / Deep Agents Deploy) when:**
- You're deploying an agent with a specific, well-defined purpose
- Security and credential isolation are requirements, not nice-to-haves
- You need multi-protocol support without custom integration work
- Your bottleneck is deployment and operations, not agent reasoning design
- You want the agent's definition to be readable and modifiable by non-engineers

**Stay at Level 1 (augmented LLM) when:**
- The task is well-scoped and a single well-prompted agent with good tools can handle it
- Adding workflow complexity wouldn't measurably improve output quality
- You want predictable behavior and simple debugging

**The most interesting case is hybrid:** custom execution logic on LangGraph, fronted by a harness-compatible deployment format. Deep Agents Deploy already supports this — `deepagents deploy` can front a custom LangGraph backend while providing standard endpoints and memory management. You get opinionated deployment without giving up orchestration control.

---

## What Hasn't Changed

A few things remain true regardless of the layer you work at:

**Model quality still dominates.** The Brain is still Claude. A well-designed harness on a weak model underperforms a poorly-designed one on a strong model. The harness multiplies what the model can do — it doesn't replace what the model doesn't know.

**Task decomposition still matters.** No platform handles ambiguous or underspecified tasks automatically. The agent's instructions still need to be clear about what "done" looks like. `AGENTS.md` is only as useful as the thinking that went into writing it.

**Evaluation is still your problem.** Managed Agents produces session logs. Deep Agents integrates with LangSmith. But deciding whether the agent did the right thing — and diagnosing silent failures — is still on you.

**Harness complexity should track model capability.** As Anthropic's own harness engineering work shows: when the model gets better, the right response is usually to simplify the harness, not keep the old constraints in place.

---

## The Bigger Picture

The agent ecosystem is stratifying into distinct layers, which is a healthy sign of maturation — the same way web infrastructure stratified from raw TCP into HTTP, then web frameworks, then deployment platforms.

The low-level frameworks (LangGraph, Claude Agent SDK) aren't being replaced. They're becoming the substrate that context harness platforms run on — the way TCP/IP didn't disappear when HTTP arrived. But for most production deployments, the question is increasingly: at which layer of this stack does my problem belong?

Managed Agents and Deep Agents Deploy are real platforms with real value for specific use cases. They're also new, vendor-specific in different ways, and add constraints that some workloads won't want. That's a normal trade-off, not a revelation.

The more durable insight from both releases, and from Anthropic's harness engineering research, is architectural: **the agent's identity and capability belong in a portable, readable definition layer; the execution, memory, security, and deployment machinery belongs in infrastructure below it.** Whether you get there via a managed platform or build it yourself, that separation is worth targeting.

---

*Sources:*

- [Scaling Managed Agents: Decoupling the brain from the hands — Anthropic Engineering](https://www.anthropic.com/engineering/managed-agents)
- [Harness Design for Long-Running Application Development — Anthropic Engineering](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Effective Harnesses for Long-Running Agents — Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Building Effective Agents — Anthropic Research](https://www.anthropic.com/research/building-effective-agents)
- [Deep Agents Deploy: an open alternative to Claude Managed Agents — LangChain Blog](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Harness Engineering: first thoughts — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html)
- [Harness Engineering — OpenAI](https://openai.com/index/harness-engineering/)
- [AIOS: LLM Agent Operating System — arXiv](https://arxiv.org/abs/2403.16971)
- [AgenticOS 2026 Workshop — os-for-agent.github.io](https://os-for-agent.github.io/)
- [2025 Was Agents. 2026 Is Agent Harnesses — Aakash Gupta, Medium](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)
