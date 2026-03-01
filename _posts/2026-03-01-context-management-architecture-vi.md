---
layout: post
title: "Kiến Trúc Hoàn Chỉnh Cho Context Management Trong Agentic AI"
description: "Context management không phải memory management. Bài viết phân tích landscape 2026, đề xuất kiến trúc 4 components + 3 tiers + 3 pipelines, và chỉ rõ 60% dùng tool có sẵn, 40% cần xây mới."
featured: true
lang: vi
ref: context-management-architecture
permalink: /vi/context-management-architecture/
banner: /assets/images/blog-cm-05-architecture-overview.drawio.png
banner_alt: "Architecture Overview - Context Management Layer"
date: 2026-03-01
---

## Mở đầu — Context là vấn đề cốt lõi, không phải model

Năm 2024, cộng đồng AI chạy đua về model — GPT-4, Claude 3, Gemini. Năm 2025, cuộc đua chuyển sang agent — AutoGPT, CrewAI, LangGraph. Nhưng đến 2026, một sự thật trở nên rõ ràng: **bottleneck không phải model intelligence, mà là context management.**

Một agent với GPT-5 nhưng context management kém vẫn thua một agent với GPT-4o-mini nhưng context management tốt. Letta đã chứng minh điều này khi agent đơn giản dùng GPT-4o-mini đạt 74% trên LoCoMo benchmark — cao hơn cả Mem0g (68.5%) với pipeline phức tạp hơn nhiều.

Bài viết này không so sánh model. Bài viết này trả lời câu hỏi: **một kiến trúc context management hoàn chỉnh cho agentic platform gồm những gì?** — và quan trọng hơn: **phần nào dùng tool có sẵn, phần nào cần xây mới?**

---

## 1. Context Management ≠ Memory Management

Đây là sự phân biệt quan trọng nhất mà hầu hết đội ngũ engineering đang bỏ qua.

### Memory Management: "Nhớ gì và quên gì"

Memory management trả lời câu hỏi: *làm sao lưu trữ, truy xuất, cập nhật, và xóa kiến thức qua thời gian?*

Nó giống như bộ nhớ dài hạn của con người — hippocampus lưu trữ ký ức, cortex tổ chức kiến thức. Khi bạn hỏi *"user này thích framework nào?"*, memory management lo tìm câu trả lời trong kho lưu trữ.

**Các sản phẩm memory management:** Mem0, Zep, LangMem, Supermemory.

### Context Management: "Đưa gì vào não lúc này"

Context management bao hàm memory management và mở rộng hơn nhiều. Nó trả lời câu hỏi: *với query hiện tại, tôi nên compose prompt như thế nào để LLM trả lời tốt nhất — bao gồm chọn gì từ memory, nén ra sao, sắp xếp thế nào, và phân bổ token budget cho từng phần?*

Nó giống như bộ não đang hoạt động — không chỉ truy xuất ký ức mà còn tổ chức attention, quyết định focus vào đâu, bỏ qua gì, và tổng hợp từ nhiều nguồn khác nhau.

![Context Management vs Memory Management](/assets/images/blog-cm-01-context-vs-memory.drawio.png)

### Mapping theo paper "Context Engineering 2.0"

Paper *"Context Engineering 2.0: The Context of Context Engineering"* (arXiv:2510.26493) định nghĩa context engineering gồm **8 operations**:

```
f_context(C) = F(φ₁, φ₂, ..., φ₈)(C)

① Collecting    — Thu thập raw context từ nhiều nguồn
② Storing       — Lưu trữ & quản lý
③ Representing  — Biểu diễn nhất quán
④ Processing    — Xử lý multimodal
⑤ Self-baking   — Tiêu hóa context thô → persistent knowledge
⑥ Selecting     — Chọn context phù hợp nhất cho query hiện tại
⑦ Sharing       — Chia sẻ giữa agents/systems
⑧ Adapting      — Điều chỉnh strategy dựa trên feedback
```

**Memory management = φ₂ (Storing) + φ₅ (Self-baking)**. Chỉ 2 trong 8 operations.

**Context management = Toàn bộ F(φ₁...φ₈)** — một superset bao trùm memory cộng thêm collection, selection, compression, sharing, adaptation.

### So sánh trực tiếp

| Dimension | Memory Management | Context Management |
|---|---|---|
| **Câu hỏi trung tâm** | "Lưu gì, ở đâu, lấy lại khi nào?" | "Đưa gì vào prompt LẦN NÀY để LLM trả lời tốt nhất?" |
| **Scope thời gian** | Dài hạn — cross-session, cross-task | Tức thì — per-turn, per-request |
| **Output** | Stored/retrieved knowledge items | Composed prompt tối ưu |
| **Metric chính** | Recall accuracy, storage cost | Response quality, token efficiency, KV-cache hit rate |
| **Bao gồm** | Store + Retrieve + Update + Forget | Memory + Prompt engineering + Selection + Compression + Sharing + Adaptation |

### Ví dụ cụ thể

Khi user gửi message *"Refactor auth module"*:

**Memory management** truy xuất: *"User đã quyết định dùng JWT, auth.py đang tách thành 2 file, lần trước bị block vì DB schema chưa update."*

**Context management** làm tất cả những gì memory làm, CỘNG THÊM:
- **Token budget allocation**: 10K system prompt + 30K long-term facts + 40K session history + 80K recent turns + 20K user input + 20K response
- **Selection & ranking**: Trong 50 memory items, chọn 8 relevant nhất bằng scoring formula (ví dụ: semantic similarity × 0.4 + recency × 0.25 + frequency × 0.15 + dependency × 0.2)
- **Compression**: Tóm tắt 20 turns cũ thành 1 paragraph
- **Prompt composition**: Sắp xếp thứ tự optimal (system → facts → session → recent → user) theo research về "Lost in the Middle"
- **KV-cache optimization**: Giữ system prompt ổn định ở prefix để cache hit
- **Adaptation**: Nếu lần trước user phàn nàn thiếu context, tăng weight cho dependency scoring

**Insight:** Mỗi Mem0, Zep, Supermemory chỉ giải quyết phần "memory" — chưa ai giải quyết trọn vẹn "context management". Letta (MemGPT) là tool tiến gần nhất vì nó quản lý cả memory lẫn context window allocation, nhưng vẫn thiếu compression pipeline và cross-system sharing.

![Memory Management vs Context Management - Side by Side Flow](/assets/images/blog-cm-02-side-by-side-flow.drawio.png)

---

## 2. Landscape Các Tools Hiện Nay (2026)

Thị trường AI memory/context đang bùng nổ. Để hiểu rõ landscape, tôi phân loại theo **vai trò trong 8 operations của [Context Engineering 2.0](https://arxiv.org/abs/2510.26493)** (φ₁-φ₈), không phải theo marketing category.

![Tool Positioning Matrix](/assets/images/blog-cm-03-tool-positioning-matrix.drawio.png)

### 2.1 Memory Storage & Retrieval Layer

Các tool chuyên lưu trữ và truy xuất memory. Đây là **φ₂ (Storing)** trong framework.

| Tool | Approach | Storage | Strengths | Weaknesses |
|---|---|---|---|---|
| **[Mem0](https://mem0.ai)** | Auto-extract facts → hybrid store | Vector (Qdrant/Chroma/FAISS) + Graph (Neo4j) + KV (Redis) | Drop-in 3-line integration, 90% token savings proven, AWS partnership | Graph memory paywalled ($249/mo), black-box extraction, 13x price jump |
| **[Zep](https://getzep.com)** | Temporal knowledge graph | Graphiti (bi-temporal graph) + vector | Time-aware memory, knows when facts change, strong enterprise focus | Smaller ecosystem, no validation/dedup layer |
| **[Supermemory](https://supermemory.ai)** | Brain-inspired auto-forget | Cloudflare Durable Objects + KV + Postgres | Sub-400ms, smart decay, Claude Code plugin | English only, plugin v0.0.2, declining benchmark position |
| **[LangMem](https://langchain-ai.github.io/langmem/)** | Library for LangGraph | Flat KV + vector search | Free, pip install, no API keys | Requires LangGraph, no graph/relationship modeling |
| **[Cognee](https://cognee.ai)** | Knowledge graph pipeline | RDF ontologies + vector/graph DB | Strong structured extraction, on-premises | Newer, smaller community |

### 2.2 Agent Framework with Built-in Memory

Các framework coi memory là first-class citizen trong agent runtime. Kết hợp **φ₂ + φ₅ + φ₆**.

| Tool | Approach | Storage | Strengths | Weaknesses |
|---|---|---|---|---|
| **[Letta (MemGPT)](https://letta.com)** | LLM-as-OS: agent self-manages memory via tools | PostgreSQL + pgvector | White-box memory, git-versioned Context Repositories (Feb 2026), full OSS parity | Depends on LLM intelligence, no native graph, smaller community |
| **[Google ADK](https://google.github.io/adk-docs/)** | Session-based + context compaction | Configurable (memory stores) | Sliding window compaction (60-80% token reduction), multi-level (conversation/workflow/tool), incremental | Session-centric, less cross-session persistence |

### 2.3 Context Compression Tools

Chuyên nén context để fit vào context window. Đây là một phần của **φ₅ (Self-baking)**.

| Tool | Approach | Compression Ratio | Strengths | Weaknesses |
|---|---|---|---|---|
| **[LLMLingua 2](https://llmlingua.com)** (Microsoft) | Perplexity-based token filtering | Lên đến 20x | Nhanh 3-6x so với v1, tích hợp LangChain/LlamaIndex | Chỉ compression, không memory/persistence |
| **[Acon](https://arxiv.org/abs/...)** | Agent observation compression | 26-54% peak token reduction | Dành riêng cho agentic workflows | Research, chưa production-ready |
| **[Mastra OM](https://mastra.ai/docs/memory/observational-memory)** | Observer + Reflector agents nén conversation | 3-6x (text), 5-40x (tool output) | 94.87% LongMemEval (SOTA), no vector DB needed, leverages prompt caching | Text-only, no external knowledge retrieval |

### 2.4 Context Protocol & Sharing

Chuẩn hóa cách agents chia sẻ context. Đây là **φ₇ (Sharing)**.

| Tool | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **[MCP](https://modelcontextprotocol.io)** (Anthropic → Linux Foundation) | Standardized bidirectional protocol | Adopted by OpenAI, Google DeepMind; SDKs Python/TS/C#/Java | Memory server còn basic |
| **[MCP Memory Server](https://github.com/modelcontextprotocol/servers/tree/main/src/memory)** | Knowledge graph qua MCP | Standard protocol, compatible với Claude/Cursor/VS Code | Basic (chỉ entity-relation triplets) |
| **[OpenMemory](https://github.com/mem0ai/openmemory)** (Mem0) | Local-first memory qua MCP | Privacy-focused, migration tool from Mem0/Zep/Supermemory | Phụ thuộc Mem0 ecosystem |
| **[Letta Agent File (.af)](https://docs.letta.com)** | Open format cho portable agents | Bao gồm cả memory state | Letta-specific |

### 2.5 Context Versioning & Tracing

Theo dõi context qua thời gian. Đây là một phần của **φ₈ (Adapting)**.

| Tool | Approach | Strengths | Weaknesses |
|---|---|---|---|
| **[Lilypad](https://mirascope.com)** (Mirascope) | Context tracing & versioning | Track context changes, compare versions | Không memory/persistence |
| **[Letta Context Repositories](https://www.letta.com/blog/context-repositories)** | Git-backed memory filesystem | Rollbacks, changelogs, diffs, multi-agent concurrent writes | Mới (Feb 2026), chưa ổn định |

### 2.6 Emerging: Self-Baking & Evolving Context

Research-stage tools cho **φ₅ (Self-baking) + φ₈ (Adapting)** — chưa có sản phẩm production.

| Research | Approach | Results |
|---|---|---|
| **[ACE](https://arxiv.org/abs/2510.04618)** (Stanford/SambaNova) | Generator → Reflector → Curator: delta contexts, not full rewrites | +10.6% agent benchmarks, 86.9% lower adaptation latency |
| **[Dynamic Cheatsheet](https://arxiv.org/abs/2504.07952)** | Adaptive external memory of reusable strategies | Claude 3.5 AIME: >2x accuracy; GPT-4o Game of 24: 10%→99% |

### Tóm lại: Không ai giải quyết trọn vẹn

Bảng dưới đây là **đánh giá của tác giả** dựa trên documentation review và hands-on testing của từng tool, mapping theo [8 operations của Context Engineering 2.0](https://arxiv.org/abs/2510.26493):

![Tool Coverage Heatmap](/assets/images/blog-cm-04-tool-coverage-heatmap.drawio.png)

**Gap lớn nhất: φ₆ (Selecting), φ₈ (Adapting), và integration pipeline thống nhất.**

---

## 3. Kiến Trúc Hoàn Chỉnh Cho Agentic Platform

Dựa trên framework φ₁-φ₈ của [Context Engineering 2.0](https://arxiv.org/abs/2510.26493), phân tích gap ở Section 2, và kinh nghiệm thực tế từ [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus), [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), [Google ADK](https://google.github.io/adk-docs/), tôi đề xuất kiến trúc sau:

### 3.1 Architecture Overview

![Architecture Overview - Context Management Layer](/assets/images/blog-cm-05-architecture-overview.drawio.png)

### 3.2 Component 1: Context Collector (φ₁ + φ₃)

**Vai trò:** Thu thập context từ mọi nguồn và biểu diễn thống nhất thành `ContextItem`.

```python
@dataclass
class ContextItem:
    id: str                          # Unique identifier
    content: str                     # Nội dung
    source: ContextSource            # USER | FILE | TOOL | HISTORY | MEMORY
    timestamp: datetime              # Thời điểm tạo
    tags: list[str]                  # Functional/semantic tags
    importance: float                # 0.0 - 1.0
    token_count: int                 # Estimated tokens
    metadata: dict                   # Additional data
    embedding: Optional[list[float]] # Lazy-computed vector
```

**Các nguồn thu thập:**
- **User input** — Messages, commands
- **File system** — CLAUDE.md, source code, configs
- **Tool output** — Search results, file reads, API responses
- **Session history** — Previous turns in current session
- **Long-term memory** — Cross-session facts and summaries
- **External sources** — MCP servers, databases, APIs

### 3.3 Component 2: Context Manager (φ₂ + φ₄ + φ₅)

**Vai trò:** Quản lý vòng đời context — store, compress, self-bake, transfer giữa memory tiers.

**3 Memory Tiers:**

| Tier | Storage | Lifespan | Content |
|---|---|---|---|
| **Working Memory** | In-RAM | Current session only | Raw conversation turns, active tool outputs |
| **Session Memory** | SQLite | Persisted per session | Compressed history, session summaries |
| **Long-term Memory** | File (JSON/MD) + Vector DB | Cross-session | Facts, decisions, preferences, embeddings |

**Self-baking Pipeline** — khái niệm từ [Context Engineering 2.0](https://arxiv.org/abs/2510.26493) (φ₅), pattern 4 lấy cảm hứng từ [ACE](https://arxiv.org/abs/2510.04618). Đây là điểm khác biệt lớn nhất — chưa tool nào implement đầy đủ:

![Self-Baking Pipeline: 4 Consolidation Patterns](/assets/images/blog-cm-06-self-baking-pipeline.drawio.png)

**Memory Transfer Rules:**

```
Short-term → Long-term when:
  - importance(item) > θ_importance (e.g., 0.7)
  - item contains: decision, preference, architecture choice, error-solution
  - session ends → auto-bake remaining working memory
```

### 3.4 Component 3: Context Selector (φ₆)

**Vai trò:** Chọn subset context tối ưu cho từng query, fit vào token budget.

**Scoring formula** (proposed — các weights dưới đây là giá trị khởi đầu gợi ý, cần tuning theo use case cụ thể):

![Context Scoring Formula: Factor Weights](/assets/images/blog-cm-08-scoring-weights.drawio.png)

**Token Budget Allocation** (proposed — tỷ lệ gợi ý cho 200K context window, cần điều chỉnh theo model và workload):

![Token Budget Allocation](/assets/images/blog-cm-07-token-budget.drawio.png)

**KV-cache optimization** (insight từ [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)):

> *"If you had to choose just one metric, the KV-cache hit rate is the single most important metric for a production-stage AI agent."*

Giữ system prompt + long-term context ổn định ở prefix → cache hit → giảm latency + cost.

### 3.5 Component 4: Context Sharer (φ₇ + φ₈)

**Vai trò:** Chia sẻ context giữa agents và cross-system, điều chỉnh strategy dựa trên feedback.

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

## 4. Mapping Tools Vào Kiến Trúc — Dùng Gì, Xây Gì

Đây là phần thực dụng nhất: với kiến trúc 4 components + 3 tiers + 2 pipelines ở trên, tool nào có sẵn lắp vào được, và gap nào cần xây?

### 4.1 Bảng Mapping Toàn Diện

![Component Mapping: Available Tools vs Custom Build](/assets/images/blog-cm-13-component-mapping.drawio.png)

### 4.2 Tổng kết: 60% dùng tool có sẵn, 40% cần xây

**Dùng tool có sẵn (60%):**
- Memory storage: Mem0 hoặc Zep (tùy use case)
- Vector search: FAISS/Chroma (self-hosted) hoặc Qdrant/Pinecone (managed)
- NL compression: LLM-based summarization hoặc LLMLingua 2
- Observational compression: Mastra OM (nếu dùng TypeScript)
- Protocol: MCP cho cross-tool sharing
- Static context: CLAUDE.md pattern
- Session compaction: Google ADK compaction (nếu dùng ADK)

**Cần xây mới (40%) — đây là phần tạo differentiation:**

| Component Cần Xây | Tại Sao Chưa Ai Làm | Complexity |
|---|---|---|
| **Context Selector** (scoring + budgeting) | Mỗi tool chỉ lo retrieve, không ai lo tổng hợp + rank + budget | Medium |
| **Self-baking Pipeline** | Paper mới đề xuất (2025), chưa tool nào implement đúng | High |
| **Schema Extraction Engine** | Cần domain-specific schemas, khó generalize | Medium |
| **KV-cache Optimizer** | Cần hiểu LLM internals, ít người focus | Medium |
| **Adaptation Loop** | Cần feedback signal, chưa ai define rõ metric | High |
| **Unified .context/ Protocol** | CLAUDE.md là static; cần dynamic, auto-updated version | Medium |
| **Export/Import Pipeline** | Mỗi tool format riêng, cần standardize | Low |

![Build vs Buy: 60/40 Split](/assets/images/blog-cm-10-build-vs-buy.drawio.png)

---

## 5. Gợi Ý Compositions & Use Cases

### 5.1 Composition A: "Production Chatbot" — Drop-in Memory

**Use case:** Customer support bot cần nhớ user preferences, lịch sử tương tác.

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
**Cần xây thêm:** Không — Mem0 handle trọn vẹn
**Cost:** $19-249/mo managed, hoặc self-hosted (vector DB only, graph paywalled)
**Khi nào dùng:** MVP, B2B copilot, customer support — cần đi nhanh, không cần custom

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

**Use case:** Long-running agent cần tự quản lý memory, transparent reasoning, audit trail.

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

**Tools used:** Letta (self-hosted hoặc Cloud)
**Cần xây thêm:** Domain-specific memory block schemas
**Cost:** Free (self-hosted, single PostgreSQL) hoặc $20/mo (Cloud)
**Khi nào dùng:** AI researcher assistant, document analysis agent, autonomous coding agent

---

### 5.3 Composition C: "Long-Running Coding Agent" — Memory + Compression

**Use case:** Coding agent chạy hàng giờ, context window overflow, cần nhớ decisions across sessions.

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
**Cần xây thêm:** Context Selector (scoring + budgeting khi combine 3 nguồn)
**Cost:** Supermemory $19/mo + Mastra (self-hosted, free)
**Khi nào dùng:** Long coding sessions, multi-session refactoring projects

---

### 5.4 Composition D: "Enterprise Platform" — Full Context Management

**Use case:** Enterprise agentic platform cần full context management — multi-agent, compliance, temporal reasoning.

![Enterprise Context Management Architecture](/assets/images/blog-cm-09-enterprise-architecture.drawio.png)

**Tools used:**
- Zep (temporal knowledge graph — $)
- LLMLingua 2 (token compression — free)
- MCP SDK (protocol — free)
- FAISS/Chroma (vector search — free)

**Cần xây:**
- Context Selector (scoring + budgeting + KV-cache optimization)
- Self-baking Pipeline (4 patterns)
- Schema Extraction Engine
- .context/ file protocol
- Adaptation Loop
- Export/Import pipeline

**Cost:** Zep Cloud (custom pricing) + self-hosted infra + engineering team
**Khi nào dùng:** Enterprise platform serving multiple agent types, compliance-heavy, temporal reasoning needed

---

### 5.5 Composition E: "Lightweight CLI Agent" — File-First, Zero Dependencies

**Use case:** CLI agent (kiểu Claude Code) cần persistent memory across sessions, zero external services.

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
**Cần xây:** Entire lightweight CML library
**Cost:** $0 infrastructure (chỉ LLM API cost)
**Khi nào dùng:** CLI tools, developer workflows, offline-capable agents

Đây là composition phù hợp nhất cho project context-engineering trong repo này — một lightweight, file-first context management layer tương thích với CLAUDE.md pattern, không phụ thuộc external services.

---

## 6. Decision Framework — Chọn Composition Nào?

![Decision Flowchart: Chọn Composition Nào?](/assets/images/blog-cm-11-decision-flowchart.drawio.png)

---

## 7. Những Gì Sẽ Thay Đổi (2026-2027)

### Trends đang định hình landscape

1. **Memory as infrastructure, not feature.** Giống như database đã trở thành infrastructure layer, AI memory đang trải qua quá trình tương tự. 2026 là năm memory chuyển từ experimental sang essential.

2. **Convergence of memory types.** Production systems đang kết hợp nhiều loại memory thay vì chọn một: vector + graph + temporal + observational. Ranh giới giữa các tools đang mờ dần.

3. **Self-baking sẽ trở thành standard.** Khi paper [Context Engineering 2.0](https://arxiv.org/abs/2510.26493) lan rộng, việc agent tự "tiêu hóa" context thô thành knowledge có cấu trúc sẽ trở thành practice tiêu chuẩn — tương tự cách RAG đã trở thành standard sau 2023.

4. **Context is the new data engineering.** Như Andrej Karpathy đã nói, context engineering không phải prompt engineering — đó là software architecture cho intelligence. Các team sẽ cần "context engineer" như họ cần data engineer.

5. **MCP sẽ chuẩn hóa sharing layer.** Với việc OpenAI, Google DeepMind cùng adopt MCP, đây sẽ trở thành protocol chuẩn cho agent-to-tool và agent-to-agent communication — bao gồm cả memory sharing.

![Context Management: Evolution Timeline](/assets/images/blog-cm-12-trends-timeline.drawio.png)

---

## Kết luận

Quay lại câu hỏi ban đầu: **một kiến trúc context management hoàn chỉnh gồm những gì?**

**4 components:**
1. **Collector** — Thu thập context từ mọi nguồn, normalize thành format thống nhất
2. **Manager** — Lưu trữ 3-tier (working/session/long-term), compression, self-baking
3. **Selector** — Scoring, ranking, token budgeting, KV-cache optimization
4. **Sharer** — Cross-agent sharing (MCP), export/import, adaptation loop

**3 pipelines:**
1. **Compression Pipeline** — NL summary + token-level + schema extraction + vector
2. **Self-baking Pipeline** — Raw → Summarized → Structured → Abstract (with backlinks)
3. **Adaptation Pipeline** — Track quality → adjust weights → improve over time

**60% dùng tools có sẵn, 40% cần xây mới.** Phần cần xây — Selector, Self-baking, Adaptation — chính là phần tạo nên sự khác biệt. Đó là lý do chưa có giải pháp "all-in-one": vì phần khó nhất chưa ai giải quyết.

Memory management trả lời *"nhớ gì và quên gì"*. Context management trả lời *"nhớ gì, quên gì, VÀ đưa gì vào prompt lúc này, sắp xếp thế nào, nén ra sao, chia sẻ với ai, và điều chỉnh thế nào dựa trên kết quả"*.

Memory là bộ nhớ. Context management là bộ não.

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
