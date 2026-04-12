---
layout: post
title: "Agent Stack 2026: Các Tầng, Harness, và Nơi Bạn Thực Sự Build"
description: "Hai platform mới định nghĩa lại cách deploy agent — nhưng trước khi kết luận, đáng để map xem một production agent hiện tại có bao nhiêu tầng, và Anthropic's own research nói gì về harness complexity."
featured: true
lang: vi
ref: agents-are-not-code-anymore
permalink: /vi/agent-khong-con-la-code/
date: 2026-04-12
---

# Agent Stack 2026: Các Tầng, Harness, và Nơi Bạn Thực Sự Build

---

Đầu tháng 4/2026, hai release xuất hiện cách nhau vài tuần và kéo theo nhiều ý kiến mạnh: [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) của Anthropic (public beta, ngày 8/4) và [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/) của LangChain. Một số người gọi đây là "sự thay đổi căn bản" trong agent development. Số khác thì hoài nghi hơn.

Trước khi tranh luận theo hướng nào, đáng để lùi lại và hỏi một câu đơn giản hơn: **một production AI agent hiện nay có bao nhiêu tầng thực sự?** Và hai platform mới này đang fit vào đâu trong cái stack đó?

Bài này sẽ map stack đó — bao gồm cả những gì Anthropic's own research nói về architectural complexity — và cố gắng phân tích thực tế cái gì đã thay đổi và cái gì thì không.

---

## Framework Của Chính Anthropic: Ba Mức Complexity

Trước khi release Managed Agents, Anthropic đã publish ["Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) — một practical guide mô tả rõ ràng progression của architectural complexity:

**Mức 1: Augmented LLM**
Nền tảng. Một LLM duy nhất được tăng cường với retrieval, tools, và memory. Hầu hết agents đang hoạt động tốt đều ở đây — một model được prompt đúng với tools phù hợp thường là đủ, và khó beat về mặt simplicity.

**Mức 2: Workflows**
Predefined code paths điều phối LLMs và tools. Anthropic mô tả năm patterns:
- *Prompt chaining* — sequential calls, mỗi bước xử lý output của bước trước
- *Routing* — phân loại input và chuyển đến specialized handlers
- *Parallelization* — chia independent subtasks hoặc chạy nhiều models để voting
- *Orchestrator-workers* — một LLM trung tâm dynamically delegate cho workers
- *Evaluator-optimizer* — vòng lặp iterative giữa generator và evaluator

**Mức 3: Agents**
Hệ thống mà LLMs dynamically quyết định process và tool usage của chính mình. LLM tự quyết định bước tiếp theo, không phải predefined code paths.

Nguyên tắc cốt lõi Anthropic nhấn mạnh: **bắt đầu ở mức thấp nhất còn hoạt động được và chỉ thêm complexity khi performance measurably cải thiện.** Nguyên tắc này cần nhớ khi đánh giá những platform mới mặc định chọn "full agent" architecture.

---

## Stack Hiện Tại: Một Spectrum Từ Thấp Đến Cao

*(Phần dưới là một mental model để phân tích hệ sinh thái — không phải taxonomy chính thức hay industry standard đã được confirm. Các "tầng" này không có ranh giới rõ ràng; chúng là các vị trí trên một spectrum từ "tự build nhiều" đến "chỉ cần configure".)*

Hãy nghĩ như một cái dial: từ "viết mọi thứ bằng code" đến "chỉ mô tả agent là gì":

**Thấp nhất — Tự viết toàn bộ orchestration**

Bạn gọi LLM API trực tiếp, tự wire tool execution, tự quản lý state, tự handle retry. Mọi quyết định về routing, memory, error handling đều nằm trong code bạn viết và bạn sở hữu.

*Công cụ: LangGraph, Claude Agent SDK, raw Anthropic/OpenAI API*

*Ví dụ: Bạn tự viết LangGraph graph bằng Python — nodes cho từng bước, edges cho routing, state management tường minh. Toàn quyền kiểm soát, toàn bộ trách nhiệm.*

**Giữa — Framework lo phần cơ sở hạ tầng**

Framework cung cấp các abstraction (agents, chains, memory) và quản lý vòng lặp gọi LLM. Bạn tập trung vào tool configuration và agent logic, không phải raw API. Vẫn nặng về code, nhưng có guardrails.

*Ví dụ: LangChain agents với built-in memory và tool abstractions. Bạn viết chains và tools; framework lo orchestration.*

**Cao hơn — Platform lo deployment và runtime**

Bạn mô tả *agent là ai* — vai trò, kỹ năng, kiến thức — trong các file có cấu trúc. Platform tự lo deploy, memory, sandboxing, credential isolation, và protocol support. Thứ từng mất một tuần để setup nay chỉ là một lệnh.

*Ví dụ (Deep Agents Deploy): Bạn viết `AGENTS.md` (identity và behavior bằng plain markdown) và `SKILL.md` files (domain knowledge load on demand). Chạy `deepagents deploy`. Platform lo toàn bộ memory, sandboxing, và server MCP + A2A + Agent Protocol.*

*Ví dụ (Claude Managed Agents): Bạn configure agent làm gì. Infrastructure của Anthropic tách Brain (Claude + control logic) khỏi Hands (execution sandbox không có access vào credentials). Một append-only session log đóng vai trò external memory bền vững — nếu Brain crash giữa task, instance mới tiếp tục từ event cuối.*

**Cao nhất — Chỉ cần prompt, không cần infrastructure**

Model được prompt tốt với tools phù hợp. Không custom orchestration, không deployment pipeline — chỉ system prompt và tool list.

*Ví dụ: Một Claude instance với search và code interpreter. Được prompt kỹ, phạm vi rõ ràng, dễ debug.*

---

Hai release tháng 4 — Managed Agents và Deep Agents Deploy — nằm ở vùng "cao hơn" trên spectrum này. Chúng không loại bỏ code hoàn toàn. Nhưng chúng kéo một phần lớn công việc (memory management, credential isolation, multi-protocol support, sandboxing) ra khỏi application code và đưa vào platform infrastructure.

Trước 2026, xây dựng tương đương "credential-isolated sandbox + append-only session log + MCP/A2A/Agent Protocol server" sẽ mất khoảng một tuần setup infrastructure. Các platform này biến điều đó thành mặc định.

---

## Managed Agents và Deep Agents Deploy Thực Sự Cung Cấp Gì

Cả hai platform đồng thuận về một architectural conclusion: **identity và capability của agent phải tách biệt khỏi execution infrastructure**.

**Deep Agents Deploy của LangChain** hiện thực hóa điều này qua open standards:
- `AGENTS.md` — agent là ai, hành xử thế nào, biết gì. Plain markdown.
- `SKILL.md` files — modular knowledge được load on demand, giảm token usage trong khi mở rộng capability.

Chạy `deepagents deploy`, framework handle memory (filesystem-backed, per-session), sandboxing (Daytona, Runloop, Modal), và production server với MCP + A2A + Agent Protocol support.

**Managed Agents của Anthropic** đi xa hơn ở infrastructure level:
- **Brain** — Claude cộng với controller logic. Stateless, replaceable.
- **Hands** — disposable ephemeral container để tools chạy. Zero access vào credentials.
- **Session** — append-only event log, chính là external memory của agent. Nếu Brain crash, instance mới pick up từ event cuối.

Pricing: $0.08/session-hour cộng token costs. Bạn đang trả cho runtime, không phải infrastructure.

Cả hai platform đều real và architecture đều coherent. Câu hỏi là liệu chúng có phải right tool cho một problem cụ thể hay không — và câu trả lời trung thực phụ thuộc vào cái bạn đang build.

---

## Anthropic Về Harness Complexity: Đừng Over-Engineer

Song song với Managed Agents release, Anthropic Engineering blog publish ["Harness Design for Long-Running Application Development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — mô tả chi tiết cách họ build một three-agent harness để generate full applications:

- **Planner**: Chuyển user prompts ngắn thành product specifications chi tiết
- **Generator**: Implement features incrementally (React, Vite, FastAPI, SQLite)
- **Evaluator**: Test qua Playwright, grade theo criteria, cung cấp feedback cho iteration tiếp theo

Vòng lặp Planner/Generator/Evaluator cho kết quả tốt — nhưng finding thú vị hơn là những gì xảy ra khi underlying model cải thiện từ Claude Opus 4.5 lên 4.6.

> *"Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing."*

Khi Claude 4.6 xử lý tốt hơn với longer coherent work, Anthropic simplified harness: bỏ sprint decomposition, chuyển evaluator từ per-sprint sang end-of-run assessment. Orchestration trở nên ít complex hơn, không phải nhiều hơn — vì model đã có thể handle nhiều hơn trực tiếp.

Nguyên tắc thứ hai: *"find the simplest solution possible, and only increase complexity when needed."*

Điều này cắt cả hai chiều. Đây là cảnh báo chống premature context harness adoption cũng như chống over-engineering orchestration của riêng bạn. Nếu problem fit một well-prompted single agent (Mức 1), thêm workflow patterns hay full harness platform thường là overhead, không phải value.

---

## Trade-off Thực Sự

**Context harness platforms làm dễ dàng hơn những gì:**
- Credential isolation và sandbox security by default — không cần tự architect
- Multi-protocol support (MCP, A2A, Agent Protocol) không cần custom integration work
- Deploy một functional agent trong một ngày thay vì một tuần setup infrastructure
- Agent definition files mà non-engineers có thể đọc và sửa đổi
- Memory management mà không cần chọn và configure vector store

**Những gì chúng làm khó hơn hoặc giới thiệu như new concerns:**
- Custom routing logic không fit execution model của platform
- Fine-grained control over khi nào và cách memory được write hoặc read
- Debugging: append-only session logs tốt cho recovery, ít readable hơn cho inspection
- Vendor dependency — Managed Agents là Claude-specific infrastructure
- Cost predictability ở scale: $0.08/session-hour rẻ cho đến khi bạn chạy nhiều long-lived agents
- Câu hỏi mở: platform defaults có match actual needs của agent bạn không

**Deep Agents Deploy** handle portability concern tốt hơn Managed Agents: chạy trên bất kỳ model nào (OpenAI, Anthropic, Google, Ollama), deploy self-hosted hoặc cloud, và `AGENTS.md`/`SKILL.md` format là MIT-licensed. Switching cost thấp. Managed Agents cung cấp tighter infrastructure integration với giá là lock-in.

---

## Tại Sao Security Là Driver Thực Sự

Một lý do cụ thể shift này xảy ra không phải philosophical — mà là practical security.

Trong coupled design, code mà agent generate hoặc execute chạy trong cùng process space với credentials. Một prompt injection thành công trong scenario đó không chỉ compromise một task — mà potentially expose mọi thứ agent đó có thể access.

Credential isolation giải quyết điều này trực tiếp. Đây không phải concept mới: service meshes, secret managers như Vault, per-service IAM roles đã apply nguyên tắc này từ lâu. Pattern rất well-understood — giới hạn những gì một execution context cụ thể có thể reach.

Brain/Hands separation của Managed Agents apply nguyên tắc này vào agent architecture: sandbox nơi generated code chạy không có access vào authentication tokens. Deep Agents Deploy enforce isolation tương tự thông qua sandbox provider abstraction.

Bạn hoàn toàn có thể đạt isolation tương tự khi tự build harness — pattern là architectural, không phải proprietary. Các team đã quan tâm đến security trong infra của mình thường đã làm điều này rồi. Sự khác biệt nằm ở packaging: context harness platforms biến credential isolation thành mặc định thay vì một design decision cần chủ động thực hiện. Đó là convenience benefit thực sự, không phải claim rằng self-built systems kém secure hơn.

Trade-off trung thực hơn: tự build harness thì có full control over security model, đổi lại là effort để design và maintain nó. Dùng platform harness thì có một tested default, đổi lại là phụ thuộc vào security assumptions của platform. Không cái nào tốt hơn tuyệt đối — chúng là những bet khác nhau về việc đầu tư effort vào đâu.

---

## AgentOS: Research Layer Đang Hình Thành

Cả academia lẫn industry đều converging về cùng một set of primitives, dù gọi tên khác nhau.

**AIOS** (Rutgers University, COLM 2025) là implementation peer-reviewed đầu tiên của cái thực chất là "operating system for agents":

- Agent Scheduler, Context Manager, Memory Manager, Storage Manager, Tool Manager, Access Manager

Map điều này vào Managed Agents: Brain ≈ Agent Scheduler + Context Manager. Session ≈ Memory Manager + Storage Manager. Hands ≈ Tool Manager + Access Manager. Convergence này không phải ngẫu nhiên — hai research và industry paths riêng biệt đến cùng architectural conclusions vì họ đang giải cùng underlying resource-management problems.

AIOS demonstrate 2.1x faster execution khi serve multiple agents so với naive approaches — primitives có real performance implications.

**AgenticOS 2026 Workshop** (co-located với ASPLOS 2026) signal academia đang treat đây là serious systems research: OS-level mechanisms cho AI-agent workloads, isolation models, scheduling techniques. Cùng rigor đã tạo ra virtual memory và process scheduling nay được apply vào agent infrastructure.

---

## Build Ở Layer Nào?

Với stack này, câu hỏi thay thế "framework nào?" bằng "layer nào?"

**Build ở Control Plane (LangGraph / Claude Agent SDK) khi:**
- Routing logic của bạn genuinely novel hoặc domain-specific
- Cần tight control over state transitions — financial workflows, auditability requirements
- Bạn đang build infrastructure mà người khác sẽ deploy agents lên trên
- Bạn đang research agent architectures, không phải deploy sản phẩm

**Build ở Context Harness layer (Managed Agents / Deep Agents Deploy) khi:**
- Deploy agent với mục đích cụ thể, well-defined
- Security và credential isolation là requirement, không phải nice-to-have
- Cần multi-protocol support không có custom integration work
- Bottleneck của team là deployment và operations, không phải agent reasoning design
- Muốn agent definition đọc được và sửa được bởi non-engineers

**Ở lại Mức 1 (augmented LLM) khi:**
- Task well-scoped và một single well-prompted agent với good tools có thể handle
- Thêm workflow complexity sẽ không measurably cải thiện output quality
- Bạn muốn predictable behavior và simple debugging

**Case thú vị nhất là hybrid:** custom execution logic trên LangGraph, fronted bởi harness-compatible deployment format. Deep Agents Deploy đã support điều này — `deepagents deploy` có thể front một custom LangGraph backend trong khi vẫn provide standard endpoints và memory management. Bạn có opinionated deployment mà không mất orchestration control.

---

## Những Thứ Không Thay Đổi

Vài điều vẫn đúng bất kể bạn build ở layer nào:

**Model quality vẫn dominant.** Brain vẫn là Claude. Harness thiết kế tốt trên model yếu vẫn thua harness thiết kế kém trên model mạnh. Harness nhân lên những gì model có thể làm — không thay thế những gì model không biết.

**Task decomposition vẫn quan trọng.** Không platform nào handle ambiguous hoặc underspecified tasks tự động. Instructions của agent vẫn cần clear về "done" trông như thế nào. `AGENTS.md` chỉ hữu ích như cái thinking đã đi vào việc viết nó.

**Evaluation vẫn là việc của bạn.** Managed Agents produce session logs. Deep Agents tích hợp LangSmith. Nhưng quyết định agent có làm đúng không — và chẩn đoán silent failures — vẫn là trách nhiệm của bạn.

**Harness complexity nên track model capability.** Như harness engineering work của chính Anthropic cho thấy: khi model cải thiện, phản ứng đúng thường là simplify harness, không phải giữ nguyên old constraints.

---

## Bức Tranh Lớn Hơn

Agent ecosystem đang stratify thành các distinct layers — đây là dấu hiệu của sự trưởng thành, giống cách web infrastructure stratify từ raw TCP thành HTTP, rồi web frameworks, rồi deployment platforms.

Low-level frameworks (LangGraph, Claude Agent SDK) không bị thay thế. Chúng đang trở thành substrate mà context harness platforms chạy lên trên — cách TCP/IP không biến mất khi HTTP xuất hiện. Nhưng với hầu hết production deployments, câu hỏi ngày càng là: problem của tôi thuộc layer nào trong stack này?

Managed Agents và Deep Agents Deploy là real platforms với real value cho specific use cases. Chúng cũng còn mới, vendor-specific theo những cách khác nhau, và thêm constraints mà một số workloads sẽ không muốn. Đó là trade-off bình thường, không phải revelation.

Insight bền vững hơn từ cả hai releases, và từ Anthropic's harness engineering research, là architectural: **identity và capability của agent thuộc về một portable, readable definition layer; execution, memory, security, và deployment machinery thuộc về infrastructure bên dưới nó.** Dù bạn đến đó qua managed platform hay tự build, sự phân tách đó là worth targeting.

---

*Nguồn:*

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
