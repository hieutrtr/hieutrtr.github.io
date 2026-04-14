---
layout: post
title: "Agent Stack 2026: Layers, Harness, và Cách Bạn Build"
description: "Hai nền tảng mới định nghĩa lại cách build và deploy agent — nhưng trước khi kết luận, đáng để xem một agent đang chạy thật có bao nhiêu layers, và nghiên cứu của chính Anthropic nói gì về độ phức tạp của harness."
featured: true
lang: vi
ref: agents-are-not-code-anymore
permalink: /vi/agent-khong-con-la-code/
date: 2026-04-12
---

# Agent Stack 2026: Layers, Harness, và Cách Bạn Build

---

Đầu tháng 4/2026, hai bản phát hành xuất hiện cách nhau vài tuần và kéo theo nhiều ý kiến mạnh: [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) của Anthropic (bản thử nghiệm công khai, ngày 8/4) và [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/) của LangChain. Một số người gọi đây là "sự thay đổi căn bản" trong phát triển agent. Số khác thì hoài nghi hơn.

Trước khi tranh luận theo hướng nào, đáng để lùi lại và hỏi một câu đơn giản hơn: **một agent AI đang chạy thật hiện nay có bao nhiêu layers?** Và hai nền tảng mới này đang nằm ở đâu trong cái stack đó?

Bài này sẽ vẽ lại bản đồ các layers đó — bao gồm cả những gì nghiên cứu của chính Anthropic nói về độ phức tạp kiến trúc — và cố gắng phân tích thực tế cái gì đã thay đổi và cái gì thì không.

---

## Framework Của Chính Anthropic: Ba Levels Phức Tạp

Trước khi phát hành Managed Agents, Anthropic đã công bố ["Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) — một hướng dẫn thực hành mô tả rõ ràng sự tiến triển của độ phức tạp kiến trúc:

**Level 1: Augmented LLM**
Nền tảng. Một LLM duy nhất được bổ sung khả năng truy xuất dữ liệu, công cụ, và memory. Hầu hết agent đang hoạt động tốt đều ở đây — một model được hướng dẫn đúng với bộ tool phù hợp thường là đủ, và khó mà đơn giản hơn được.

**Level 2: Predefined Workflows**
Các processing pipeline được định nghĩa trước, điều phối LLM và công cụ. Anthropic mô tả năm pattern:
- *Prompt chaining* — gọi lần lượt, mỗi bước xử lý kết quả của bước trước
- *Routing* — phân loại đầu vào và chuyển đến bộ xử lý chuyên biệt
- *Parallelization* — chia các tác vụ con độc lập hoặc chạy nhiều model để vote
- *Orchestrator-workers* — một LLM trung tâm phân công linh hoạt cho các thành phần con
- *Evaluator-optimizer* — loop cải tiến giữa generator và evaluator

**Level 3: Agent tự chủ**
Hệ thống mà LLM tự quyết định quy trình và cách dùng tool của chính mình. LLM tự quyết định bước tiếp theo, không phải theo pipeline định sẵn.

Nguyên tắc cốt lõi Anthropic nhấn mạnh: **bắt đầu ở level thấp nhất còn hoạt động được và chỉ thêm phức tạp khi hiệu quả cải thiện đo được.** Nguyên tắc này cần nhớ khi đánh giá những platform mới mặc định chọn kiến trúc "agent đầy đủ".

---

## Stack Hiện Tại: Một Spectrum Từ Thấp Đến Cao

*(Phần dưới là một mô hình tư duy để phân tích hệ sinh thái — không phải phân loại chính thức hay tiêu chuẩn ngành đã được xác nhận. Các "layer" này không có ranh giới rõ ràng; chúng là các vị trí trên một spectrum từ "tự viết nhiều" đến "chỉ cần cấu hình".)*

Hãy nghĩ như một núm vặn: từ "viết mọi thứ bằng code" đến "chỉ mô tả agent là gì":

**Thấp nhất — Tự viết toàn bộ orchestration**

Bạn gọi LLM API trực tiếp, tự nối tool execution, tự quản lý state, tự xử lý retry. Mọi quyết định về routing, memory, error handling đều nằm trong code bạn viết và bạn chịu trách nhiệm.

*Công cụ: LangGraph, Claude Agent SDK, API gốc của Anthropic/OpenAI*

*Ví dụ: Bạn tự viết đồ thị LangGraph bằng Python — các nút cho từng bước, các cạnh cho routing, quản lý state tường minh. Toàn quyền kiểm soát, toàn bộ trách nhiệm.*

**Giữa — Framework lo phần infrastructure**

Framework cung cấp các abstraction layer (agent, chain, memory) và quản lý loop gọi LLM. Bạn tập trung vào cấu hình tool và logic của agent, không phải raw API. Vẫn nặng code, nhưng có guardrail.

*Ví dụ: LangChain agents với memory tích hợp và abstraction layer cho tool. Bạn viết chain và tool; framework lo orchestration.*

**Cao hơn — Platform lo deploy và vận hành**

Bạn mô tả *agent là ai* — vai trò, kỹ năng, kiến thức — trong các file có cấu trúc. Platform tự lo deploy, memory, sandbox, credential isolation, và multi-protocol support. Thứ từng mất một tuần để dựng nay chỉ là một lệnh.

*Ví dụ (Deep Agents Deploy): Bạn viết `AGENTS.md` (danh tính và hành vi bằng markdown thuần) và các file `SKILL.md` (kiến thức chuyên môn nạp theo yêu cầu). Chạy `deepagents deploy`. Platform lo toàn bộ memory, sandbox, và server MCP + A2A + Agent Protocol.*

*Ví dụ (Claude Managed Agents): Bạn cấu hình agent làm gì. Infrastructure của Anthropic tách Brain (Claude + logic điều khiển) khỏi Hands (môi trường thực thi tạm thời, không truy cập được credential). Một append-only session log đóng vai trò persistent external memory — nếu Brain gặp sự cố giữa chừng, instance mới tiếp tục từ event cuối.*

**Cao nhất — Chỉ cần system prompt, không cần infrastructure**

Model được hướng dẫn tốt với bộ tool phù hợp. Không custom orchestration, không deployment pipeline — chỉ system prompt và danh sách tool.

*Ví dụ: Một phiên Claude với search và code interpreter. Được hướng dẫn kỹ, phạm vi rõ ràng, dễ debug.*

---

Hai bản phát hành tháng 4 — Managed Agents và Deep Agents Deploy — nằm ở vùng "cao hơn" trên spectrum này. Chúng không loại bỏ code hoàn toàn. Nhưng chúng kéo một phần lớn công việc (memory management, credential isolation, multi-protocol support, sandbox) ra khỏi application code và đưa vào platform infrastructure.

Trước 2026, build tương đương "sandbox có credential isolation + append-only session log + MCP/A2A/Agent Protocol server" sẽ mất khoảng một tuần dựng infrastructure. Các platform này biến điều đó thành mặc định.

---

## Managed Agents và Deep Agents Deploy Thực Sự Cung Cấp Gì

Cả hai platform đồng thuận về một kết luận kiến trúc: **danh tính và năng lực của agent phải tách biệt khỏi execution infrastructure**.

**Deep Agents Deploy của LangChain** hiện thực hóa điều này qua các chuẩn mở:
- `AGENTS.md` — agent là ai, hành xử thế nào, biết gì. Markdown thuần.
- Các file `SKILL.md` — kiến thức chuyên môn dạng module, nạp theo yêu cầu, giảm token tiêu thụ trong khi mở rộng năng lực.

Chạy `deepagents deploy`, framework xử lý memory (lưu trên filesystem, theo từng session), sandbox (Daytona, Runloop, Modal), và production server với MCP + A2A + Agent Protocol.

**Managed Agents của Anthropic** đi xa hơn ở tầng infrastructure:
- **Brain** — Claude cộng với logic điều khiển. Stateless, thay thế được.
- **Hands** — container tạm thời dùng một lần để tool chạy. Không truy cập được credential.
- **Session** — append-only event log, chính là external memory của agent. Nếu Brain gặp sự cố, instance mới tiếp tục từ event cuối.

Giá: $0.08/giờ-session cộng chi phí token. Bạn đang trả cho thời gian chạy, không phải infrastructure.

Cả hai platform đều thật và kiến trúc đều nhất quán. Câu hỏi là liệu chúng có phải tool đúng cho một bài toán cụ thể hay không — và câu trả lời trung thực phụ thuộc vào cái bạn đang build.

---

## Anthropic Về Độ Phức Tạp Của Harness: Đừng Over-engineer

Song song với bản phát hành Managed Agents, Anthropic Engineering blog đăng ["Harness Design for Long-Running Application Development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — mô tả chi tiết cách họ build một harness ba agent để sinh ứng dụng hoàn chỉnh:

- **Planner**: Chuyển chỉ dẫn ngắn của người dùng thành đặc tả sản phẩm chi tiết
- **Executor**: Implement tính năng từng bước (React, Vite, FastAPI, SQLite)
- **Evaluator**: Kiểm thử qua Playwright, chấm theo tiêu chí, cung cấp phản hồi cho loop tiếp theo

Loop Plan/Execute/Evaluate cho kết quả tốt — nhưng phát hiện thú vị hơn là những gì xảy ra khi model nền cải thiện từ Claude Opus 4.5 lên 4.6.

> *"Mỗi thành phần trong một harness đều mã hóa một giả định về những gì model không thể tự làm, và những giả định đó đáng được kiểm tra lại."*

Khi Claude 4.6 xử lý tốt hơn với các tác vụ dài liền mạch, Anthropic đã đơn giản hóa harness: bỏ phân chia theo giai đoạn ngắn, chuyển đánh giá từ per-phase sang end-of-run. Orchestration trở nên ít phức tạp hơn, không phải nhiều hơn — vì model đã có thể xử lý nhiều hơn trực tiếp.

Nguyên tắc thứ hai: *"tìm giải pháp đơn giản nhất có thể, và chỉ tăng phức tạp khi cần thiết."*

Điều này cắt cả hai chiều. Đây là cảnh báo chống việc vội vàng dùng harness cũng như chống việc tự over-engineer orchestration. Nếu bài toán phù hợp với một agent đơn được hướng dẫn tốt (Level 1), thêm Workflow patterns hay harness platform đầy đủ thường là chi phí phát sinh, không phải giá trị.

---

## Những Đánh Đổi Thực Sự

**Các harness platform làm dễ dàng hơn những gì:**
- Credential isolation và sandbox security mặc định — không cần tự thiết kế
- Multi-protocol support (MCP, A2A, Agent Protocol) không cần tự tích hợp
- Deploy một agent chạy được trong một ngày thay vì một tuần dựng infrastructure
- File định nghĩa agent mà người không viết code cũng đọc và sửa được
- Memory management mà không cần chọn và cấu hình vector store

**Những gì chúng làm khó hơn hoặc đưa vào như mối quan tâm mới:**
- Custom routing logic không khớp với execution model của platform
- Kiểm soát chi tiết thời điểm và cách memory được ghi hoặc đọc
- Debug: append-only session log tốt cho recovery, kém trực quan hơn khi inspect
- Vendor lock-in — Managed Agents là infrastructure chỉ dành cho Claude
- Dự đoán chi phí ở quy mô lớn: $0.08/giờ-session rẻ cho đến khi bạn chạy nhiều long-lived agent
- Câu hỏi mở: các default của platform có khớp với nhu cầu thực của agent bạn không

**Deep Agents Deploy** xử lý vấn đề portability tốt hơn Managed Agents: chạy trên bất kỳ model nào (OpenAI, Anthropic, Google, Ollama), deploy self-hosted hoặc cloud, và định dạng `AGENTS.md`/`SKILL.md` được cấp phép MIT. Chi phí chuyển đổi thấp. Managed Agents cung cấp tích hợp infrastructure chặt hơn với giá là vendor lock-in.

---

## Tại Sao Bảo Mật Là Động Lực Thực Sự

Một lý do cụ thể sự dịch chuyển này xảy ra không phải mang tính triết lý — mà là bảo mật thiết thực.

Trong thiết kế gắn chặt, code mà agent sinh ra hoặc thực thi chạy trong cùng process space với credential. Một cuộc tấn công prompt injection thành công trong kịch bản đó không chỉ compromise một task — mà có thể phơi bày mọi thứ agent đó có quyền truy cập.

Credential isolation giải quyết điều này trực tiếp. Đây không phải khái niệm mới: service mesh, secret manager như Vault, per-service IAM role đã áp dụng nguyên tắc này từ lâu. Pattern rất rõ ràng — giới hạn những gì một execution context cụ thể có thể tiếp cận.

Sự tách biệt Brain/Hands của Managed Agents áp dụng nguyên tắc này vào kiến trúc agent: sandbox nơi code sinh ra chạy không truy cập được authentication token. Deep Agents Deploy thực thi sự cách ly tương tự thông qua environment provider abstraction layer.

Bạn hoàn toàn có thể đạt mức isolation tương tự khi tự build harness — pattern thuộc về kiến trúc, không phải độc quyền. Các team đã quan tâm đến security trong infrastructure của mình thường đã làm điều này rồi. Sự khác biệt nằm ở cách đóng gói: các harness platform biến credential isolation thành mặc định thay vì một quyết định thiết kế cần chủ động thực hiện. Đó là lợi ích tiện lợi thực sự, không phải tuyên bố rằng hệ thống tự build kém secure hơn.

Trade-off trung thực hơn: tự build harness thì có toàn quyền kiểm soát security model, đổi lại là công sức thiết kế và duy trì nó. Dùng harness platform thì có một bộ default đã được kiểm thử, đổi lại là phụ thuộc vào các security assumption của platform. Không cái nào tốt hơn tuyệt đối — chúng là những lựa chọn khác nhau về việc đầu tư công sức vào đâu.

---

## OpenClaw: Khi Định Nghĩa Agent Ở Context Layer Không Đi Kèm Infrastructure Bảo Mật

Cả Managed Agents lẫn Deep Agents Deploy đều chia sẻ một concept cốt lõi: định nghĩa agent ở context layer — danh tính, kỹ năng, kiến thức trong các file có cấu trúc, không phải trong code. OpenClaw, một agent framework do cộng đồng phát triển, đã đến cùng kết luận kiến trúc này theo con đường riêng.

OpenClaw định nghĩa agent theo cùng cách: một agent là danh tính cộng với tập kỹ năng tải từ ClawHub — kho kỹ năng cộng đồng của nó. Concept ánh xạ trực tiếp vào `AGENTS.md` + `SKILL.md` hay cấu hình Brain của Managed Agents. Cùng ý tưởng, khác execution context.

Điểm khác biệt nằm ở những gì đi kèm — hoặc chính xác hơn, những gì không đi kèm.

Tháng 2/2026, một cuộc tấn công supply chain nhắm vào ClawHub (được gọi là "ClawHavoc") được tiết lộ. Kẻ tấn công đã compromise 12 tài khoản publisher và đăng tải 1.184 skill độc hại. Một khi được nạp vào context của agent, các skill này có thể leak credential, redirect tool call, hoặc nhiễm độc reasoning của agent. Cuộc tấn công đã hoạt động trong một khoảng thời gian chưa xác định trước khi bị phát hiện.

Nghiên cứu bảo mật từ Snyk củng cố thêm quy mô của vấn đề: báo cáo ToxicSkills phát hiện 36,8% skill trên ClawHub có lỗ hổng ở dạng nào đó, với 13,4% được đánh giá ở mức critical.

Đây không phải là phê phán concept kiến trúc của OpenClaw — ý tưởng định nghĩa agent ở context layer là đúng đắn. Đây là bằng chứng về những gì concept đó cần để khả thi trong môi trường production: một trust model cho skill supply chain, credential isolation để skill được nạp không thể truy cập authentication token, và audit infrastructure ghi lại skill nào đã chạy và khi nào.

So sánh giữa ba cách tiếp cận:

| | Mô hình supply chain | Credential isolation | Nguồn skill |
|---|---|---|---|
| **OpenClaw** | Registry cộng đồng mở (ClawHub) | Không bắt buộc | Tải lúc runtime từ publisher chưa kiểm tra |
| **Deep Agents Deploy** | File bạn kiểm soát | Sandbox abstraction layer | File `SKILL.md` cục bộ |
| **Managed Agents** | Platform kiểm soát, đóng | Tách Brain/Hands theo thiết kế | Cấu hình trong platform |

ClawHavoc là căn cứ thực tế hữu ích cho phần bảo mật ở trên: định nghĩa agent ở context layer không phải rủi ro bảo mật tự thân. Rủi ro là nạp skill từ unverified community registry vào execution context có quyền truy cập credential. Pattern kiến trúc tách biệt với sự cố bảo mật — nhưng chỉ khi infrastructure bên dưới nó xử lý nghiêm túc mối đe dọa đó.

---

## AgentOS: Layer Nghiên Cứu Đang Hình Thành

Cả giới học thuật lẫn ngành công nghiệp đều đang hội tụ về cùng một bộ thành phần cơ bản, dù gọi tên khác nhau.

**AIOS** (Đại học Rutgers, COLM 2025) là hiện thực hóa có bình duyệt đầu tiên của cái thực chất là "hệ điều hành cho agent":

- Agent Scheduler, Context Manager, Memory Manager, Storage Manager, Tool Manager, Access Manager

Ánh xạ điều này vào Managed Agents: Brain ≈ Agent Scheduler + Context Manager. Session ≈ Memory Manager + Storage Manager. Hands ≈ Tool Manager + Access Manager. Sự hội tụ này không phải ngẫu nhiên — hai con đường nghiên cứu và industry riêng biệt đến cùng kết luận kiến trúc vì họ đang giải cùng bài toán quản lý tài nguyên.

AIOS cho thấy tốc độ thực thi nhanh hơn 2.1 lần khi phục vụ nhiều agent so với cách tiếp cận thông thường — các thành phần cơ bản có tác động thực sự đến performance.

**Hội thảo AgenticOS 2026** (tổ chức cùng ASPLOS 2026) cho thấy giới học thuật đang coi đây là nghiên cứu hệ thống nghiêm túc: OS-level mechanism cho AI agent workload, isolation model, scheduling technique. Cùng mức độ nghiêm ngặt đã tạo ra virtual memory và process scheduling nay được áp dụng vào agent infrastructure.

---

## Build Ở Layer Nào?

Câu hỏi này thực ra tách làm hai — và rất dễ bị nhầm lẫn với nhau:

**Câu hỏi 1: Dùng tool/platform nào?** (vị trí trên spectrum)

**Dùng low-level framework (LangGraph / Claude Agent SDK) khi:**
- Routing logic của bạn thực sự mới lạ hoặc đặc thù theo domain
- Cần kiểm soát chặt state transition — quy trình tài chính, yêu cầu audit
- Bạn đang build infrastructure mà người khác sẽ deploy agent lên trên
- Bạn đang nghiên cứu kiến trúc agent, không phải ship sản phẩm

**Dùng managed platform (Managed Agents / Deep Agents Deploy) khi:**
- Deploy agent với mục đích cụ thể, rõ ràng
- Security và credential isolation là yêu cầu bắt buộc, không phải nice-to-have
- Cần multi-protocol support mà không cần tự tích hợp
- Bottleneck của team là deployment và operations, không phải thiết kế agent reasoning
- Muốn định nghĩa agent mà người không viết code cũng đọc và sửa được

**Câu hỏi 2: Agent thực sự cần bao nhiêu orchestration?** (3 levels của Anthropic — một trục riêng biệt)

Sự tiến triển của Anthropic — Augmented LLM → Workflows → Agent tự chủ — mô tả *độ phức tạp* của agent, không phải lựa chọn platform. Một managed platform hoàn toàn có thể chạy một agent Level 1. Một custom LangGraph harness hoàn toàn có thể implement các pattern Level 3. Hai trục này độc lập với nhau.

Câu trả lời mặc định trên trục này: **bắt đầu ở Level 1 và chỉ tăng phức tạp khi có thể đo được cải thiện.** Hầu hết các bài toán tưởng như cần multi-agent workflow thực ra có thể giải bởi một model đơn được hướng dẫn tốt với bộ tool đúng. Thêm Workflow patterns Level 2 hay điều khiển tự chủ Level 3 chỉ đáng khi hiệu quả cải thiện đo được — bất kể bạn đang dùng platform nào.

**Trường hợp thú vị nhất là kết hợp:** custom execution logic trên LangGraph, phía trước là deployment format tương thích harness. Deep Agents Deploy đã hỗ trợ điều này — `deepagents deploy` có thể đặt trước một custom LangGraph backend trong khi vẫn cung cấp các standard endpoint và memory management. Bạn có opinionated deployment mà không mất quyền kiểm soát orchestration.

---

## Những Thứ Không Thay Đổi

Vài điều vẫn đúng bất kể bạn build ở layer nào:

**Chất lượng model vẫn chi phối.** Brain vẫn là Claude. Harness thiết kế tốt trên model yếu vẫn thua harness thiết kế kém trên model mạnh. Harness nhân lên những gì model có thể làm — không thay thế những gì model không biết.

**Task decomposition vẫn quan trọng.** Không platform nào tự động xử lý được task mơ hồ hoặc thiếu thông tin. Instruction cho agent vẫn cần rõ ràng về "hoàn thành" trông như thế nào. `AGENTS.md` chỉ hữu ích như cái suy nghĩ đã đi vào việc viết nó.

**Evaluation vẫn là việc của bạn.** Managed Agents sinh session log. Deep Agents tích hợp LangSmith. Nhưng quyết định agent có làm đúng không — và chẩn đoán các lỗi im lặng — vẫn là trách nhiệm của bạn.

**Độ phức tạp harness nên theo sát năng lực model.** Như công trình kỹ thuật harness của chính Anthropic cho thấy: khi model cải thiện, phản ứng đúng thường là đơn giản hóa harness, không phải giữ nguyên các ràng buộc cũ.

---

## Bức Tranh Lớn Hơn

Hệ sinh thái agent đang phân tầng thành các layer riêng biệt — đây là dấu hiệu trưởng thành, giống cách web infrastructure phân tầng từ raw TCP thành HTTP, rồi web framework, rồi deployment platform.

Low-level framework (LangGraph, Claude Agent SDK) không bị thay thế. Chúng đang trở thành nền tảng mà các harness platform chạy lên trên — cách TCP/IP không biến mất khi HTTP xuất hiện. Nhưng với hầu hết product deployment, câu hỏi ngày càng là: bài toán của tôi thuộc layer nào trong stack này?

Managed Agents và Deep Agents Deploy là các platform thật với giá trị thật cho các use case cụ thể. Chúng cũng còn mới, có vendor lock-in theo những cách khác nhau, và thêm ràng buộc mà một số workload sẽ không muốn. Đó là trade-off bình thường, không phải phát hiện mang tính cách mạng.

Bài học bền vững hơn từ cả hai bản phát hành, và từ nghiên cứu kỹ thuật harness của Anthropic, là về kiến trúc: **danh tính và năng lực của agent thuộc về một definition layer di động, dễ đọc; execution, memory, security, và deployment machinery thuộc về infrastructure bên dưới nó.** Dù bạn đến đó qua managed platform hay tự build, sự phân tách đó là đáng hướng tới.

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
