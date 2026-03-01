---
layout: post
title: "Complete Context Management Architecture for Agentic AI"
description: "Context management is not memory management. This post analyzes the 2026 landscape, proposes a 4-component + 3-tier + 3-pipeline architecture, and identifies what's 60% available vs 40% custom build."
featured: true
lang: en
ref: context-management-architecture
permalink: /context-management-architecture/
banner: /assets/images/blog-cm-05-architecture-overview.drawio.png
banner_alt: "Architecture Overview - Context Management Layer"
date: 2026-03-01
---

## Introduction — Context Is the Core Problem, Not the Model

In 2024, the AI community raced on models — GPT-4, Claude 3, Gemini. In 2025, the race shifted to agents — AutoGPT, CrewAI, LangGraph. But by 2026, one truth became clear: **the bottleneck isn't model intelligence — it's context management.**

An agent with GPT-5 but poor context management still loses to an agent with GPT-4o-mini but excellent context management. Letta proved this when a simple GPT-4o-mini agent scored 74% on the LoCoMo benchmark — higher than Mem0g (68.5%) with a far more complex pipeline.

This post doesn't compare models. It answers the question: **what does a complete context management architecture for an agentic platform look like?** — and more importantly: **what can you use off-the-shelf, and what do you need to build?**

---

## 1. Context Management ≠ Memory Management

This is the most important distinction that most engineering teams are overlooking.

### Memory Management: "What to Remember and What to Forget"

Memory management answers: *how do we store, retrieve, update, and delete knowledge over time?*

It's like the human long-term memory — the hippocampus stores memories, the cortex organizes knowledge. When you ask *"which framework does this user prefer?"*, memory management finds the answer from storage.

**Memory management products:** Mem0, Zep, LangMem, Supermemory.

### Context Management: "What to Feed the Brain Right Now"

Context management encompasses memory management and extends far beyond. It answers: *given the current query, how should I compose the prompt so the LLM responds optimally — including what to select from memory, how to compress, how to order, and how to allocate the token budget for each section?*

It's like the active brain — not just retrieving memories but organizing attention, deciding what to focus on, what to ignore, and synthesizing from multiple sources.

![Context Management vs Memory Management](/assets/images/blog-cm-01-context-vs-memory.drawio.png)

### Mapping to "Context Engineering 2.0"

The paper *"Context Engineering 2.0: The Context of Context Engineering"* (arXiv:2510.26493) defines context engineering as **8 operations**:

```
f_context(C) = F(φ₁, φ₂, ..., φ₈)(C)

① Collecting    — Gather raw context from multiple sources
② Storing       — Store & manage
③ Representing  — Represent consistently
④ Processing    — Process multimodal inputs
⑤ Self-baking   — Digest raw context → persistent knowledge
⑥ Selecting     — Choose the best context for the current query
⑦ Sharing       — Share between agents/systems
⑧ Adapting      — Adjust strategy based on feedback
```

**Memory management = φ₂ (Storing) + φ₅ (Self-baking)**. Only 2 of 8 operations.

**Context management = The full F(φ₁...φ₈)** — a superset encompassing memory plus collection, selection, compression, sharing, and adaptation.

### Direct Comparison

| Dimension | Memory Management | Context Management |
|---|---|---|
| **Core question** | "What to store, where, and when to retrieve?" | "What to put in the prompt THIS TIME for the best LLM response?" |
| **Time scope** | Long-term — cross-session, cross-task | Immediate — per-turn, per-request |
| **Output** | Stored/retrieved knowledge items | Optimally composed prompt |
| **Key metric** | Recall accuracy, storage cost | Response quality, token efficiency, KV-cache hit rate |
| **Includes** | Store + Retrieve + Update + Forget | Memory + Prompt engineering + Selection + Compression + Sharing + Adaptation |

### Concrete Example

When a user sends *"Refactor auth module"*:

**Memory management** retrieves: *"User decided to use JWT, auth.py is being split into 2 files, last time was blocked because DB schema wasn't updated."*

**Context management** does everything memory does, PLUS:
- **Token budget allocation**: 10K system prompt + 30K long-term facts + 40K session history + 80K recent turns + 20K user input + 20K response
- **Selection & ranking**: From 50 memory items, select the 8 most relevant using a scoring formula (e.g.: semantic similarity × 0.4 + recency × 0.25 + frequency × 0.15 + dependency × 0.2)
- **Compression**: Summarize 20 old turns into 1 paragraph
- **Prompt composition**: Arrange in optimal order (system → facts → session → recent → user) following "Lost in the Middle" research
- **KV-cache optimization**: Keep system prompt stable at prefix for cache hits
- **Adaptation**: If user previously complained about missing context, increase weight for dependency scoring

**Insight:** Mem0, Zep, and Supermemory each only solve the "memory" part — none solve "context management" completely. Letta (MemGPT) comes closest as it manages both memory and context window allocation, but still lacks a compression pipeline and cross-system sharing.

![Memory Management vs Context Management - Side by Side Flow](/assets/images/blog-cm-02-side-by-side-flow.drawio.png)

---

## 2. Current Tool Landscape (2026)

The AI memory/context market is booming. To understand the landscape, I categorize by **role in the 8 operations of [Context Engineering 2.0](https://arxiv.org/abs/2510.26493)** (φ₁-φ₈), not by marketing category.

![Tool Positioning Matrix](/assets/images/blog-cm-03-tool-positioning-matrix.drawio.png)

### 2.1 Memory Storage & Retrieval Layer

Tools specialized in storing and retrieving memory. This is **φ₂ (Storing)** in the framework.

| Tool | Approach | Storage | Strengths | Weaknesses |
|---|---|---|---|---|
| **[Mem0](https://mem0.ai)** | Auto-extract facts → hybrid store | Vector (Qdrant/Chroma/FAISS) + Graph (Neo4j) + KV (Redis) | Drop-in 3-line integration, 90% token savings proven, AWS partnership | Graph memory paywalled ($249/mo), black-box extraction, 13x price jump |
| **[Zep](https://getzep.com)** | Temporal knowledge graph | Graphiti (bi-temporal graph) + vector | Time-aware memory, knows when facts change, strong enterprise focus | Smaller ecosystem, no validation/dedup layer |
| **[Supermemory](https://supermemory.ai)** | Brain-inspired auto-forget | Cloudflare Durable Objects + KV + Postgres | Sub-400ms, smart decay, Claude Code plugin | English only, plugin v0.0.2, declining benchmark position |
| **[LangMem](https://langchain-ai.github.io/langmem/)** | Library for LangGraph | Flat KV + vector search | Free, pip install, no API keys | Requires LangGraph, no graph/relationship modeling |
| **[Cognee](https://cognee.ai)** | Knowledge graph pipeline | RDF ontologies + vector/graph DB | Strong structured extraction, on-premises | Newer, smaller community |

### 2.2 Agent Frameworks with Built-in Memory

Frameworks treating memory as a first-class citizen in the agent runtime. Combining **φ₂ + φ₅ + φ₆**.

| Tool | Approach | Storage | Strengths | Weaknesses |
|---|---|---|---|---|
| **[Letta (MemGPT)](https://letta.com)** | LLM-as-OS: agent self-manages memory via tools | PostgreSQL + pgvector | White-box memory, git-versioned Context Repositories (Feb 2026), full OSS parity | Depends on LLM intelligence, no native graph, smaller community |
| **[Google ADK](https://google.github.io/adk-docs/)** | Session-based + context compaction | Configurable (memory stores) | Sliding window compaction (60-80% token reduction), multi-level (conversation/workflow/tool), incremental | Session-centric, less cross-session persistence |

### 2.3 Context Compression Tools

Specialized in compressing context to fit within context windows. Part of **φ₅ (Self-baking)**.

| Tool | Approach | Compression Ratio | Strengths | Weaknesses |
|---|---|---|---|---|
| **[LLMLingua 2](https://llmlingua.com)** (Microsoft) | Perplexity-based token filtering | Up to 20x | 3-6x faster than v1, LangChain/LlamaIndex integration | Compression only, no memory/persistence |
| **[Acon](https://arxiv.org/abs/...)** | Agent observation compression | 26-54% peak token reduction | Purpose-built for agentic workflows | Research stage, not production-ready |
| **[Mastra OM](https://mastra.ai/docs/memory/observational-memory)** | Observer + Reflector agents compress conversation | 3-6x (text), 5-40x (tool output) | 94.87% LongMemEval (SOTA), no vector DB needed, leverages prompt caching | Text-only, no external knowledge retrieval |

### 2.4 Context Protocol & Sharing

Standardizing how agents share context. This is **φ₇ (Sharing)**.

| Tool | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **[MCP](https://modelcontextprotocol.io)** (Anthropic → Linux Foundation) | Standardized bidirectional protocol | Adopted by OpenAI, Google DeepMind; SDKs for Python/TS/C#/Java | Memory server still basic |
| **[MCP Memory Server](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)** | Knowledge graph via MCP | Standard protocol, compatible with Claude/Cursor/VS Code | Basic (entity-relation triplets only) |
| **[OpenMemory](https://github.com/mem0ai/openmemory)** (Mem0) | Local-first memory via MCP | Privacy-focused, migration tool from Mem0/Zep/Supermemory | Dependent on Mem0 ecosystem |
| **[Letta Agent File (.af)](https://docs.letta.com)** | Open format for portable agents | Includes full memory state | Letta-specific |

### 2.5 Context Versioning & Tracing

Tracking context over time. Part of **φ₈ (Adapting)**.

| Tool | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **[Lilypad](https://mirascope.com)** (Mirascope) | Context tracing & versioning | Track context changes, compare versions | No memory/persistence |
| **[Letta Context Repositories](https://www.letta.com/blog/context-repositories)** | Git-backed memory filesystem | Rollbacks, changelogs, diffs, multi-agent concurrent writes | New (Feb 2026), not yet stable |

### 2.6 Emerging: Self-Baking & Evolving Context

Research-stage tools for **φ₅ (Self-baking) + φ₈ (Adapting)** — no production products yet.

| Research | Approach | Results |
|---|---|---|
| **[ACE](https://arxiv.org/abs/2510.04618)** (Stanford/SambaNova) | Generator → Reflector → Curator: delta contexts, not full rewrites | +10.6% agent benchmarks, 86.9% lower adaptation latency |
| **[Dynamic Cheatsheet](https://arxiv.org/abs/2504.07952)** | Adaptive external memory of reusable strategies | Claude 3.5 AIME: >2x accuracy; GPT-4o Game of 24: 10%→99% |

### Summary: No One Solves It Completely

The table below is the **author's assessment** based on documentation review and hands-on testing of each tool, mapped to the [8 operations of Context Engineering 2.0](https://arxiv.org/abs/2510.26493):

![Tool Coverage Heatmap](/assets/images/blog-cm-04-tool-coverage-heatmap.drawio.png)

**Biggest gaps: φ₆ (Selecting), φ₈ (Adapting), and a unified integration pipeline.**

---

## 3. Complete Architecture for an Agentic Platform

Based on the φ₁-φ₈ framework from [Context Engineering 2.0](https://arxiv.org/abs/2510.26493), gap analysis from Section 2, and practical experience from [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus), [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), and [Google ADK](https://google.github.io/adk-docs/), I propose the following architecture:

### 3.1 Architecture Overview

![Architecture Overview - Context Management Layer](/assets/images/blog-cm-05-architecture-overview.drawio.png)

### 3.2 Component 1: Context Collector (φ₁ + φ₃)

**Role:** Collect context from all sources and represent them uniformly as `ContextItem`.

```python
@dataclass
class ContextItem:
    id: str                          # Unique identifier
    content: str                     # Content
    source: ContextSource            # USER | FILE | TOOL | HISTORY | MEMORY
    timestamp: datetime              # Creation time
    tags: list[str]                  # Functional/semantic tags
    importance: float                # 0.0 - 1.0
    token_count: int                 # Estimated tokens
    metadata: dict                   # Additional data
    embedding: Optional[list[float]] # Lazy-computed vector
```

**Collection sources:**
- **User input** — Messages, commands
- **File system** — CLAUDE.md, source code, configs
- **Tool output** — Search results, file reads, API responses
- **Session history** — Previous turns in current session
- **Long-term memory** — Cross-session facts and summaries
- **External sources** — MCP servers, databases, APIs

### 3.3 Component 2: Context Manager (φ₂ + φ₄ + φ₅)

**Role:** Manage the context lifecycle — store, compress, self-bake, transfer between memory tiers.

**3 Memory Tiers:**

| Tier | Storage | Lifespan | Content |
|---|---|---|---|
| **Working Memory** | In-RAM | Current session only | Raw conversation turns, active tool outputs |
| **Session Memory** | SQLite | Persisted per session | Compressed history, session summaries |
| **Long-term Memory** | File (JSON/MD) + Vector DB | Cross-session | Facts, decisions, preferences, embeddings |

**Self-baking Pipeline** — concept from [Context Engineering 2.0](https://arxiv.org/abs/2510.26493) (φ₅), pattern 4 inspired by [ACE](https://arxiv.org/abs/2510.04618). This is the biggest differentiator — no tool fully implements it yet:

![Self-Baking Pipeline: 4 Consolidation Patterns](/assets/images/blog-cm-06-self-baking-pipeline.drawio.png)

**Memory Transfer Rules:**

```
Short-term → Long-term when:
  - importance(item) > θ_importance (e.g., 0.7)
  - item contains: decision, preference, architecture choice, error-solution
  - session ends → auto-bake remaining working memory
```

### 3.4 Component 3: Context Selector (φ₆)

**Role:** Select the optimal context subset for each query, fitting within the token budget.

**Scoring formula** (proposed — the weights below are suggested starting values that need tuning per use case):

![Context Scoring Formula: Factor Weights](/assets/images/blog-cm-08-scoring-weights.drawio.png)

**Token Budget Allocation** (proposed — suggested ratios for a 200K context window, adjust per model and workload):

![Token Budget Allocation](/assets/images/blog-cm-07-token-budget.drawio.png)

**KV-cache optimization** (insight from [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)):

> *"If you had to choose just one metric, the KV-cache hit rate is the single most important metric for a production-stage AI agent."*

Keep system prompt + long-term context stable at prefix → cache hit → reduced latency + cost.

### 3.5 Component 4: Context Sharer (φ₇ + φ₈)

**Role:** Share context between agents and cross-system, adjust strategy based on feedback.

**Intra-system sharing:**
- Agent → subagent: Embed relevant context in prompt
- Shared memory namespace: Blackboard pattern
- Multi-agent concurrent writes: Git worktrees (Letta approach)

**Cross-system sharing:**
- MCP protocol: Standardized tools/resources
- Export/Import: JSON, Markdown, YAML
- File-based: CLAUDE.md, GEMINI.md patterns

**Adaptation loop:**
- Track response quality metrics
- If context selection led to poor response → adjust scoring weights
- If compression was too aggressive → lower compression ratio
- A/B test different context compositions

---

## 4. Mapping Tools to Architecture — What to Use, What to Build

This is the most practical section: given the 4-component + 3-tier + 2-pipeline architecture above, which tools can be plugged in, and which gaps need custom work?

### 4.1 Comprehensive Mapping Table

![Component Mapping: Available Tools vs Custom Build](/assets/images/blog-cm-13-component-mapping.drawio.png)

### 4.2 Summary: 60% Off-the-Shelf, 40% Custom Build

**Off-the-shelf tools (60%):**
- Memory storage: Mem0 or Zep (depending on use case)
- Vector search: FAISS/Chroma (self-hosted) or Qdrant/Pinecone (managed)
- NL compression: LLM-based summarization or LLMLingua 2
- Observational compression: Mastra OM (if using TypeScript)
- Protocol: MCP for cross-tool sharing
- Static context: CLAUDE.md pattern
- Session compaction: Google ADK compaction (if using ADK)

**Custom build required (40%) — this is where differentiation lives:**

| Component to Build | Why No One Has Built It | Complexity |
|---|---|---|
| **Context Selector** (scoring + budgeting) | Each tool only handles retrieval, none handles aggregation + ranking + budgeting | Medium |
| **Self-baking Pipeline** | Recently proposed in paper (2025), no tool implements it properly | High |
| **Schema Extraction Engine** | Requires domain-specific schemas, hard to generalize | Medium |
| **KV-cache Optimizer** | Requires understanding LLM internals, few people focus here | Medium |
| **Adaptation Loop** | Needs feedback signals, no one has clearly defined metrics | High |
| **Unified .context/ Protocol** | CLAUDE.md is static; needs a dynamic, auto-updated version | Medium |
| **Export/Import Pipeline** | Each tool has its own format, needs standardization | Low |

![Build vs Buy: 60/40 Split](/assets/images/blog-cm-10-build-vs-buy.drawio.png)

---

## 5. Suggested Compositions & Use Cases

### 5.1 Composition A: "Production Chatbot" — Drop-in Memory

**Use case:** Customer support bot that needs to remember user preferences and interaction history.

```
┌─────────────────────────────────────────────┐
│              YOUR APP / AGENT               │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│         Mem0 (Memory Layer)                 │
│  • Auto-extract facts from conversations    │
│  • Retrieve relevant memories per query     │
│  • User/session/agent scoping built-in      │
└──────────────────────┬──────────────────────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
        Vector Store       Graph Store
        (Qdrant)           (Neo4j — $249/mo)
```

**Tools used:** Mem0 (managed platform)
**Custom build:** None — Mem0 handles everything
**Cost:** $19-249/mo managed, or self-hosted (vector DB only, graph paywalled)
**When to use:** MVP, B2B copilot, customer support — need speed, no customization required

**Code:**

```python
from mem0 import Memory
from openai import OpenAI

memory = Memory()
client = OpenAI()

def chat(message: str, user_id: str) -> str:
    relevant = memory.search(query=message, user_id=user_id, limit=5)
    memories_str = "\n".join(f"- {m['memory']}" for m in relevant["results"])

    response = client.chat.completions.create(
        model="gpt-4.1-nano",
        messages=[
            {"role": "system", "content": f"User context:\n{memories_str}"},
            {"role": "user", "content": message},
        ],
    )
    answer = response.choices[0].message.content
    memory.add([{"role": "user", "content": message},
                {"role": "assistant", "content": answer}], user_id=user_id)
    return answer
```

---

### 5.2 Composition B: "Stateful Agent" — Full Agent Runtime

**Use case:** Long-running agent that needs self-managed memory, transparent reasoning, and audit trail.

```
┌─────────────────────────────────────────────┐
│          Letta Agent Runtime                │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │        CONTEXT WINDOW               │   │
│  │  Core Memory Blocks (in-context)    │   │
│  │  ┌──────────┐  ┌─────────────────┐ │   │
│  │  │ "persona"│  │    "human"      │ │   │
│  │  └──────────┘  └─────────────────┘ │   │
│  │  Recent conversation messages       │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  Memory Tools (agent calls autonomously):   │
│  • memory_replace / memory_insert           │
│  • archival_memory_search                   │
│  • conversation_search                      │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │   PostgreSQL + pgvector             │   │
│  │   Archival + Conversational Memory  │   │
│  │   Context Repositories (git-backed) │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Tools used:** Letta (self-hosted or Cloud)
**Custom build:** Domain-specific memory block schemas
**Cost:** Free (self-hosted, single PostgreSQL) or $20/mo (Cloud)
**When to use:** AI researcher assistant, document analysis agent, autonomous coding agent

---

### 5.3 Composition C: "Long-Running Coding Agent" — Memory + Compression

**Use case:** Coding agent running for hours, context window overflow, needs to remember decisions across sessions.

```
┌─────────────────────────────────────────────────────┐
│              CODING AGENT (Claude Code / OpenCode)   │
└──────────────────────────┬──────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
┌────────────────┐ ┌──────────────┐ ┌───────────────┐
│  Supermemory   │ │  Mastra OM   │ │  CLAUDE.md    │
│  (persistent   │ │  (compress   │ │  (static      │
│   project &    │ │   long conv  │ │   project     │
│   user memory) │ │   → obs log) │ │   context)    │
└────────────────┘ └──────────────┘ └───────────────┘
         │                 │                 │
         └────── Combined context ───────────┘
                           │
                    ┌──────▼──────┐
                    │ LLM Backend │
                    └─────────────┘
```

**Tools used:** Supermemory (memory) + Mastra OM (compression) + CLAUDE.md (static context)
**Custom build:** Context Selector (scoring + budgeting when combining 3 sources)
**Cost:** Supermemory $19/mo + Mastra (self-hosted, free)
**When to use:** Long coding sessions, multi-session refactoring projects

---

### 5.4 Composition D: "Enterprise Platform" — Full Context Management

**Use case:** Enterprise agentic platform needing full context management — multi-agent, compliance, temporal reasoning.

![Enterprise Context Management Architecture](/assets/images/blog-cm-09-enterprise-architecture.drawio.png)

**Tools used:**
- Zep (temporal knowledge graph — $)
- LLMLingua 2 (token compression — free)
- MCP SDK (protocol — free)
- FAISS/Chroma (vector search — free)

**Custom build:**
- Context Selector (scoring + budgeting + KV-cache optimization)
- Self-baking Pipeline (4 patterns)
- Schema Extraction Engine
- .context/ file protocol
- Adaptation Loop
- Export/Import pipeline

**Cost:** Zep Cloud (custom pricing) + self-hosted infra + engineering team
**When to use:** Enterprise platform serving multiple agent types, compliance-heavy, temporal reasoning needed

---

### 5.5 Composition E: "Lightweight CLI Agent" — File-First, Zero Dependencies

**Use case:** CLI agent (like Claude Code) needing persistent memory across sessions with zero external services.

```
┌─────────────────────────────────────────────┐
│              CLI AGENT                       │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│     Lightweight CML (embedded library)       │
│                                              │
│  Working Memory:  In-process dictionary      │
│  Session Memory:  SQLite (local file)        │
│  Long-term:       .context/ directory        │
│     ├── facts.json     (schema-baked facts)  │
│     ├── summaries/     (NL summaries by topic)│
│     ├── sessions/      (SQLite DB)           │
│     └── config.yaml    (CML configuration)   │
│                                              │
│  Compression:  LLM-based summarization       │
│  Selection:    Simple recency + importance    │
│  Self-baking:  Session-end auto-bake          │
│                                              │
│  NO external DB. NO vector store.            │
│  NO running services. Just files.            │
└──────────────────────────────────────────────┘
```

**Tools used:** SQLite (built-in), LLM (any)
**Custom build:** Entire lightweight CML library
**Cost:** $0 infrastructure (LLM API cost only)
**When to use:** CLI tools, developer workflows, offline-capable agents

This is the most suitable composition for the context-engineering project in this repo — a lightweight, file-first context management layer compatible with the CLAUDE.md pattern, with no external service dependencies.

---

## 6. Decision Framework — Which Composition to Choose?

![Decision Flowchart: Which Composition?](/assets/images/blog-cm-11-decision-flowchart.drawio.png)

---

## 7. What Will Change (2026-2027)

### Trends Shaping the Landscape

1. **Memory as infrastructure, not feature.** Just as databases became an infrastructure layer, AI memory is undergoing a similar transition. 2026 is the year memory shifts from experimental to essential.

2. **Convergence of memory types.** Production systems are combining multiple memory types instead of choosing one: vector + graph + temporal + observational. Boundaries between tools are blurring.

3. **Self-baking will become standard.** As the [Context Engineering 2.0](https://arxiv.org/abs/2510.26493) paper gains traction, agents self-digesting raw context into structured knowledge will become standard practice — similar to how RAG became standard after 2023.

4. **Context is the new data engineering.** As Andrej Karpathy said, context engineering isn't prompt engineering — it's software architecture for intelligence. Teams will need "context engineers" just as they need data engineers.

5. **MCP will standardize the sharing layer.** With OpenAI and Google DeepMind adopting MCP, it will become the standard protocol for agent-to-tool and agent-to-agent communication — including memory sharing.

![Context Management: Evolution Timeline](/assets/images/blog-cm-12-trends-timeline.drawio.png)

---

## Conclusion

Back to the original question: **what does a complete context management architecture include?**

**4 components:**
1. **Collector** — Gather context from all sources, normalize into a unified format
2. **Manager** — 3-tier storage (working/session/long-term), compression, self-baking
3. **Selector** — Scoring, ranking, token budgeting, KV-cache optimization
4. **Sharer** — Cross-agent sharing (MCP), export/import, adaptation loop

**3 pipelines:**
1. **Compression Pipeline** — NL summary + token-level + schema extraction + vector
2. **Self-baking Pipeline** — Raw → Summarized → Structured → Abstract (with backlinks)
3. **Adaptation Pipeline** — Track quality → adjust weights → improve over time

**60% off-the-shelf, 40% custom build.** The parts that need building — Selector, Self-baking, Adaptation — are precisely the parts that create differentiation. That's why there's no "all-in-one" solution: because the hardest parts remain unsolved.

Memory management answers *"what to remember and what to forget"*. Context management answers *"what to remember, what to forget, AND what to put in the prompt right now, how to order it, how to compress it, who to share with, and how to adjust based on results"*.

Memory is storage. Context management is the brain.

---

## Sources

### Research Papers
- [Context Engineering 2.0 (arXiv:2510.26493)](https://arxiv.org/abs/2510.26493) — Formal framework for context engineering
- [MemGPT (arXiv:2310.08560)](https://arxiv.org/abs/2310.08560) — OS-inspired memory management (ICLR 2024)
- [Mem0 (arXiv:2504.19413)](https://arxiv.org/abs/2504.19413) — Production-ready memory layer (ECAI 2025)
- [Agentic Context Engineering / ACE (arXiv:2510.04618)](https://arxiv.org/abs/2510.04618) — Evolving playbooks (ICLR 2026)
- [Dynamic Cheatsheet (arXiv:2504.07952)](https://arxiv.org/abs/2504.07952) — Adaptive memory for test-time learning

### Products & Tools
- [Mem0](https://mem0.ai) — Memory-as-a-Service ([GitHub](https://github.com/mem0ai/mem0), ~44K stars)
- [Letta](https://letta.com) — Stateful agent framework ([GitHub](https://github.com/letta-ai/letta), ~21K stars)
- [Zep](https://getzep.com) — Temporal knowledge graph memory
- [Supermemory](https://supermemory.ai) — Universal memory API ([GitHub](https://github.com/supermemoryai/supermemory), ~13.5K stars)
- [LangMem](https://langchain-ai.github.io/langmem/) — Memory library for LangGraph
- [Cognee](https://cognee.ai) — Knowledge graph memory
- [LLMLingua](https://llmlingua.com) — Prompt compression (Microsoft)
- [Mastra OM](https://mastra.ai/docs/memory/observational-memory) — Observational Memory (SOTA on LongMemEval)
- [Google ADK](https://google.github.io/adk-docs/) — Agent Development Kit with context compaction
- [MCP](https://modelcontextprotocol.io) — Model Context Protocol (Anthropic → Linux Foundation)

### Industry Insights
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Manus: Context Engineering Lessons](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [Google: Context-Aware Multi-Agent Framework for Production](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/)
- [Letta: Benchmarking AI Agent Memory](https://www.letta.com/blog/benchmarking-ai-agent-memory)
- [VentureBeat: Observational Memory Cuts Agent Costs 10x](https://venturebeat.com/data/observational-memory-cuts-ai-agent-costs-10x-and-outscores-rag-on-long)
- [The New Stack: Memory for AI Agents — A New Paradigm of Context Engineering](https://thenewstack.io/memory-for-ai-agents-a-new-paradigm-of-context-engineering/)
- [Graphlit: Survey of AI Agent Memory Frameworks](https://www.graphlit.com/blog/survey-of-ai-agent-memory-frameworks)
