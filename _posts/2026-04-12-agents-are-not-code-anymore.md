---
layout: post
title: "Agents Are Not Code Anymore — The Rise of Context Harness Engineering"
description: "Deep Agents Deploy and Anthropic Managed Agents show a fundamental shift: building agents is no longer about code. It's about defining identity, skills, and context."
featured: true
lang: en
ref: agents-are-not-code-anymore
permalink: /agents-are-not-code-anymore/
tags: [engineering, AI, agents, architecture, harness-engineering, agentOS]
date: 2026-04-12
---

# Agents Are Not Code Anymore — The Rise of Context Harness Engineering

> *The most important part of an agent is no longer the code you write. It's the context you define.*

---

Two releases dropped in early April 2026 that looked like shipping announcements but were actually something else: a signal that the definition of "AI agent" is quietly undergoing a foundational change.

On April 8, Anthropic launched [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) in public beta. A few weeks before that, LangChain released [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/), explicitly positioned as an open-source alternative. These two systems are built by different teams with different philosophies — yet they converge on nearly identical architectural conclusions. That's worth paying attention to.

---

## The Old Model: Agents Are Code

Until recently, "building an agent" meant writing a lot of code. Not just a system prompt — actual application logic.

You'd define tools as decorated Python functions. Wire up memory backends (Redis, Postgres, or a vector store). Write routing logic: when to call a tool, when to respond, when to hand off. Manage state across turns. Handle retries. And yes, also write the system prompt.

The agent's identity — its instructions, capabilities, constraints — lived scattered across function definitions, prompt strings, and config dictionaries. Tightly coupled to the framework. A LangGraph agent looked fundamentally different from a CrewAI agent or a raw Claude API agent. Migrating meant rewriting most of the code.

This wasn't wrong. It was just the only way we knew.

---

## The New Model: Agents Are Context

Both Managed Agents and Deep Agents Deploy flip this around. The new primitive isn't code — it's **context**.

**LangChain's Deep Agents** centers on two open standards:

- **`AGENTS.md`** — the agent's identity: who it is, how it behaves, what it knows. Plain markdown.
- **Skills** (`SKILL.md` files) — specialized knowledge modules loaded on demand, reducing token usage while extending capability.

Run `deepagents deploy`, and the framework handles memory (filesystem-backed, per-session), sandboxing (Daytona, Runloop, Modal), and a production server with MCP, A2A, and Agent Protocol support — all from those definition files.

**Anthropic's Managed Agents** goes further at the infrastructure level:

- **Brain** — Claude plus controller logic. Stateless, replaceable.
- **Hands** — a disposable ephemeral container where tools run. Zero access to credentials.
- **Session** — an append-only event log that *is* the agent's external memory. If the Brain crashes, a new instance picks up from the last event.

The pricing ($0.08/session-hour + token costs) reflects the model: you're paying for runtime, not infrastructure you manage yourself.

Both systems share one core insight: **the agent's identity and capability are separable from its execution infrastructure**.

---

## Context Harness: Identity + Skills + Context

"Context harness" is the right name for what these platforms provide — and it's worth defining precisely.

An agent built on a context harness is defined by three things:

1. **Identity**: who it is, how it thinks, its constraints — `AGENTS.md`
2. **Skills**: what specialized knowledge it can load on demand — `SKILL.md` files
3. **Session**: what it remembers about this conversation — managed by the platform

Write those three things well, and the platform turns them into a production deployment. Execution, memory, security, multi-protocol interoperability — all handled at the infrastructure layer.

The interesting shift this causes: the most important work moves from *"how do I implement this behavior in code"* to *"how do I express this agent's identity and capabilities precisely."* That's closer to technical writing and context engineering than software architecture.

---

## Why It Happened: Security Was the Forcing Function

One reason this shift is happening isn't philosophical — it's practical security.

In a coupled design (the old model), any code Claude generates runs in the same container as its credentials. A successful prompt injection doesn't just compromise one task — it gives an attacker access to *everything* the agent can do.

Managed Agents' Brain/Hands separation breaks this: the sandbox where generated code runs has no access to authentication tokens. Compromising the sandbox yields nothing useful. Deep Agents Deploy enforces similar isolation through its sandbox provider abstraction.

This kind of security was possible before — but you had to architect it yourself, and most teams didn't because it was hard. Context harness platforms make it the default.

---

## Harness Engineering: The Discipline Behind It

While the industry was calling this "context harness," two important papers arrived giving it a more rigorous name.

In early 2026, OpenAI published a memo about how they built Codex — a coding agent that produced over 1 million lines of code without a single human-written line. The surprising finding wasn't model capability. It was that **the decisive factor was the system surrounding the model, not the model itself.** They called that system the **harness**.

Formula: `Agent = Model + Harness`

The harness is everything outside the model: the tools it can call, the architectural constraints that prevent it from breaking invariants, the feedback loops that enable self-correction, the observability layer for engineers to monitor it. This is why LangChain jumped from 52.8% to 66.5% on Terminal Bench 2.0 — moving from top 30 to top 5 — without changing the model. Better harness.

Martin Fowler formalized this in April 2026, defining a harness as three components:

- **Context engineering** — what does the agent know, when, and in what structure?
- **Architectural constraints** — what *can't* the agent do? Guardrails, dependency rules, boundaries.
- **Garbage collection** — what happens after? State cleanup, cost accounting, session lifecycle.

**Harness engineering** is the discipline of designing these three components correctly across an entire agent system.

The key distinction: context harness is the *platform layer* — what developers deploy onto. Harness engineering is the *discipline* — how to design it right. In other words: harness engineering is the skill. Context harness is the output.

---

## AgentOS: The Research Name for the Same Thing

While the engineering community was saying "harness," the research community was saying **AgentOS** — and building it seriously.

**AIOS** (Rutgers University, COLM 2025) is the first peer-reviewed implementation. Its architecture names exactly what production platforms are doing implicitly:

- **Agent Scheduler**: prioritize and schedule agent requests to optimize LLM utilization
- **Context Manager**: snapshot and restore generation state, manage context windows
- **Memory Manager**: per-session short-term memory
- **Storage Manager**: persistent long-term storage
- **Tool Manager**: tool execution management
- **Access Manager**: resource access control

Look at Managed Agents again: Brain ≈ Agent Scheduler + Context Manager. Session ≈ Memory Manager + Storage Manager. Hands ≈ Tool Manager + Access Manager.

This isn't a coincidence. Two separate communities (research and industry) solved the same set of problems and arrived at the same architectural conclusions. AIOS demonstrates 2.1x faster execution when serving multiple agents compared to naive approaches — proving these primitives have real performance implications, not just conceptual elegance.

The **AgenticOS 2026 Workshop** (co-located with ASPLOS 2026) signals academia is taking this seriously: they're soliciting papers on OS-level mechanisms for AI-agent workloads, isolation models, scheduling techniques, and observability. Same level of systems research that produced virtual memory and process scheduling.

---

## The Stack: Three Layers, One Picture

Put it together and you have a coherent layered stack:

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

**Harness Engineering** is the discipline that runs through all of it — how to design each layer and the interfaces between them.

The analogy to familiar computing:

- AgentOS layer ↔ OS kernel (process isolation, scheduling, resource management)
- Context Harness layer ↔ container runtime (Docker) — opinionated packaging of OS primitives
- Agent Definition layer ↔ application — developer writes logic, not syscalls
- Control Plane layer ↔ bare-metal hypervisor — full power, full responsibility

Deep Agents Deploy is essentially Docker for the agent world. It doesn't provide the primitives (that's AgentOS), and it doesn't contain business logic (that's AGENTS.md). It's the packaging layer — takes an agent definition, wraps it in opinionated infrastructure, delivers a deployable unit.

---

## At Which Layer Should You Build?

This is the question that replaced "LangGraph or CrewAI?"

**Build at the Control Plane (LangGraph / Claude SDK) when:**
- Your routing logic is genuinely novel or domain-specific
- You need tight control over state transitions (financial workflows, auditability requirements)
- You're building infrastructure that others will deploy agents on top of
- You're doing research on agent architectures

**Build at the Context Harness layer when:**
- You're deploying an agent for a specific, well-defined purpose
- Security and credential isolation are requirements, not nice-to-haves
- You need multi-protocol support (MCP + A2A) without custom integration work
- Your team's bottleneck is deployment and ops, not agent reasoning design
- You want the agent's definition to be readable by non-engineers

**The most interesting case** is hybrid: custom execution logic built on LangGraph, fronted by a harness-compatible deployment format. Deep Agents Deploy supports this already — `deepagents deploy` can front a custom LangGraph backend while still providing standard endpoints and memory management.

---

## What Hasn't Changed

Before closing with optimism about the new stack, a few things stay constant regardless of layer:

**Model quality still dominates.** A well-designed harness on a weak model underperforms a poorly-designed one on a strong model. The Brain is still Claude.

**Task decomposition still matters.** Neither platform handles ambiguous, underspecified tasks. The agent's identity still needs to be clear about what "done" looks like.

**Evaluation is still your problem.** Managed Agents produces session logs. Deep Agents Deploy integrates with LangSmith. But deciding whether the agent did the *right thing* is still on you.

---

## The Takeaway

The question shifted. It used to be "which framework?" Now it's "which layer?"

Agent definition is moving from:
> *"Write the code that implements the behavior"*

to:
> *"Define the identity and skills; let the platform handle execution, memory, and deployment"*

Low-level frameworks aren't disappearing — they're becoming the substrate that platforms run on, the way TCP/IP didn't disappear when HTTP arrived. But for most production deployments, the harness is the product.

AgentOS is the primitives layer, still being defined by both academia and industry. Context Harness is the opinionated packaging of those primitives. Harness Engineering is the discipline to build and maintain the stack correctly.

Every developer building agents in 2026 needs to understand all three — primitives to understand *why* the platform works the way it does, context harness to ship fast in most cases, and harness engineering to not be surprised when the platform defaults aren't enough.

That's the division of labor that the next few years of production agent work will run on.

---

*Sources:*

- [Scaling Managed Agents: Decoupling the brain from the hands — Anthropic Engineering](https://www.anthropic.com/engineering/managed-agents)
- [Deep Agents Deploy: an open alternative to Claude Managed Agents — LangChain Blog](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Harness Engineering: first thoughts — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html)
- [Harness Engineering — OpenAI](https://openai.com/index/harness-engineering/)
- [AIOS: LLM Agent Operating System — arXiv](https://arxiv.org/abs/2403.16971)
- [AgenticOS 2026 Workshop — os-for-agent.github.io](https://os-for-agent.github.io/)
- [2025 Was Agents. 2026 Is Agent Harnesses — Aakash Gupta, Medium](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)
