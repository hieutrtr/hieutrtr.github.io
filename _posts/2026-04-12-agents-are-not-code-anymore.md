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

## The Current Stack: A Spectrum of Abstraction

*(What follows is a mental model for thinking about the ecosystem — not an established industry standard or official taxonomy. The "layers" below aren't clean boundaries; they're positions on a spectrum of how much you build versus how much you configure.)*

Think of it as a dial from "write everything in code" to "just describe what the agent is":

**Low end — You write the orchestration directly**

You're calling LLM APIs, wiring tool execution, managing state, and handling retries in code you own. Every decision about routing, memory, and error handling is explicit in your codebase.

*Tools: LangGraph, Claude Agent SDK, raw Anthropic/OpenAI API*

*Example: You write a LangGraph graph in Python — nodes for each step, edges for routing, explicit state management. Full control, full responsibility.*

**Middle — A framework handles the plumbing**

The framework provides abstractions (agents, chains, memory) and manages the LLM call loop. You focus on tool configuration and agent logic, not raw API bookkeeping. Still code-heavy, but with guardrails.

*Example: LangChain agents with built-in memory and tool abstractions. You write chains and tools; the framework handles orchestration.*

**Upper middle — A platform owns deployment and runtime**

You describe *who the agent is* — its role, skills, and knowledge — in structured files. The platform handles deployment, memory, sandboxing, credential isolation, and protocol support. What used to take a week of infrastructure work becomes a single command.

*Example (Deep Agents Deploy): You write `AGENTS.md` (identity and behavior in plain markdown) and `SKILL.md` files (domain knowledge loaded on demand). Run `deepagents deploy`. The platform handles memory, sandboxing, and serves MCP + A2A + Agent Protocol endpoints.*

*Example (Claude Managed Agents): You configure what the agent does. Anthropic's infrastructure separates the Brain (Claude + control logic) from the Hands (execution sandbox with no credential access). An append-only session log acts as durable external memory — if the Brain crashes mid-task, a new instance resumes from the last event.*

**High end — Pure configuration, no custom infrastructure**

A well-prompted model with the right tools. No custom orchestration, no deployment pipeline — just a system prompt and a tool list.

*Example: A Claude instance with search and a code interpreter. Well-prompted, tightly scoped, easy to debug.*

---

The two April releases — Managed Agents and Deep Agents Deploy — sit in the "upper middle" range. They're not eliminating code entirely. But they pull a substantial chunk of work (memory management, credential isolation, multi-protocol support, sandboxing) out of application code and into platform infrastructure.

Before 2026, building the equivalent of "credential-isolated sandbox + append-only session log + MCP/A2A/Agent Protocol server" would take a week of infrastructure setup. These platforms make it the default.

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

In a coupled design, code that an agent generates or executes runs in the same process space as its credentials. A successful prompt injection in that scenario doesn't just compromise one task — it potentially exposes everything that agent can access.

Credential isolation addresses this directly. It's not a new concept: service meshes, secret managers like Vault, and per-service IAM roles have applied this principle for years. The pattern is well understood — limit what any given execution context can reach.

Managed Agents' Brain/Hands separation applies this principle to agent architecture: the sandbox where generated code runs has no access to authentication tokens. Deep Agents Deploy enforces similar isolation through its sandbox provider abstraction.

You can achieve the same isolation when building your own harness — the pattern is architectural, not proprietary. Teams that already take security seriously in their own infra do this. The difference is packaging: context harness platforms make credential isolation the default rather than a deliberate design decision. That's a real convenience benefit, not a claim that self-built systems are inherently less secure.

The honest trade-off: build your own harness and you get full control over the security model, at the cost of designing and maintaining it yourself. Use a platform harness and you get a tested default, at the cost of depending on the platform's security assumptions. Neither is strictly better — they're different bets on where to spend the effort.

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

This question actually splits into two — and they're easy to conflate:

**Question 1: Which tooling/platform?** (position on the spectrum)

**Build with low-level frameworks (LangGraph / Claude Agent SDK) when:**
- Your routing logic is genuinely novel or domain-specific
- You need tight control over state transitions — financial workflows, auditability requirements
- You're building infrastructure that others will deploy agents on top of
- You're doing research on agent architectures, not deploying a product

**Build with managed platforms (Managed Agents / Deep Agents Deploy) when:**
- You're deploying an agent with a specific, well-defined purpose
- Security and credential isolation are requirements, not nice-to-haves
- You need multi-protocol support without custom integration work
- Your bottleneck is deployment and operations, not agent reasoning design
- You want the agent's definition to be readable and modifiable by non-engineers

**Question 2: How much orchestration does the agent actually need?** (Anthropic's 3 levels — a separate axis)

Anthropic's progression — Augmented LLM → Workflows → Agents — describes agent *complexity*, not platform choice. A managed platform can run a Level 1 agent. A custom LangGraph harness can implement Level 3 patterns. The two axes are orthogonal.

The default answer on this axis: **start at Level 1 and only add complexity when you can measure the improvement.** Most problems that seem to need multi-agent workflows can be handled by a single well-prompted model with the right tools. Adding Level 2 workflow patterns or Level 3 autonomous control is only worth it when performance measurably improves — regardless of which platform you're on.

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
