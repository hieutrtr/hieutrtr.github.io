---
layout: post
title: "Điều phối đa đồ thị phân tán với LangGraph"
description: "Cách engine Pregel và giao diện PregelProtocol của LangGraph cho phép kết hợp các AI agent được triển khai độc lập thành một hệ thống phân tán thống nhất -- với streaming, interrupt và checkpointing xuyên ranh giới dịch vụ."
featured: true
lang: vi
ref: langgraph-multi-graph-orchestration
permalink: /vi/langgraph-multi-graph-orchestration/
banner: /assets/images/03-multi-graph-architecture.png
banner_alt: "Kiến trúc Đa Đồ thị Phân tán"
date: 2026-03-06
---

Bạn có một research agent. Một coding agent. Một review agent. Mỗi cái được xây dựng bởi một team khác nhau, triển khai trên các server khác nhau. Bây giờ bạn cần chúng làm việc cùng nhau như một hệ thống.

Đây là bài toán điều phối đa đồ thị (multi-graph orchestration), và LangGraph giải quyết nó với một ý tưởng đơn giản đến bất ngờ: một remote graph chỉ là một node bình thường.

---

## Pregel là gì, và tại sao bạn nên quan tâm?

Trước khi đi sâu vào điều phối đa đồ thị, đáng để hiểu về engine làm nên tất cả.

Runtime của LangGraph được gọi là **Pregel**, đặt tên theo [Google's Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) -- một hệ thống được thiết kế để xử lý đồ thị quy mô lớn. Pregel gốc được xây dựng để chạy các thuật toán như PageRank trên hàng tỷ trang web. LangGraph mượn cùng mô hình thực thi, nhưng áp dụng cho các workflow AI agent.

### Mô hình thực thi

Pregel sử dụng mô hình thực thi **Bulk Synchronous Parallel (BSP)**. Nghe có vẻ học thuật, nhưng ý tưởng rất đơn giản. Đồ thị của bạn chạy theo các **superstep** rời rạc, và mỗi superstep có ba giai đoạn:

1. **Plan** -- Xác định node nào cần chạy. Ở bước đầu tiên, đó là các node kết nối với `START`. Ở các bước tiếp theo, đó là bất kỳ node nào có input channel được cập nhật ở bước trước.

2. **Execute** -- Chạy tất cả các node được chọn **song song**. Đây là điểm mấu chốt: các node không thấy được các thao tác ghi của nhau trong quá trình thực thi. Một research node và một coding node chạy trong cùng một superstep không thể can thiệp lẫn nhau.

3. **Update** -- Áp dụng tất cả output của node vào các state channel dùng chung. Bây giờ các thao tác ghi trở nên hữu hình, và superstep tiếp theo có thể bắt đầu.

Quá trình này lặp lại cho đến khi không còn node nào được kích hoạt, hoặc đạt `recursion_limit`.

![Mô hình thực thi Pregel BSP](/assets/images/01-pregel-superstep.png)

### Node và Channel

Trong mô hình của Pregel, các node không gọi trực tiếp lẫn nhau. Chúng giao tiếp thông qua **channel** -- các container có kiểu dữ liệu giữ trạng thái giữa các superstep:

- **LastValue** -- lưu giá trị gần nhất (mặc định cho các trường `StateGraph`)
- **BinaryOperatorAggregate** -- tích luỹ giá trị với một reducer (đây là thứ cung cấp sức mạnh cho `add_messages`)
- **Topic** -- tích luỹ pub/sub cho nhiều giá trị
- **EphemeralValue** -- giá trị tạm thời được xoá sau mỗi bước

Khi bạn viết `messages: Annotated[list, add_messages]` trong state, bạn đang định nghĩa một `BinaryOperatorAggregate` channel với `add_messages` là toán tử nhị phân. Khi hai node cùng ghi message trong cùng một superstep, reducer sẽ hợp nhất chúng một cách tất định.

### Tại sao điều này quan trọng cho điều phối phân tán

Mô hình BSP cung cấp ba thuộc tính thiết yếu cho hệ thống phân tán:

1. **Tính tất định** -- Các node song song không thể tranh chấp trên state dùng chung. Giai đoạn cập nhật channel đảm bảo thứ tự nhất quán.
2. **Khả năng checkpoint** -- State sạch giữa các superstep, nên bạn có thể chụp ảnh, lưu trữ và tiếp tục sau đó.
3. **Khả năng tổ hợp** -- Toàn bộ mô hình thực thi nằm sau một giao diện (`PregelProtocol`). Bất kỳ thứ gì implement giao diện đó -- đồ thị cục bộ, đồ thị từ xa, functional entrypoint -- đều có thể được tổ hợp như một node.

Hầu hết người dùng không tương tác trực tiếp với Pregel. `StateGraph.compile()` và `@entrypoint` đều tạo ra các instance Pregel. Nhưng hiểu mô hình giải thích *tại sao* những thứ như streaming, interrupt và tổ hợp từ xa hoạt động trơn tru -- chúng đều được xây dựng trên cùng một mô hình thực thi superstep với giao tiếp qua channel.

---

## Ý tưởng cốt lõi

Engine thực thi của LangGraph (Pregel) và client từ xa (RemoteGraph) đều implement cùng một giao diện -- `PregelProtocol`. Điều này có nghĩa là bạn có thể kết nối một đồ thị chạy trên server ở Tokyo vào một orchestrator chạy trên laptop của bạn, và nó hoạt động giống hệt như một hàm cục bộ.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.pregel.remote import RemoteGraph

# Đây là các HTTP client, không phải code cục bộ
research = RemoteGraph("research-agent", url="https://research.example.com", name="research")
coder    = RemoteGraph("code-agent",     url="https://code.example.com",     name="coder")

# Nhưng chúng được kết nối như bất kỳ node nào khác
graph = StateGraph(MyState)
graph.add_node("research", research)
graph.add_node("coder", coder)
graph.add_edge(START, "research")
graph.add_edge("research", "coder")
graph.add_edge("coder", END)

app = graph.compile()
```

![PregelProtocol: Giao diện Toàn cục](/assets/images/02-pregel-protocol.png)

Đó là tất cả. `invoke()`, `stream()`, interrupt, checkpointing -- mọi thứ hoạt động xuyên ranh giới.

---

## Tại sao điều này quan trọng

Hầu hết các framework điều phối AI cho bạn một trong hai lựa chọn: mọi thứ chạy trong một tiến trình, hoặc bạn tự xử lý với REST call và message queue.

LangGraph cho bạn lựa chọn thứ ba: xây dựng mỗi agent như một đồ thị độc lập với deployment riêng, state riêng, team riêng sở hữu -- rồi tổ hợp chúng với cùng API bạn dùng cho các node cục bộ. Không cần glue code. Không cần serialization tuỳ chỉnh. Không cần logic retry tự viết.

Điều này mở khoá các kiến trúc trông như hệ thống phân tán thực sự:

![Kiến trúc Đa Đồ thị Phân tán](/assets/images/03-multi-graph-architecture.png)

Mỗi ô có thể triển khai độc lập qua `langgraph up`. Mỗi ô có checkpoint storage riêng. Mỗi ô có thể được versioning và rollback độc lập. Orchestrator chỉ đơn giản kết nối chúng lại.

---

## Xây dựng Orchestrator

### Thiết kế State

Orchestrator quản lý state điều phối. Các worker graph quản lý state domain riêng của chúng. Đây là các mối quan tâm riêng biệt.

```python
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import add_messages

class OrchestratorState(TypedDict):
    messages: Annotated[list, add_messages]
    plan: list[dict]
    results: Annotated[dict[str, Any], lambda a, b: {**a, **b}]
    status: str
```

Orchestrator không cần biết state nội bộ của research agent. Nó gửi message vào, nhận kết quả trả về. Trường `results` tích luỹ output từ nhiều worker sử dụng merge reducer.

### Định tuyến

Pipeline tĩnh là trường hợp đơn giản nhất, nhưng sức mạnh thực sự nằm ở định tuyến động. Bạn có thể sử dụng conditional edge, LLM-based router, hoặc fan-out pattern.

**Định tuyến có điều kiện** -- chọn worker dựa trên state:

```python
def route(state):
    if "code" in state["messages"][-1].content.lower():
        return "coder"
    return "research"

graph.add_conditional_edges("router", route)
```

**Fan-out với Send** -- gửi đến nhiều worker song song:

```python
from langgraph.types import Send

def dispatch(state):
    return [Send(task["worker"], state) for task in state["plan"]]

graph.add_conditional_edges("planner", dispatch)
```

Mỗi `Send` kích hoạt một lời gọi song song đến remote graph đích. Kết quả được hợp nhất lại thông qua state reducer. Đây là map-reduce cho AI agent.

![Fan-Out / Map-Reduce với Send()](/assets/images/06-fan-out-map-reduce.png)

---

## Streaming xuyên ranh giới dịch vụ

Đây là nơi hầu hết các cách tiếp cận DIY thất bại. Khi bạn lồng đồ thị xuyên ranh giới HTTP, bạn muốn streaming hoạt động bình thường -- output LLM token-by-token từ một remote agent phải chảy qua orchestrator đến client mà không bị buffer.

![Streaming xuyên ranh giới dịch vụ](/assets/images/04-stream-propagation.png)

LangGraph xử lý điều này tự động. Việc implement stream của `RemoteGraph`:

1. Chuyển tiếp các stream mode của parent đến remote server
2. Thêm namespace prefix để bạn biết subgraph nào phát ra sự kiện nào
3. Dẫn các sự kiện ngược lên qua stream của orchestrator

```python
async for chunk in app.astream(
    {"messages": [("user", "Build an auth API")]},
    config={"configurable": {"thread_id": "project-1"}},
    stream_mode=["updates", "messages"],
    subgraphs=True,
):
    ns, mode, data = chunk
    origin = "/".join(ns) or "orchestrator"
    print(f"[{origin}] {mode}: {data}")
```

Output trông như:

```
[orchestrator] updates: {"planner": {"plan": [...]}}
[research] messages: ("To build", {...})
[research] messages: (" an auth", {...})
[research] messages: (" API...", {...})
[research] updates: {"search": {"findings": "..."}}
[coder] messages: ("```python\n", {...})
[coder] messages: ("from fastapi", {...})
...
```

Cả bảy stream mode đều hoạt động xuyên ranh giới: `values`, `updates`, `messages`, `custom`, `tasks`, `checkpoints`, và `debug`.

---

## Human-in-the-Loop xuyên đồ thị

Đây là một kịch bản: coding agent của bạn tạo một database migration. Trước khi chạy, bạn muốn một người duyệt. Coding agent là một remote graph. Người dùng đang tương tác với client của orchestrator.

![Human-in-the-Loop xuyên ranh giới đồ thị](/assets/images/05-interrupt-flow.png)

Điều này hoạt động ngay. Khi một remote graph gọi `interrupt()`, interrupt được truyền lên qua orchestrator đến client:

```python
# Bên trong remote coding agent (triển khai riêng):
def migration_node(state):
    migration = generate_migration(state)
    approval = interrupt({"sql": migration, "question": "Run this migration?"})
    if approval["approved"]:
        run_migration(migration)
    return {"result": "Migration applied"}
```

Client của orchestrator thấy interrupt:

```python
state = app.get_state(config)
print(state.interrupts)  # [Interrupt(value={"sql": "ALTER TABLE...", "question": "..."})]

# Tiếp tục
app.invoke(Command(resume={"approved": True}), config=config)
```

Lệnh resume chạy ngược qua orchestrator, qua `RemoteGraph`, đến đúng thread trên remote server. Không cần plumbing.

---

## Checkpointing độc lập

Mỗi đồ thị trong hệ thống checkpoint độc lập. Orchestrator lưu các quyết định định tuyến và kết quả tích luỹ. Mỗi remote graph lưu lịch sử hội thoại nội bộ và tool call riêng. Chúng không chia sẻ database.

Đây là một tính năng, không phải hạn chế. Nó có nghĩa là:

- Bạn có thể kiểm tra và phát lại workflow của orchestrator mà không cần chạm vào state của worker
- Mỗi team kiểm soát backend persistence riêng (SQLite cho dev, Postgres cho prod)
- Checkpoint storage được mở rộng độc lập cho mỗi dịch vụ
- Time-travel debugging hoạt động ở mỗi tầng

```python
# Kiểm tra lịch sử orchestrator
for snapshot in app.get_state_history(config, limit=5):
    print(f"Step {snapshot.metadata['step']}: next={snapshot.next}")

# Kiểm tra lịch sử remote graph trực tiếp
remote = RemoteGraph("research-agent", url="https://research.example.com")
for snapshot in remote.get_state_history(remote_config, limit=5):
    print(f"Step {snapshot.metadata['step']}: next={snapshot.next}")
```

---

## Xử lý lỗi

Các lời gọi từ xa có thể thất bại. Mạng không đáng tin cậy. LangGraph cung cấp retry policy ở cấp node:

```python
from langgraph.types import RetryPolicy

graph.add_node(
    create_remote("research"),
    retry_policy=RetryPolicy(
        max_attempts=3,
        initial_interval=1.0,
        backoff_factor=2.0,
    ),
)
```

Để kiểm soát tốt hơn, bọc lời gọi từ xa:

```python
def resilient_research(state):
    try:
        return create_remote("research-v2").invoke(state)
    except RemoteException:
        return create_remote("research-v1").invoke(state)  # fallback về phiên bản trước
```

Lỗi từ xa xuất hiện dưới dạng `RemoteException`. Interrupt xuất hiện dưới dạng `GraphInterrupt`. Command xuất hiện dưới dạng `ParentCommand`. Mỗi loại có ngữ nghĩa xử lý riêng biệt -- chúng không bị gộp chung.

---

## Uỷ quyền phân cấp

Các remote graph có thể gửi lệnh ngược lại orchestrator cha sử dụng `Command(graph=Command.PARENT)`. Điều này cho phép cấu trúc team phân cấp:

```
Orchestrator
+-- Team Lead A (remote)
|   +-- Worker A1 (remote)
|   +-- Worker A2 (remote)
+-- Team Lead B (remote)
    +-- Worker B1 (remote)
```

Một worker có thể leo thang lên team lead. Một team lead có thể leo thang lên orchestrator. Cơ chế `ParentCommand` xử lý điều này ở bất kỳ độ sâu lồng nào.

```python
# Worker bên trong một remote team graph
def worker(state):
    if state["confidence"] < 0.5:
        return Command(
            graph=Command.PARENT,
            update={"needs_help": True},
            goto="escalation_handler",
        )
    return {"result": do_work(state)}
```

---

## Triển khai

Mỗi đồ thị triển khai độc lập. File config `langgraph.json` định nghĩa điểm vào của đồ thị:

```json
{
    "python_version": "3.12",
    "dependencies": ["./requirements.txt"],
    "graphs": {
        "research-agent": "./graphs/research.py:graph"
    }
}
```

Sau đó triển khai:

```bash
langgraph up --port 8001
```

Hoặc sử dụng `langgraph dev` để phát triển cục bộ với hot-reloading. Trong môi trường phát triển, tất cả remote graph có thể chạy trên localhost với các port khác nhau. Trong production, chúng là các dịch vụ riêng biệt sau các URL riêng biệt.

Bản thân orchestrator cũng có thể được triển khai theo cách tương tự, hoặc chạy như một script cục bộ. Nó chỉ là một đồ thị tình cờ gọi các đồ thị khác qua HTTP.

---

## Quan sát hệ thống

Bật `distributed_tracing=True` trên các instance `RemoteGraph` để có trace thống nhất trong LangSmith xuyên tất cả dịch vụ:

```python
research = RemoteGraph(
    "research-agent",
    url="https://research.example.com",
    distributed_tracing=True,
)
```

Một trace hiển thị toàn bộ hành trình: orchestrator định tuyến -> research agent tìm kiếm -> coding agent tạo code -> reviewer kiểm tra. Phân tích độ trễ, số lượng token, và quy lỗi xuyên toàn bộ hệ thống phân tán.

Bạn cũng có thể trực quan hoá toàn bộ topology đồ thị:

```python
app.get_graph(xray=2).draw_mermaid_png()
```

Điều này render đồ thị orchestrator với nội bộ remote subgraph được mở rộng đến độ sâu 2.

---

## Khi nào nên sử dụng

**Sử dụng điều phối đa đồ thị khi:**

- Các team khác nhau sở hữu các agent khác nhau và cần chu kỳ triển khai độc lập
- Bạn cần mở rộng các agent độc lập (research agent cần GPU, router thì không)
- Bạn muốn versioning và rollback các agent độc lập
- Hệ thống của bạn có ranh giới domain tự nhiên (research vs. coding vs. review)
- Bạn cần cách ly lỗi -- một agent bị crash không nên làm sập toàn bộ hệ thống

**Giữ một đồ thị duy nhất khi:**

- Một team sở hữu mọi thứ
- Các agent chia sẻ hầu hết state
- Chi phí của HTTP call không đáng để đổi lấy lợi ích kiến trúc
- Bạn đang làm prototype và tốc độ lặp lại quan trọng hơn tính linh hoạt triển khai

---

## Kết luận

Quyết định thiết kế cốt lõi trong LangGraph là `PregelProtocol` là giao diện toàn cục. Một `StateGraph` đã compile, một hàm `@entrypoint`, và một HTTP client `RemoteGraph` đều implement nó. Điều này có nghĩa là tổ hợp là miễn phí -- bạn không phải trả thuế trừu tượng khi chuyển sang phân tán.

Xây dựng các agent của bạn như các đồ thị độc lập. Triển khai chúng riêng rẽ. Kết nối chúng với một orchestrator coi mỗi cái như chỉ là một node bình thường. Streaming, interrupt, checkpointing, và xử lý lỗi hoạt động xuyên ranh giới mà không cần thêm code.

Khoảng cách giữa "nó chạy trên laptop của tôi" và "nó chạy trong production xuyên năm dịch vụ" nhỏ hơn bạn nghĩ.

---

*Nếu bài viết có ích, kết nối với mình trên [LinkedIn](https://www.linkedin.com/in/hieu-tran-data/) hoặc xem thêm các project tại [GitHub](https://github.com/hieutrtr/).*
