---
layout: post
title: "Dieu phoi da do thi phan tan voi LangGraph"
description: "Cach engine Pregel va giao dien PregelProtocol cua LangGraph cho phep ket hop cac AI agent duoc trien khai doc lap thanh mot he thong phan tan thong nhat -- voi streaming, interrupt va checkpointing xuyen ranh gioi dich vu."
featured: true
lang: vi
ref: langgraph-multi-graph-orchestration
permalink: /vi/langgraph-multi-graph-orchestration/
banner: /assets/images/03-multi-graph-architecture.png
banner_alt: "Kien truc Da Do thi Phan tan"
date: 2026-03-06
---

Ban co mot research agent. Mot coding agent. Mot review agent. Moi cai duoc xay dung boi mot team khac nhau, trien khai tren cac server khac nhau. Bay gio ban can chung lam viec cung nhau nhu mot he thong.

Day la bai toan dieu phoi da do thi (multi-graph orchestration), va LangGraph giai quyet no voi mot y tuong don gian den bat ngo: mot remote graph chi la mot node binh thuong.

---

## Pregel la gi, va tai sao ban nen quan tam?

Truoc khi di sau vao dieu phoi da do thi, dang de hieu ve engine lam nen tat ca.

Runtime cua LangGraph duoc goi la **Pregel**, dat ten theo [Google's Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) -- mot he thong duoc thiet ke de xu ly do thi quy mo lon. Pregel goc duoc xay dung de chay cac thuat toan nhu PageRank tren hang ty trang web. LangGraph muon cung mo hinh thuc thi, nhung ap dung cho cac workflow AI agent.

### Mo hinh thuc thi

Pregel su dung mo hinh thuc thi **Bulk Synchronous Parallel (BSP)**. Nghe co ve hoc thuat, nhung y tuong rat don gian. Do thi cua ban chay theo cac **superstep** roi rac, va moi superstep co ba giai doan:

1. **Plan** -- Xac dinh node nao can chay. O buoc dau tien, do la cac node ket noi voi `START`. O cac buoc tiep theo, do la bat ky node nao co input channel duoc cap nhat o buoc truoc.

2. **Execute** -- Chay tat ca cac node duoc chon **song song**. Day la diem mau chot: cac node khong thay duoc cac thao tac ghi cua nhau trong qua trinh thuc thi. Mot research node va mot coding node chay trong cung mot superstep khong the can thiep lan nhau.

3. **Update** -- Ap dung tat ca output cua node vao cac state channel dung chung. Bay gio cac thao tac ghi tro nen huu hinh, va superstep tiep theo co the bat dau.

Qua trinh nay lap lai cho den khi khong con node nao duoc kich hoat, hoac dat `recursion_limit`.

![Mo hinh thuc thi Pregel BSP](/assets/images/01-pregel-superstep.png)

### Node va Channel

Trong mo hinh cua Pregel, cac node khong goi truc tiep lan nhau. Chung giao tiep thong qua **channel** -- cac container co kieu du lieu giu trang thai giua cac superstep:

- **LastValue** -- luu gia tri gan nhat (mac dinh cho cac truong `StateGraph`)
- **BinaryOperatorAggregate** -- tich luy gia tri voi mot reducer (day la thu cung cap suc manh cho `add_messages`)
- **Topic** -- tich luy pub/sub cho nhieu gia tri
- **EphemeralValue** -- gia tri tam thoi duoc xoa sau moi buoc

Khi ban viet `messages: Annotated[list, add_messages]` trong state, ban dang dinh nghia mot `BinaryOperatorAggregate` channel voi `add_messages` la toan tu nhi phan. Khi hai node cung ghi message trong cung mot superstep, reducer se hop nhat chung mot cach tat dinh.

### Tai sao dieu nay quan trong cho dieu phoi phan tan

Mo hinh BSP cung cap ba thuoc tinh thiet yeu cho he thong phan tan:

1. **Tinh tat dinh** -- Cac node song song khong the tranh chap tren state dung chung. Giai doan cap nhat channel dam bao thu tu nhat quan.
2. **Kha nang checkpoint** -- State sach giua cac superstep, nen ban co the chup anh, luu tru va tiep tuc sau do.
3. **Kha nang to hop** -- Toan bo mo hinh thuc thi nam sau mot giao dien (`PregelProtocol`). Bat ky thu gi implement giao dien do -- do thi cuc bo, do thi tu xa, functional entrypoint -- deu co the duoc to hop nhu mot node.

Hau het nguoi dung khong tuong tac truc tiep voi Pregel. `StateGraph.compile()` va `@entrypoint` deu tao ra cac instance Pregel. Nhung hieu mo hinh giai thich *tai sao* nhung thu nhu streaming, interrupt va to hop tu xa hoat dong tron tru -- chung deu duoc xay dung tren cung mot mo hinh thuc thi superstep voi giao tiep qua channel.

---

## Y tuong cot loi

Engine thuc thi cua LangGraph (Pregel) va client tu xa (RemoteGraph) deu implement cung mot giao dien -- `PregelProtocol`. Dieu nay co nghia la ban co the ket noi mot do thi chay tren server o Tokyo vao mot orchestrator chay tren laptop cua ban, va no hoat dong giong het nhu mot ham cuc bo.

```python
from langgraph.graph import StateGraph, START, END
from langgraph.pregel.remote import RemoteGraph

# Day la cac HTTP client, khong phai code cuc bo
research = RemoteGraph("research-agent", url="https://research.example.com", name="research")
coder    = RemoteGraph("code-agent",     url="https://code.example.com",     name="coder")

# Nhung chung duoc ket noi nhu bat ky node nao khac
graph = StateGraph(MyState)
graph.add_node("research", research)
graph.add_node("coder", coder)
graph.add_edge(START, "research")
graph.add_edge("research", "coder")
graph.add_edge("coder", END)

app = graph.compile()
```

![PregelProtocol: Giao dien Toan cuc](/assets/images/02-pregel-protocol.png)

Do la tat ca. `invoke()`, `stream()`, interrupt, checkpointing -- moi thu hoat dong xuyen ranh gioi.

---

## Tai sao dieu nay quan trong

Hau het cac framework dieu phoi AI cho ban mot trong hai lua chon: moi thu chay trong mot tien trinh, hoac ban tu xu ly voi REST call va message queue.

LangGraph cho ban lua chon thu ba: xay dung moi agent nhu mot do thi doc lap voi deployment rieng, state rieng, team rieng so huu -- roi to hop chung voi cung API ban dung cho cac node cuc bo. Khong can glue code. Khong can serialization tuy chinh. Khong can logic retry tu viet.

Dieu nay mo khoa cac kien truc trong nhu he thong phan tan thuc su:

![Kien truc Da Do thi Phan tan](/assets/images/03-multi-graph-architecture.png)

Moi o co the trien khai doc lap qua `langgraph up`. Moi o co checkpoint storage rieng. Moi o co the duoc versioning va rollback doc lap. Orchestrator chi don gian ket noi chung lai.

---

## Xay dung Orchestrator

### Thiet ke State

Orchestrator quan ly state dieu phoi. Cac worker graph quan ly state domain rieng cua chung. Day la cac moi quan tam rieng biet.

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

Orchestrator khong can biet state noi bo cua research agent. No gui message vao, nhan ket qua tra ve. Truong `results` tich luy output tu nhieu worker su dung merge reducer.

### Dinh tuyen

Pipeline tinh la truong hop don gian nhat, nhung suc manh thuc su nam o dinh tuyen dong. Ban co the su dung conditional edge, LLM-based router, hoac fan-out pattern.

**Dinh tuyen co dieu kien** -- chon worker dua tren state:

```python
def route(state):
    if "code" in state["messages"][-1].content.lower():
        return "coder"
    return "research"

graph.add_conditional_edges("router", route)
```

**Fan-out voi Send** -- gui den nhieu worker song song:

```python
from langgraph.types import Send

def dispatch(state):
    return [Send(task["worker"], state) for task in state["plan"]]

graph.add_conditional_edges("planner", dispatch)
```

Moi `Send` kich hoat mot loi goi song song den remote graph dich. Ket qua duoc hop nhat lai thong qua state reducer. Day la map-reduce cho AI agent.

![Fan-Out / Map-Reduce voi Send()](/assets/images/06-fan-out-map-reduce.png)

---

## Streaming xuyen ranh gioi dich vu

Day la noi hau het cac cach tiep can DIY that bai. Khi ban long do thi xuyen ranh gioi HTTP, ban muon streaming hoat dong binh thuong -- output LLM token-by-token tu mot remote agent phai chay qua orchestrator den client ma khong bi buffer.

![Streaming xuyen ranh gioi dich vu](/assets/images/04-stream-propagation.png)

LangGraph xu ly dieu nay tu dong. Viec implement stream cua `RemoteGraph`:

1. Chuyen tiep cac stream mode cua parent den remote server
2. Them namespace prefix de ban biet subgraph nao phat ra su kien nao
3. Dan cac su kien nguoc len qua stream cua orchestrator

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

Output trong nhu:

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

Ca bay stream mode deu hoat dong xuyen ranh gioi: `values`, `updates`, `messages`, `custom`, `tasks`, `checkpoints`, va `debug`.

---

## Human-in-the-Loop xuyen do thi

Day la mot kich ban: coding agent cua ban tao mot database migration. Truoc khi chay, ban muon mot nguoi duyet. Coding agent la mot remote graph. Nguoi dung dang tuong tac voi client cua orchestrator.

![Human-in-the-Loop xuyen ranh gioi do thi](/assets/images/05-interrupt-flow.png)

Dieu nay hoat dong ngay. Khi mot remote graph goi `interrupt()`, interrupt duoc truyen len qua orchestrator den client:

```python
# Ben trong remote coding agent (trien khai rieng):
def migration_node(state):
    migration = generate_migration(state)
    approval = interrupt({"sql": migration, "question": "Run this migration?"})
    if approval["approved"]:
        run_migration(migration)
    return {"result": "Migration applied"}
```

Client cua orchestrator thay interrupt:

```python
state = app.get_state(config)
print(state.interrupts)  # [Interrupt(value={"sql": "ALTER TABLE...", "question": "..."})]

# Tiep tuc
app.invoke(Command(resume={"approved": True}), config=config)
```

Lenh resume chay nguoc qua orchestrator, qua `RemoteGraph`, den dung thread tren remote server. Khong can plumbing.

---

## Checkpointing doc lap

Moi do thi trong he thong checkpoint doc lap. Orchestrator luu cac quyet dinh dinh tuyen va ket qua tich luy. Moi remote graph luu lich su hoi thoai noi bo va tool call rieng. Chung khong chia se database.

Day la mot tinh nang, khong phai han che. No co nghia la:

- Ban co the kiem tra va phat lai workflow cua orchestrator ma khong can cham vao state cua worker
- Moi team kiem soat backend persistence rieng (SQLite cho dev, Postgres cho prod)
- Checkpoint storage duoc mo rong doc lap cho moi dich vu
- Time-travel debugging hoat dong o moi tang

```python
# Kiem tra lich su orchestrator
for snapshot in app.get_state_history(config, limit=5):
    print(f"Step {snapshot.metadata['step']}: next={snapshot.next}")

# Kiem tra lich su remote graph truc tiep
remote = RemoteGraph("research-agent", url="https://research.example.com")
for snapshot in remote.get_state_history(remote_config, limit=5):
    print(f"Step {snapshot.metadata['step']}: next={snapshot.next}")
```

---

## Xu ly loi

Cac loi goi tu xa co the that bai. Mang khong dang tin cay. LangGraph cung cap retry policy o cap node:

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

De kiem soat tot hon, boc loi goi tu xa:

```python
def resilient_research(state):
    try:
        return create_remote("research-v2").invoke(state)
    except RemoteException:
        return create_remote("research-v1").invoke(state)  # fallback ve phien ban truoc
```

Loi tu xa xuat hien duoi dang `RemoteException`. Interrupt xuat hien duoi dang `GraphInterrupt`. Command xuat hien duoi dang `ParentCommand`. Moi loai co ngu nghia xu ly rieng biet -- chung khong bi gop chung.

---

## Uy quyen phan cap

Cac remote graph co the gui lenh nguoc lai orchestrator cha su dung `Command(graph=Command.PARENT)`. Dieu nay cho phep cau truc team phan cap:

```
Orchestrator
+-- Team Lead A (remote)
|   +-- Worker A1 (remote)
|   +-- Worker A2 (remote)
+-- Team Lead B (remote)
    +-- Worker B1 (remote)
```

Mot worker co the leo thang len team lead. Mot team lead co the leo thang len orchestrator. Co che `ParentCommand` xu ly dieu nay o bat ky do sau long nao.

```python
# Worker ben trong mot remote team graph
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

## Trien khai

Moi do thi trien khai doc lap. File config `langgraph.json` dinh nghia diem vao cua do thi:

```json
{
    "python_version": "3.12",
    "dependencies": ["./requirements.txt"],
    "graphs": {
        "research-agent": "./graphs/research.py:graph"
    }
}
```

Sau do trien khai:

```bash
langgraph up --port 8001
```

Hoac su dung `langgraph dev` de phat trien cuc bo voi hot-reloading. Trong moi truong phat trien, tat ca remote graph co the chay tren localhost voi cac port khac nhau. Trong production, chung la cac dich vu rieng biet sau cac URL rieng biet.

Ban than orchestrator cung co the duoc trien khai theo cach tuong tu, hoac chay nhu mot script cuc bo. No chi la mot do thi tinh co goi cac do thi khac qua HTTP.

---

## Quan sat he thong

Bat `distributed_tracing=True` tren cac instance `RemoteGraph` de co trace thong nhat trong LangSmith xuyen tat ca dich vu:

```python
research = RemoteGraph(
    "research-agent",
    url="https://research.example.com",
    distributed_tracing=True,
)
```

Mot trace hien thi toan bo hanh trinh: orchestrator dinh tuyen -> research agent tim kiem -> coding agent tao code -> reviewer kiem tra. Phan tich do tre, so luong token, va quy loi xuyen toan bo he thong phan tan.

Ban cung co the truc quan hoa toan bo topology do thi:

```python
app.get_graph(xray=2).draw_mermaid_png()
```

Dieu nay render do thi orchestrator voi noi bo remote subgraph duoc mo rong den do sau 2.

---

## Khi nao nen su dung

**Su dung dieu phoi da do thi khi:**

- Cac team khac nhau so huu cac agent khac nhau va can chu ky trien khai doc lap
- Ban can mo rong cac agent doc lap (research agent can GPU, router thi khong)
- Ban muon versioning va rollback cac agent doc lap
- He thong cua ban co ranh gioi domain tu nhien (research vs. coding vs. review)
- Ban can cach ly loi -- mot agent bi crash khong nen lam sap toan bo he thong

**Giu mot do thi duy nhat khi:**

- Mot team so huu moi thu
- Cac agent chia se hau het state
- Chi phi cua HTTP call khong dang de doi lay loi ich kien truc
- Ban dang lam prototype va toc do lap lai quan trong hon tinh linh hoat trien khai

---

## Ket luan

Quyet dinh thiet ke cot loi trong LangGraph la `PregelProtocol` la giao dien toan cuc. Mot `StateGraph` da compile, mot ham `@entrypoint`, va mot HTTP client `RemoteGraph` deu implement no. Dieu nay co nghia la to hop la mien phi -- ban khong phai tra thue truu tuong khi chuyen sang phan tan.

Xay dung cac agent cua ban nhu cac do thi doc lap. Trien khai chung rieng re. Ket noi chung voi mot orchestrator coi moi cai nhu chi la mot node binh thuong. Streaming, interrupt, checkpointing, va xu ly loi hoat dong xuyen ranh gioi ma khong can them code.

Khoang cach giua "no chay tren laptop cua toi" va "no chay trong production xuyen nam dich vu" nho hon ban nghi.
