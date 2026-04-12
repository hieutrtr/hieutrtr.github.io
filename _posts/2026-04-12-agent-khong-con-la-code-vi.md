---
layout: post
title: "Agent Không Còn Là Code — Kỷ Nguyên Context Harness Engineering"
description: "Deep Agents Deploy và Anthropic Managed Agents cho thấy thay đổi căn bản: build agent không còn là viết code. Mà là định nghĩa identity, skills, và context."
featured: true
lang: vi
ref: agents-are-not-code-anymore
permalink: /vi/agent-khong-con-la-code/
date: 2026-04-12
---

# Agent Không Còn Là Code — Kỷ Nguyên Context Harness Engineering

> *Thứ quan trọng nhất trong một agent không còn là code bạn viết. Mà là context bạn định nghĩa.*

---

Đầu tháng 4/2026, hai release cách nhau vài tuần trông có vẻ là shipping announcement bình thường — nhưng thực ra là tín hiệu rằng định nghĩa "AI agent" đang thay đổi căn bản.

Ngày 8 tháng 4, Anthropic launch [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) public beta. Vài tuần trước đó, LangChain release [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/) — một open-source alternative. Hai hệ thống này được build bởi hai team khác nhau, với philosophy khác nhau — nhưng lại converge về gần như cùng một architectural conclusions.

Đó không phải trùng hợp. Đó là signal.

---

## Model cũ: Agent là Code

Cho đến gần đây, "build một agent" có nghĩa là viết *rất nhiều* code. Không chỉ là system prompt — mà là application logic thực sự.

Bạn define tools dưới dạng decorated Python functions. Wire up memory backends (Redis, Postgres, hoặc vector store). Viết routing logic: khi nào call tool, khi nào respond, khi nào hand off cho subagent. Manage state across turns. Handle retries. Và tất nhiên, viết system prompt để tie everything together.

Kết quả là một agent bị *coupled chặt* với framework. LangGraph agent trông hoàn toàn khác CrewAI agent hay raw Claude API agent. Migrate giữa chúng = rewrite hầu hết code. Identity của agent — instructions, capabilities, constraints — nằm rải rác trong function definitions, prompt strings, và config dictionaries.

Không phải cách này sai. Đó chỉ là cách duy nhất chúng ta biết làm.

---

## Model mới: Agent là Context

Managed Agents và Deep Agents Deploy flip điều này. Primitive mới không phải code — mà là **context**.

**LangChain's Deep Agents** xoay quanh hai open standards:

- **`AGENTS.md`** — identity của agent: nó là ai, hành xử thế nào, biết gì. Plain markdown.
- **Skills** (`SKILL.md` files) — specialized knowledge modules được load on demand, giảm token usage trong khi mở rộng capability.

Chạy `deepagents deploy`, framework handle memory (filesystem-backed, per-session), sandboxing (Daytona, Runloop, Modal), và production server với MCP + A2A + Agent Protocol — tất cả từ những definition files đó.

**Anthropic's Managed Agents** đi xa hơn ở infrastructure level với architecture 3 thành phần:

- **Brain** — Claude cộng với controller logic. Stateless, replaceable.
- **Hands** — disposable ephemeral container để tools chạy trong đó. Zero access vào credentials.
- **Session** — append-only event log, chính là external memory của agent. Nếu Brain crash, instance mới pick up từ event cuối.

Pricing model ($0.08/session-hour + token costs) phản ánh điều này: bạn đang trả cho *runtime*, không phải infrastructure bạn phải tự manage.

Cả hai share cùng một insight: **identity và capability của agent tách biệt với execution infrastructure**.

---

## Context Harness: Identity + Skills + Context

"Context harness" là cái tên đúng cho những gì các platform này cung cấp.

Một agent build trên context harness được định nghĩa bởi ba thứ:

1. **Identity**: nó là ai, suy nghĩ thế nào, constraints là gì — `AGENTS.md`
2. **Skills**: specialized knowledge nào nó có thể load on demand — `SKILL.md` files
3. **Session**: nó nhớ gì về cuộc conversation này — được platform manage

Viết ba thứ đó tốt, platform biến chúng thành production deployment. Execution, memory, security, multi-protocol interoperability — tất cả được handle ở infrastructure layer.

Điều này tạo ra một sự dịch chuyển thú vị: công việc quan trọng nhất chuyển từ *"làm sao implement behavior này trong code"* sang *"làm sao express identity và capabilities của agent này thật chính xác."* Gần với technical writing và context engineering hơn là software architecture.

---

## Tại sao shift này xảy ra: Security là forcing function

Không phải chỉ vì philosophical — mà vì security thực tế.

Trong coupled design (model cũ), bất kỳ code nào Claude generate đều chạy trong cùng container với credentials của nó. Một prompt injection thành công không chỉ compromise một task — nó cho attacker access vào *mọi thứ* agent đó có thể làm.

Brain/Hands separation của Managed Agents phá vỡ điều này: sandbox nơi generated code chạy không có access vào authentication tokens. Compromise sandbox = không thu được gì hữu ích. Deep Agents Deploy enforce isolation tương tự thông qua sandbox provider abstraction.

Loại isolation này từng có thể làm trước đây — nhưng bạn phải tự architect, và hầu hết team không làm vì nó khó. Context harness platforms biến nó thành mặc định.

---

## Harness Engineering: Discipline đằng sau nó

Trong khi industry đang gọi thứ này là "context harness," hai paper quan trọng đã đặt tên chính xác hơn cho nó.

Đầu 2026, OpenAI publish một memo về cách họ build Codex — coding agent tạo ra hơn 1 triệu dòng code mà không có dòng nào do người viết. Finding đáng ngạc nhiên không phải model mạnh đến đâu. Mà là: **thứ tạo ra thành công không phải model — là hệ thống bao quanh model.** Họ gọi hệ thống đó là **harness**.

Công thức: `Agent = Model + Harness`

Harness là mọi thứ ngoài model: tools nó có thể gọi, architectural constraints ngăn nó phá vỡ invariants, feedback loops giúp tự sửa lỗi, observability layer để engineer giám sát. Đây là lý do LangChain tăng từ 52.8% lên 66.5% trên Terminal Bench 2.0 — top 30 lên top 5 — mà không thay đổi model. Họ chỉ design harness tốt hơn.

Martin Fowler formalize điều này vào tháng 4/2026, định nghĩa harness gồm ba thành phần:

- **Context engineering**: agent biết gì, khi nào, với cấu trúc gì?
- **Architectural constraints**: agent *không thể* làm gì? Guardrails, dependency rules, boundaries.
- **Garbage collection**: cleanup sau khi agent chạy xong — state management, cost accounting, session lifecycle.

**Harness engineering** là discipline thiết kế ba thành phần này đúng cách trên toàn bộ agent system.

Phân biệt quan trọng: context harness là *platform layer* — thứ developer deploy lên. Harness engineering là *discipline* — cách thiết kế nó đúng. Nói cách khác: harness engineering là kỹ năng. Context harness là kết quả.

---

## AgentOS: Tên research cho cùng một thứ

Trong khi engineering community gọi "harness," research community gọi cùng thứ đó là **AgentOS** — và đang build nó nghiêm túc hơn ta nghĩ.

**AIOS** (Rutgers University, COLM 2025) là implementation đầu tiên được peer-reviewed. Architecture của nó đặt tên chính xác cho những gì production platforms đang làm ngầm:

- **Agent Scheduler**: schedule agent requests để optimize LLM utilization
- **Context Manager**: snapshot/restore trạng thái generation, manage context windows
- **Memory Manager**: short-term memory cho mỗi agent interaction
- **Storage Manager**: persistent storage dài hạn
- **Tool Manager**: quản lý tool execution
- **Access Manager**: access control cho resources

Nhìn lại Managed Agents: Brain ≈ Agent Scheduler + Context Manager. Session ≈ Memory Manager + Storage Manager. Hands ≈ Tool Manager + Access Manager.

Đây không phải trùng hợp. Đây là convergence — hai nhóm khác nhau (research và industry) đang giải cùng một bộ vấn đề, và đến cùng một architectural conclusions. AIOS đo được 2.1x faster execution khi serve multiple agents so với naive approach — chứng minh AgentOS primitives có real performance implications, không chỉ là concept đẹp.

**AgenticOS 2026 Workshop** (co-located với ASPLOS 2026) là signal rõ nhất academia đang seriously hóa vấn đề này: họ tìm kiếm papers về OS-level mechanisms cho AI-agent workloads, isolation models, scheduling techniques. Cùng level systems research mà người ta từng nghiên cứu virtual memory hay process scheduling.

---

## Ba concept, một stack

Ghép lại và bạn thấy một layered stack rõ ràng:

```
┌─────────────────────────────────────────┐
│  Agent Definition Layer                 │  ← AGENTS.md + SKILL.md
│  (developer viết gì)                   │     identity, skills, context
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

**Harness Engineering** là discipline chạy xuyên suốt toàn stack.

Analogy với computing quen thuộc:

- AgentOS layer ↔ OS kernel (process isolation, scheduling, resource management)
- Context Harness layer ↔ container runtime (Docker) — opinionated packaging của OS primitives
- Agent Definition layer ↔ application — developer viết logic, không viết syscalls
- Control Plane layer ↔ bare-metal hypervisor — full power, full responsibility

Deep Agents Deploy thực chất là Docker của agent world. Nó không provide primitives (đó là AgentOS), không chứa business logic (đó là AGENTS.md). Nó là packaging layer — nhận agent definition, wrap trong opinionated infrastructure, deliver deployable unit.

---

## Câu hỏi thực tế: Build ở layer nào?

Đây là câu hỏi thay thế "LangGraph hay CrewAI?"

**Build ở Control Plane (LangGraph / Claude SDK) khi:**
- Routing logic của bạn thực sự novel hoặc domain-specific
- Cần kiểm soát chặt state transitions (financial workflows, auditability requirements)
- Bạn đang build infrastructure mà người khác sẽ deploy agents lên trên
- Bạn đang research agent architectures

**Build ở Context Harness layer khi:**
- Deploy agent cho mục đích cụ thể, well-defined
- Security và credential isolation là requirement, không phải nice-to-have
- Cần multi-protocol support (MCP + A2A) không có custom integration work
- Bottleneck của team là deployment và ops, không phải agent reasoning design
- Bạn muốn agent definition đọc được bởi non-engineers

**Case thú vị nhất** là hybrid: custom execution logic build trên LangGraph, fronted bởi harness-compatible deployment format. Deep Agents Deploy đã support điều này — `deepagents deploy` có thể front một custom LangGraph backend trong khi vẫn provide standard endpoints và memory management.

---

## Những thứ không thay đổi

Trước khi kết, một vài thứ vẫn giữ nguyên dù bạn build ở layer nào:

**Model quality vẫn dominant.** Harness thiết kế tốt trên model yếu vẫn thua harness thiết kế kém trên model mạnh. Brain vẫn là Claude.

**Task decomposition vẫn quan trọng.** Không platform nào magically handle ambiguous, underspecified tasks. Agent identity vẫn cần clear về "done" trông như thế nào.

**Evaluation vẫn là việc của bạn.** Managed Agents produce session logs. Deep Agents tích hợp LangSmith. Nhưng quyết định agent có làm *đúng* không — đó vẫn là trách nhiệm của bạn.

---

## Takeaway

Câu hỏi đã thay đổi. Từ "framework nào?" sang "layer nào?"

Agent definition đang chuyển từ:
> *"Viết code implement behavior này"*

sang:
> *"Define identity và skills; để platform handle execution, memory, và deployment"*

Low-level frameworks không biến mất — chúng đang trở thành substrate mà platforms chạy lên trên, cách TCP/IP không biến mất khi HTTP xuất hiện. Nhưng với hầu hết production deployments, harness là sản phẩm.

AgentOS là tầng primitives đang được định nghĩa bởi cả academia lẫn industry. Context Harness là opinionated packaging của những primitives đó. Harness Engineering là discipline để build và maintain stack đó đúng cách.

Developer build agents năm 2026 cần hiểu cả ba: primitives để hiểu *tại sao* platform hoạt động như nó hoạt động, context harness để ship nhanh trong hầu hết trường hợp, và harness engineering để không bị ngạc nhiên khi platform defaults không đủ.

Đó là division of labor mà vài năm tới của production agent work sẽ chạy trên.

---

*Nguồn:*

- [Scaling Managed Agents: Decoupling the brain from the hands — Anthropic Engineering](https://www.anthropic.com/engineering/managed-agents)
- [Deep Agents Deploy: an open alternative to Claude Managed Agents — LangChain Blog](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/)
- [Harness engineering for coding agent users — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [Harness Engineering: first thoughts — Martin Fowler](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering-memo.html)
- [Harness Engineering — OpenAI](https://openai.com/index/harness-engineering/)
- [AIOS: LLM Agent Operating System — arXiv](https://arxiv.org/abs/2403.16971)
- [AgenticOS 2026 Workshop — os-for-agent.github.io](https://os-for-agent.github.io/)
- [2025 Was Agents. 2026 Is Agent Harnesses — Aakash Gupta, Medium](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e)
