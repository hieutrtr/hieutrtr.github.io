---
layout: post
title: "Agent Stack 2026: Các Tầng, Harness, và Nơi Bạn Thực Sự Xây Dựng"
description: "Hai nền tảng mới định nghĩa lại cách triển khai agent — nhưng trước khi kết luận, đáng để xem một agent đang chạy thật có bao nhiêu tầng, và nghiên cứu của chính Anthropic nói gì về độ phức tạp của harness."
featured: true
lang: vi
ref: agents-are-not-code-anymore
permalink: /vi/agent-khong-con-la-code/
date: 2026-04-12
---

# Agent Stack 2026: Các Tầng, Harness, và Nơi Bạn Thực Sự Xây Dựng

---

Đầu tháng 4/2026, hai bản phát hành xuất hiện cách nhau vài tuần và kéo theo nhiều ý kiến mạnh: [Claude Managed Agents](https://www.anthropic.com/engineering/managed-agents) của Anthropic (bản thử nghiệm công khai, ngày 8/4) và [Deep Agents Deploy](https://blog.langchain.com/deep-agents-deploy-an-open-alternative-to-claude-managed-agents/) của LangChain. Một số người gọi đây là "sự thay đổi căn bản" trong phát triển agent. Số khác thì hoài nghi hơn.

Trước khi tranh luận theo hướng nào, đáng để lùi lại và hỏi một câu đơn giản hơn: **một agent AI đang chạy thật hiện nay có bao nhiêu tầng?** Và hai nền tảng mới này đang nằm ở đâu trong cái ngăn xếp đó?

Bài này sẽ vẽ lại bản đồ các tầng đó — bao gồm cả những gì nghiên cứu của chính Anthropic nói về độ phức tạp kiến trúc — và cố gắng phân tích thực tế cái gì đã thay đổi và cái gì thì không.

---

## Khung Phân Loại Của Chính Anthropic: Ba Mức Độ Phức Tạp

Trước khi phát hành Managed Agents, Anthropic đã công bố ["Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents) — một hướng dẫn thực hành mô tả rõ ràng sự tiến triển của độ phức tạp kiến trúc:

**Mức 1: LLM tăng cường**
Nền tảng. Một LLM duy nhất được bổ sung khả năng truy xuất dữ liệu, công cụ, và bộ nhớ. Hầu hết agent đang hoạt động tốt đều ở đây — một mô hình được hướng dẫn đúng với bộ công cụ phù hợp thường là đủ, và khó mà đơn giản hơn được.

**Mức 2: Luồng công việc định sẵn**
Các đường dẫn xử lý được định nghĩa trước, điều phối LLM và công cụ. Anthropic mô tả năm mẫu:
- *Chuỗi lệnh tuần tự* — gọi lần lượt, mỗi bước xử lý kết quả của bước trước
- *Phân luồng* — phân loại đầu vào và chuyển đến bộ xử lý chuyên biệt
- *Song song hóa* — chia các tác vụ con độc lập hoặc chạy nhiều mô hình để biểu quyết
- *Điều phối viên - nhân viên* — một LLM trung tâm phân công linh hoạt cho các thành phần con
- *Đánh giá - tối ưu* — vòng lặp cải tiến giữa bộ tạo sinh và bộ đánh giá

**Mức 3: Agent tự chủ**
Hệ thống mà LLM tự quyết định quy trình và cách dùng công cụ của chính mình. LLM tự quyết định bước tiếp theo, không phải theo đường dẫn định sẵn.

Nguyên tắc cốt lõi Anthropic nhấn mạnh: **bắt đầu ở mức thấp nhất còn hoạt động được và chỉ thêm phức tạp khi hiệu quả cải thiện đo được.** Nguyên tắc này cần nhớ khi đánh giá những nền tảng mới mặc định chọn kiến trúc "agent đầy đủ".

---

## Ngăn Xếp Hiện Tại: Một Dải Từ Thấp Đến Cao

*(Phần dưới là một mô hình tư duy để phân tích hệ sinh thái — không phải phân loại chính thức hay tiêu chuẩn ngành đã được xác nhận. Các "tầng" này không có ranh giới rõ ràng; chúng là các vị trí trên một dải từ "tự viết nhiều" đến "chỉ cần cấu hình".)*

Hãy nghĩ như một núm vặn: từ "viết mọi thứ bằng mã" đến "chỉ mô tả agent là gì":

**Thấp nhất — Tự viết toàn bộ điều phối**

Bạn gọi LLM API trực tiếp, tự nối công cụ thực thi, tự quản lý trạng thái, tự xử lý thử lại. Mọi quyết định về phân luồng, bộ nhớ, xử lý lỗi đều nằm trong mã bạn viết và bạn chịu trách nhiệm.

*Công cụ: LangGraph, Claude Agent SDK, API gốc của Anthropic/OpenAI*

*Ví dụ: Bạn tự viết đồ thị LangGraph bằng Python — các nút cho từng bước, các cạnh cho phân luồng, quản lý trạng thái tường minh. Toàn quyền kiểm soát, toàn bộ trách nhiệm.*

**Giữa — Khung phát triển lo phần cơ sở hạ tầng**

Khung phát triển cung cấp các lớp trừu tượng (agent, chuỗi xử lý, bộ nhớ) và quản lý vòng lặp gọi LLM. Bạn tập trung vào cấu hình công cụ và logic của agent, không phải API thô. Vẫn nặng về mã, nhưng có rào chắn an toàn.

*Ví dụ: LangChain agents với bộ nhớ tích hợp và các lớp trừu tượng công cụ. Bạn viết chuỗi xử lý và công cụ; khung phát triển lo điều phối.*

**Cao hơn — Nền tảng lo triển khai và vận hành**

Bạn mô tả *agent là ai* — vai trò, kỹ năng, kiến thức — trong các tệp có cấu trúc. Nền tảng tự lo triển khai, bộ nhớ, môi trường cách ly, cách ly thông tin xác thực, và hỗ trợ đa giao thức. Thứ từng mất một tuần để dựng nay chỉ là một lệnh.

*Ví dụ (Deep Agents Deploy): Bạn viết `AGENTS.md` (danh tính và hành vi bằng markdown thuần) và các tệp `SKILL.md` (kiến thức chuyên môn nạp theo yêu cầu). Chạy `deepagents deploy`. Nền tảng lo toàn bộ bộ nhớ, môi trường cách ly, và máy chủ MCP + A2A + Agent Protocol.*

*Ví dụ (Claude Managed Agents): Bạn cấu hình agent làm gì. Hạ tầng của Anthropic tách Bộ não (Claude + logic điều khiển) khỏi Tay (môi trường thực thi tạm thời, không truy cập được thông tin xác thực). Một nhật ký phiên chỉ-ghi-thêm đóng vai trò bộ nhớ ngoài bền vững — nếu Bộ não gặp sự cố giữa chừng, phiên bản mới tiếp tục từ sự kiện cuối.*

**Cao nhất — Chỉ cần chỉ dẫn, không cần hạ tầng**

Mô hình được hướng dẫn tốt với bộ công cụ phù hợp. Không điều phối tùy chỉnh, không quy trình triển khai — chỉ chỉ dẫn hệ thống và danh sách công cụ.

*Ví dụ: Một phiên Claude với tìm kiếm và trình thông dịch mã. Được hướng dẫn kỹ, phạm vi rõ ràng, dễ gỡ lỗi.*

---

Hai bản phát hành tháng 4 — Managed Agents và Deep Agents Deploy — nằm ở vùng "cao hơn" trên dải này. Chúng không loại bỏ mã hoàn toàn. Nhưng chúng kéo một phần lớn công việc (quản lý bộ nhớ, cách ly thông tin xác thực, hỗ trợ đa giao thức, môi trường cách ly) ra khỏi mã ứng dụng và đưa vào hạ tầng nền tảng.

Trước 2026, xây dựng tương đương "môi trường cách ly có phân quyền xác thực + nhật ký phiên chỉ-ghi-thêm + máy chủ MCP/A2A/Agent Protocol" sẽ mất khoảng một tuần dựng hạ tầng. Các nền tảng này biến điều đó thành mặc định.

---

## Managed Agents và Deep Agents Deploy Thực Sự Cung Cấp Gì

Cả hai nền tảng đồng thuận về một kết luận kiến trúc: **danh tính và năng lực của agent phải tách biệt khỏi hạ tầng thực thi**.

**Deep Agents Deploy của LangChain** hiện thực hóa điều này qua các chuẩn mở:
- `AGENTS.md` — agent là ai, hành xử thế nào, biết gì. Markdown thuần.
- Các tệp `SKILL.md` — kiến thức chuyên môn dạng mô-đun, nạp theo yêu cầu, giảm lượng token tiêu thụ trong khi mở rộng năng lực.

Chạy `deepagents deploy`, khung phát triển xử lý bộ nhớ (lưu trên hệ thống tệp, theo từng phiên), môi trường cách ly (Daytona, Runloop, Modal), và máy chủ sản phẩm với hỗ trợ MCP + A2A + Agent Protocol.

**Managed Agents của Anthropic** đi xa hơn ở tầng hạ tầng:
- **Bộ não** — Claude cộng với logic điều khiển. Không giữ trạng thái, thay thế được.
- **Tay** — vùng chứa tạm thời dùng một lần để công cụ chạy. Không truy cập được thông tin xác thực.
- **Phiên** — nhật ký sự kiện chỉ-ghi-thêm, chính là bộ nhớ ngoài của agent. Nếu Bộ não gặp sự cố, phiên bản mới tiếp tục từ sự kiện cuối.

Giá: $0.08/giờ-phiên cộng chi phí token. Bạn đang trả cho thời gian chạy, không phải hạ tầng.

Cả hai nền tảng đều thật và kiến trúc đều nhất quán. Câu hỏi là liệu chúng có phải công cụ đúng cho một bài toán cụ thể hay không — và câu trả lời trung thực phụ thuộc vào cái bạn đang xây.

---

## Anthropic Về Độ Phức Tạp Của Harness: Đừng Thiết Kế Quá Mức

Song song với bản phát hành Managed Agents, blog kỹ thuật của Anthropic công bố ["Harness Design for Long-Running Application Development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — mô tả chi tiết cách họ xây một harness ba agent để sinh ứng dụng hoàn chỉnh:

- **Người lập kế hoạch**: Chuyển chỉ dẫn ngắn của người dùng thành đặc tả sản phẩm chi tiết
- **Người thực hiện**: Triển khai tính năng từng bước (React, Vite, FastAPI, SQLite)
- **Người đánh giá**: Kiểm thử qua Playwright, chấm theo tiêu chí, cung cấp phản hồi cho vòng lặp tiếp theo

Vòng lặp Lập kế hoạch/Thực hiện/Đánh giá cho kết quả tốt — nhưng phát hiện thú vị hơn là những gì xảy ra khi mô hình nền cải thiện từ Claude Opus 4.5 lên 4.6.

> *"Mỗi thành phần trong một harness đều mã hóa một giả định về những gì mô hình không thể tự làm, và những giả định đó đáng được kiểm tra lại."*

Khi Claude 4.6 xử lý tốt hơn với các tác vụ dài liền mạch, Anthropic đã đơn giản hóa harness: bỏ phân chia theo giai đoạn ngắn, chuyển đánh giá từ theo-từng-giai-đoạn sang đánh-giá-cuối-cùng. Điều phối trở nên ít phức tạp hơn, không phải nhiều hơn — vì mô hình đã có thể xử lý nhiều hơn trực tiếp.

Nguyên tắc thứ hai: *"tìm giải pháp đơn giản nhất có thể, và chỉ tăng phức tạp khi cần thiết."*

Điều này cắt cả hai chiều. Đây là cảnh báo chống việc vội vàng dùng harness cũng như chống việc tự thiết kế điều phối quá mức. Nếu bài toán phù hợp với một agent đơn được hướng dẫn tốt (Mức 1), thêm các mẫu luồng công việc hay nền tảng harness đầy đủ thường là chi phí phát sinh, không phải giá trị.

---

## Những Đánh Đổi Thực Sự

**Các nền tảng harness làm dễ dàng hơn những gì:**
- Cách ly thông tin xác thực và bảo mật môi trường mặc định — không cần tự thiết kế
- Hỗ trợ đa giao thức (MCP, A2A, Agent Protocol) không cần tự tích hợp
- Triển khai một agent chạy được trong một ngày thay vì một tuần dựng hạ tầng
- Tệp định nghĩa agent mà người không viết mã cũng đọc và sửa được
- Quản lý bộ nhớ mà không cần chọn và cấu hình kho vector

**Những gì chúng làm khó hơn hoặc đưa vào như mối quan tâm mới:**
- Logic phân luồng tùy chỉnh không khớp với mô hình thực thi của nền tảng
- Kiểm soát chi tiết thời điểm và cách bộ nhớ được ghi hoặc đọc
- Gỡ lỗi: nhật ký phiên chỉ-ghi-thêm tốt cho khôi phục, kém trực quan hơn khi kiểm tra
- Phụ thuộc nhà cung cấp — Managed Agents là hạ tầng chỉ dành cho Claude
- Dự đoán chi phí ở quy mô lớn: $0.08/giờ-phiên rẻ cho đến khi bạn chạy nhiều agent sống lâu
- Câu hỏi mở: các giá trị mặc định của nền tảng có khớp với nhu cầu thực của agent bạn không

**Deep Agents Deploy** xử lý vấn đề khả năng di chuyển tốt hơn Managed Agents: chạy trên bất kỳ mô hình nào (OpenAI, Anthropic, Google, Ollama), triển khai tự lưu trữ hoặc đám mây, và định dạng `AGENTS.md`/`SKILL.md` được cấp phép MIT. Chi phí chuyển đổi thấp. Managed Agents cung cấp tích hợp hạ tầng chặt hơn với giá là phụ thuộc nhà cung cấp.

---

## Tại Sao Bảo Mật Là Động Lực Thực Sự

Một lý do cụ thể sự dịch chuyển này xảy ra không phải mang tính triết lý — mà là bảo mật thiết thực.

Trong thiết kế gắn chặt, mã mà agent sinh ra hoặc thực thi chạy trong cùng không gian tiến trình với thông tin xác thực. Một cuộc tấn công chèn chỉ dẫn thành công trong kịch bản đó không chỉ xâm phạm một tác vụ — mà có thể phơi bày mọi thứ agent đó có quyền truy cập.

Cách ly thông tin xác thực giải quyết điều này trực tiếp. Đây không phải khái niệm mới: lưới dịch vụ, trình quản lý bí mật như Vault, phân quyền theo dịch vụ (IAM roles) đã áp dụng nguyên tắc này từ lâu. Mẫu thiết kế rất rõ ràng — giới hạn những gì một ngữ cảnh thực thi cụ thể có thể tiếp cận.

Sự tách biệt Bộ não/Tay của Managed Agents áp dụng nguyên tắc này vào kiến trúc agent: môi trường cách ly nơi mã sinh ra chạy không truy cập được token xác thực. Deep Agents Deploy thực thi sự cách ly tương tự thông qua lớp trừu tượng nhà cung cấp môi trường.

Bạn hoàn toàn có thể đạt mức cách ly tương tự khi tự xây harness — mẫu thiết kế thuộc về kiến trúc, không phải độc quyền. Các đội đã quan tâm đến bảo mật trong hạ tầng của mình thường đã làm điều này rồi. Sự khác biệt nằm ở cách đóng gói: các nền tảng harness biến cách ly thông tin xác thực thành mặc định thay vì một quyết định thiết kế cần chủ động thực hiện. Đó là lợi ích tiện lợi thực sự, không phải tuyên bố rằng hệ thống tự xây kém bảo mật hơn.

Đánh đổi trung thực hơn: tự xây harness thì có toàn quyền kiểm soát mô hình bảo mật, đổi lại là công sức thiết kế và duy trì nó. Dùng harness nền tảng thì có một bộ mặc định đã được kiểm thử, đổi lại là phụ thuộc vào các giả định bảo mật của nền tảng. Không cái nào tốt hơn tuyệt đối — chúng là những lựa chọn khác nhau về việc đầu tư công sức vào đâu.

---

## OpenClaw: Khi Định Nghĩa Agent Ở Tầng Context Không Đi Kèm Hạ Tầng Bảo Mật

Cả Managed Agents lẫn Deep Agents Deploy đều chia sẻ một concept cốt lõi: định nghĩa agent ở tầng context — danh tính, kỹ năng, kiến thức trong các tệp có cấu trúc, không phải trong code. OpenClaw, một framework agent do cộng đồng phát triển, đã đến cùng kết luận kiến trúc này theo con đường riêng.

OpenClaw định nghĩa agent theo cùng cách: một agent là danh tính cộng với tập kỹ năng tải từ ClawHub — kho kỹ năng cộng đồng của nó. Concept ánh xạ trực tiếp vào `AGENTS.md` + `SKILL.md` hay cấu hình Brain của Managed Agents. Cùng ý tưởng, khác ngữ cảnh thực thi.

Điểm khác biệt nằm ở những gì đi kèm — hoặc chính xác hơn, những gì không đi kèm.

Tháng 2/2026, một cuộc tấn công chuỗi cung ứng nhắm vào ClawHub (được gọi là "ClawHavoc") được tiết lộ. Kẻ tấn công đã xâm phạm 12 tài khoản publisher và đăng tải 1.184 kỹ năng độc hại. Một khi được nạp vào context của agent, các kỹ năng này có thể rò rỉ thông tin xác thực, chuyển hướng lệnh gọi công cụ, hoặc nhiễm độc suy luận của agent. Cuộc tấn công đã hoạt động trong một khoảng thời gian chưa xác định trước khi bị phát hiện.

Nghiên cứu bảo mật từ Snyk củng cố thêm quy mô của vấn đề: báo cáo ToxicSkills phát hiện 36,8% kỹ năng trên ClawHub có lỗ hổng ở dạng nào đó, với 13,4% được đánh giá ở mức nghiêm trọng (critical).

Đây không phải là phê phán concept kiến trúc của OpenClaw — ý tưởng định nghĩa agent ở tầng context là đúng đắn. Đây là bằng chứng về những gì concept đó cần để khả thi trong môi trường sản phẩm: một mô hình tin cậy cho chuỗi cung ứng kỹ năng, cách ly thông tin xác thực để kỹ năng được nạp không thể truy cập token xác thực, và hạ tầng kiểm toán ghi lại kỹ năng nào đã chạy và khi nào.

So sánh giữa ba cách tiếp cận:

| | Mô hình chuỗi cung ứng | Cách ly thông tin xác thực | Nguồn kỹ năng |
|---|---|---|---|
| **OpenClaw** | Registry cộng đồng mở (ClawHub) | Không bắt buộc | Tải lúc runtime từ publisher chưa kiểm tra |
| **Deep Agents Deploy** | Tệp bạn kiểm soát | Lớp trừu tượng sandbox | Tệp `SKILL.md` cục bộ |
| **Managed Agents** | Nền tảng kiểm soát, đóng | Tách Brain/Hands theo thiết kế | Cấu hình trong nền tảng |

ClawHavoc là căn cứ thực tế hữu ích cho phần bảo mật ở trên: định nghĩa agent ở tầng context không phải rủi ro bảo mật tự thân. Rủi ro là nạp kỹ năng từ registry cộng đồng không được kiểm tra vào ngữ cảnh thực thi có quyền truy cập thông tin xác thực. Mẫu kiến trúc tách biệt với sự cố bảo mật — nhưng chỉ khi hạ tầng bên dưới nó xử lý nghiêm túc mối đe dọa đó.

---

## AgentOS: Tầng Nghiên Cứu Đang Hình Thành

Cả giới học thuật lẫn ngành công nghiệp đều đang hội tụ về cùng một bộ thành phần cơ bản, dù gọi tên khác nhau.

**AIOS** (Đại học Rutgers, COLM 2025) là hiện thực hóa có bình duyệt đầu tiên của cái thực chất là "hệ điều hành cho agent":

- Bộ lập lịch Agent, Trình quản lý ngữ cảnh, Trình quản lý bộ nhớ, Trình quản lý lưu trữ, Trình quản lý công cụ, Trình quản lý truy cập

Ánh xạ điều này vào Managed Agents: Bộ não ≈ Bộ lập lịch + Trình quản lý ngữ cảnh. Phiên ≈ Trình quản lý bộ nhớ + Trình quản lý lưu trữ. Tay ≈ Trình quản lý công cụ + Trình quản lý truy cập. Sự hội tụ này không phải ngẫu nhiên — hai con đường nghiên cứu và công nghiệp riêng biệt đến cùng kết luận kiến trúc vì họ đang giải cùng bài toán quản lý tài nguyên.

AIOS cho thấy tốc độ thực thi nhanh hơn 2.1 lần khi phục vụ nhiều agent so với cách tiếp cận thông thường — các thành phần cơ bản có tác động thực sự đến hiệu năng.

**Hội thảo AgenticOS 2026** (tổ chức cùng ASPLOS 2026) cho thấy giới học thuật đang coi đây là nghiên cứu hệ thống nghiêm túc: cơ chế cấp hệ điều hành cho khối lượng công việc agent AI, mô hình cách ly, kỹ thuật lập lịch. Cùng mức độ nghiêm ngặt đã tạo ra bộ nhớ ảo và lập lịch tiến trình nay được áp dụng vào hạ tầng agent.

---

## Xây Dựng Ở Tầng Nào?

Câu hỏi này thực ra tách làm hai — và rất dễ bị nhầm lẫn với nhau:

**Câu hỏi 1: Dùng công cụ/nền tảng nào?** (vị trí trên dải)

**Xây với khung phát triển cấp thấp (LangGraph / Claude Agent SDK) khi:**
- Logic phân luồng của bạn thực sự mới lạ hoặc đặc thù theo lĩnh vực
- Cần kiểm soát chặt các chuyển đổi trạng thái — quy trình tài chính, yêu cầu kiểm toán
- Bạn đang xây hạ tầng mà người khác sẽ triển khai agent lên trên
- Bạn đang nghiên cứu kiến trúc agent, không phải triển khai sản phẩm

**Xây với nền tảng quản lý (Managed Agents / Deep Agents Deploy) khi:**
- Triển khai agent với mục đích cụ thể, rõ ràng
- Bảo mật và cách ly thông tin xác thực là yêu cầu bắt buộc, không phải có-thì-tốt
- Cần hỗ trợ đa giao thức mà không cần tự tích hợp
- Nút thắt cổ chai của đội là triển khai và vận hành, không phải thiết kế suy luận agent
- Muốn định nghĩa agent mà người không viết mã cũng đọc và sửa được

**Câu hỏi 2: Agent thực sự cần bao nhiêu điều phối?** (3 mức của Anthropic — một trục riêng biệt)

Sự tiến triển của Anthropic — LLM tăng cường → Luồng công việc → Agent tự chủ — mô tả *độ phức tạp* của agent, không phải lựa chọn nền tảng. Một nền tảng quản lý hoàn toàn có thể chạy một agent Mức 1. Một harness LangGraph tùy chỉnh hoàn toàn có thể triển khai các mẫu Mức 3. Hai trục này độc lập với nhau.

Câu trả lời mặc định trên trục này: **bắt đầu ở Mức 1 và chỉ tăng phức tạp khi có thể đo được cải thiện.** Hầu hết các bài toán tưởng như cần quy trình đa agent thực ra có thể giải bởi một mô hình đơn được hướng dẫn tốt với bộ công cụ đúng. Thêm mẫu luồng công việc Mức 2 hay điều khiển tự chủ Mức 3 chỉ đáng khi hiệu quả cải thiện đo được — bất kể bạn đang dùng nền tảng nào.

**Trường hợp thú vị nhất là kết hợp:** logic thực thi tùy chỉnh trên LangGraph, phía trước là định dạng triển khai tương thích harness. Deep Agents Deploy đã hỗ trợ điều này — `deepagents deploy` có thể đặt trước một phần phụ trợ LangGraph tùy chỉnh trong khi vẫn cung cấp các điểm cuối chuẩn và quản lý bộ nhớ. Bạn có triển khai có chủ kiến mà không mất quyền kiểm soát điều phối.

---

## Những Thứ Không Thay Đổi

Vài điều vẫn đúng bất kể bạn xây ở tầng nào:

**Chất lượng mô hình vẫn chi phối.** Bộ não vẫn là Claude. Harness thiết kế tốt trên mô hình yếu vẫn thua harness thiết kế kém trên mô hình mạnh. Harness nhân lên những gì mô hình có thể làm — không thay thế những gì mô hình không biết.

**Phân rã tác vụ vẫn quan trọng.** Không nền tảng nào tự động xử lý được tác vụ mơ hồ hoặc thiếu thông tin. Chỉ dẫn cho agent vẫn cần rõ ràng về "hoàn thành" trông như thế nào. `AGENTS.md` chỉ hữu ích như cái suy nghĩ đã đi vào việc viết nó.

**Đánh giá vẫn là việc của bạn.** Managed Agents sinh nhật ký phiên. Deep Agents tích hợp LangSmith. Nhưng quyết định agent có làm đúng không — và chẩn đoán các lỗi im lặng — vẫn là trách nhiệm của bạn.

**Độ phức tạp harness nên theo sát năng lực mô hình.** Như công trình kỹ thuật harness của chính Anthropic cho thấy: khi mô hình cải thiện, phản ứng đúng thường là đơn giản hóa harness, không phải giữ nguyên các ràng buộc cũ.

---

## Bức Tranh Lớn Hơn

Hệ sinh thái agent đang phân tầng thành các lớp riêng biệt — đây là dấu hiệu của sự trưởng thành, giống cách hạ tầng web phân tầng từ TCP thô thành HTTP, rồi khung phát triển web, rồi nền tảng triển khai.

Các khung phát triển cấp thấp (LangGraph, Claude Agent SDK) không bị thay thế. Chúng đang trở thành nền tảng mà các nền tảng harness chạy lên trên — cách TCP/IP không biến mất khi HTTP xuất hiện. Nhưng với hầu hết triển khai sản phẩm, câu hỏi ngày càng là: bài toán của tôi thuộc tầng nào trong ngăn xếp này?

Managed Agents và Deep Agents Deploy là các nền tảng thật với giá trị thật cho các trường hợp sử dụng cụ thể. Chúng cũng còn mới, phụ thuộc nhà cung cấp theo những cách khác nhau, và thêm ràng buộc mà một số khối lượng công việc sẽ không muốn. Đó là đánh đổi bình thường, không phải phát hiện mang tính cách mạng.

Bài học bền vững hơn từ cả hai bản phát hành, và từ nghiên cứu kỹ thuật harness của Anthropic, là về kiến trúc: **danh tính và năng lực của agent thuộc về một tầng định nghĩa di động, dễ đọc; thực thi, bộ nhớ, bảo mật, và bộ máy triển khai thuộc về hạ tầng bên dưới nó.** Dù bạn đến đó qua nền tảng quản lý hay tự xây, sự phân tách đó là đáng hướng tới.

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
