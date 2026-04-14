---
layout: post
title: "Ý Tưởng Của Bạn Đều Hay — Bạn Chỉ Thiếu Tests"
description: "AI generate code rất giỏi. Ideas tốt, code chạy được, logic hợp lý. Vấn đề thật không phải AI sai — mà là bạn không có đủ tests để verify nó làm đúng ý bạn."
featured: true
lang: vi
ref: y-tuong-cua-ban-deu-hay-ban-chi-thieu-tests
permalink: /vi/y-tuong-cua-ban-deu-hay-ban-chi-thieu-tests/
date: 2026-04-04
---

# Ý Tưởng Của Bạn Đều Hay — Bạn Chỉ Thiếu Tests

> *"Chạy được" khác "chạy đúng ý bạn."*

---

Tôi dispatch task cho agents hàng ngày. Blogger viết bài, vn-trader phân tích cổ phiếu, claude-forge build plugin, workstream chạy workflows phức tạp.

Và sau vài tháng làm điều này, tôi nhận ra một pattern rõ ràng: **những tasks có test suite cho kết quả tốt hơn hẳn những tasks chỉ có prompt mô tả.**

Không phải vì AI của một task thông minh hơn. Không phải vì prompt của task kia được viết tốt hơn. Mà đơn giản vì một bên có cách để *verify* — và bên kia thì không.

---

## AI Generate Code Giỏi Hơn Bạn Nghĩ

Đây là điều tôi muốn nói trước: **AI generate code rất tốt.** Ideas hợp lý, structure clean, logic đúng, edge cases thường được handle. Với Claude 3.5/3.7 Sonnet, tôi liên tục bị impress bởi chất lượng code được generate ra.

Vấn đề không phải AI dốt.

Vấn đề là: AI không đọc được suy nghĩ của bạn.

Bạn có một mental model về cái function đó phải làm gì. AI có một interpretation dựa trên prompt của bạn. Hai thứ này *gần giống nhau* — nhưng "gần" trong software engineering là nguồn gốc của rất nhiều bug.

```python
# Bạn muốn: parse user input, strip whitespace, return None nếu empty
# AI generate:
def parse_input(text):
    return text.strip() or None
```

Code này đúng không? Đúng. Code này chạy được không? Chạy được. Code này làm *đúng ý bạn* không?

Phụ thuộc vào thứ bạn không nói rõ: `"0"` trả về gì? `" "` trả về gì? `False` input xử lý thế nào?

AI không cố tình sai. AI chỉ đang fill gaps dựa trên assumption của nó. Và assumptions của AI không phải lúc nào cũng match với cái bạn thực sự muốn.

---

## Pattern Thất Bại Điển Hình

Đây là workflow mà tôi thấy rất nhiều người (bao gồm cả tôi trước đây) dùng:

```
1. Có idea
2. Viết prompt cho AI
3. AI generate code
4. Manual check: "trông có vẻ ok"
5. Deploy
6. Bug ở production
7. "AI generate sai rồi"
```

Bước 4 là vấn đề. "Trông có vẻ ok" không phải test. Đó là gut feeling.

Gut feeling fail theo những cách rất predictable:
- Bạn chỉ test happy path, không test edge cases
- Bạn test những gì bạn *nhớ* là cần test, không phải tất cả những gì cần test
- Bạn bị confirmation bias — muốn nó work nên bạn thấy nó work
- Bạn không có hệ thống để phát hiện regression khi code thay đổi sau đó

Kết quả: code "ok" deploy lên production, rồi fail ở một case cụ thể mà không ai nghĩ đến.

---

## Khi Có Tests: Cùng AI, Khác Kết Quả

Ngược lại, khi tôi dispatch task với test suite đi kèm:

```
1. Có idea
2. Viết tests định nghĩa expected behavior
3. Dispatch cho AI: "implement X sao cho tests pass"
4. AI generate code
5. Run tests → pass hoặc fail với message rõ ràng
6. Nếu fail: AI fix, re-run
7. Done — confidence cao
```

Không có "trông có vẻ ok". Có pass/fail. Rõ ràng, không cần đoán mò.

Với bridge-bot system của tôi, tôi bắt đầu dùng loop pattern khi dispatch:

```
"Implement function X. Run pytest. Fix until all tests pass."
```

Agent tự loop, tự verify, tự fix. Nhưng pattern này chỉ work khi **tests đã được viết sẵn và đủ tốt**. Không có tests = agent loop mù, không có gì để verify cả.

---

## Tests Là Spec, Không Phải Afterthought

Đây là mindset shift quan trọng nhất: **tests không phải thứ bạn viết *sau* khi code xong để "kiểm tra cho chắc."**

Tests là *specification* của behavior mà bạn muốn.

Khi bạn viết test trước, bạn buộc phải trả lời những câu hỏi mà bạn thường bỏ qua:

- Input này trả về gì?
- Error case này xử lý thế nào?
- State sau operation này là gì?
- Đây có side effect không?

Những câu hỏi này thường chỉ được hỏi khi production fail. Nếu bạn hỏi *trước* — via tests — bạn đã viết spec xong.

Và một khi bạn có spec dưới dạng tests, **bạn communicate intent với AI rõ ràng hơn bất kỳ prose prompt nào:**

```python
def test_parse_user_id():
    assert parse_user_id("user_123") == 123
    assert parse_user_id("  user_456  ") == 456  # strip whitespace
    assert parse_user_id("invalid") is None       # return None, not raise
    assert parse_user_id("") is None              # empty string → None
    assert parse_user_id("user_0") == 0           # zero is valid
```

Nhìn vào 5 test cases này, AI biết *chính xác* function cần làm gì. Không có ambiguity. Không có assumption. Prose prompt "viết function parse user ID từ string" không bao giờ convey được đủ thông tin như vậy.

---

## TDD With AI: Workflow Thực Tế

Tôi không phải TDD purist. Tôi không viết tests cho mọi thứ từ đầu đến cuối. Nhưng khi làm việc với AI để generate code, TDD-style workflow có ROI rõ ràng.

**Flow tôi dùng:**

```
1. Define interface: function signature, input/output types
2. Viết test cases từ acceptance criteria
   - Happy paths
   - Edge cases tôi biết trước
   - Error cases
3. Dispatch cho agent: "Implement để pass tests này"
4. Agent generate + tự verify + tự fix nếu cần
5. Review code và tests để catch gì AI miss
```

Bước 5 quan trọng — không phải vì AI sai, mà vì *tôi* có thể đã quên test case nào đó quan trọng. Review code với tests đã pass là dễ hơn review code không có gì.

Một ví dụ thực từ claude-forge project: tôi cần function parse plugin metadata từ CLAUDE.md file. Thay vì viết prompt dài mô tả format, tôi viết tests:

```python
def test_parse_plugin_metadata():
    content = """---
name: my-plugin
version: 1.0.0
---
# My Plugin
description here
"""
    meta = parse_plugin_metadata(content)
    assert meta["name"] == "my-plugin"
    assert meta["version"] == "1.0.0"
    assert meta["description"] == "description here"

def test_parse_plugin_metadata_missing_version():
    content = """---
name: my-plugin
---
"""
    meta = parse_plugin_metadata(content)
    assert meta.get("version") is None  # graceful, không raise

def test_parse_plugin_metadata_invalid_yaml():
    content = "not yaml frontmatter at all"
    assert parse_plugin_metadata(content) is None
```

Agent nhận tests này, implement function, pass hết. Tôi không cần viết prompt giải thích YAML frontmatter format, edge cases, hoặc error handling. Tests nói hết rồi.

---

## Test Infrastructure Là Guardrails Cho AI

Có một analogy tôi thích: tests là guardrails trên đường núi.

Không có guardrails: driver có thể chạy đúng đường hoàn toàn — nếu họ biết đường, nếu điều kiện tốt, nếu không có distraction. Nhưng khi có một yếu tố sai, không có gì chặn xe lại.

Có guardrails: driver vẫn có thể làm mọi việc như bình thường, nhưng nếu lạc đường, guardrails catch lại trước khi fall off cliff.

Tests là guardrails cho AI-generated code. AI có thể và thường xuyên đi đúng hướng. Nhưng khi nó lạc — và nó *sẽ* lạc — tests catch lại trước khi bug reach production.

Điều này đặc biệt quan trọng với **regression**. Bạn dispatch task lần 1, AI generate code tốt, tests pass. Hai tuần sau, bạn dispatch task khác liên quan, AI modify code, tests catch regression ngay lập tức. Không có test suite, bạn chỉ biết regression khi production fail hoặc khi manual test catch được — nếu may mắn.

---

## Câu Hỏi Thật: Bạn Đang Invest Vào Đâu?

Tôi thấy nhiều người spend nhiều thời gian vào:
- Viết prompt tốt hơn
- Tìm model tốt hơn
- Tune agent behavior
- Cải thiện context management

Nhưng lại ít đầu tư vào test infrastructure.

Điều này backwards.

**Better prompt** giúp AI generate code gần hơn với ý bạn muốn — nhưng không verify nó đúng.

**Better tests** verify behavior thực sự — và làm cho prompt ít quan trọng hơn vì AI có feedback loop rõ ràng để self-correct.

Tôi không nói bỏ hết đầu tư vào prompt engineering. Nhưng nếu bạn phải chọn giữa "spend 2 giờ tinh chỉnh prompt" và "spend 2 giờ viết test suite tốt" — trong hầu hết cases, test suite cho ROI cao hơn.

Lý do đơn giản: **prompt tốt giúp lần này. Test suite tốt giúp mọi lần sau đó.**

---

## Không Phải Mọi Thứ Đều Cần Tests

Tôi muốn nói thẳng để tránh misunderstanding: không phải mọi task AI đều cần test suite.

Nếu bạn dùng AI để:
- Draft email
- Explain concept
- Brainstorm ideas
- Tóm tắt document
- Viết blog post (như bài này)

...thì test suite không applicable. Không có objective pass/fail cho "bài viết hay."

Tests matter khi bạn generate **code có behavior cụ thể** — logic, data transformation, API endpoints, algorithms. Khi output có thể được defined as correct/incorrect, tests có giá trị.

Heuristic đơn giản: nếu output của AI task có thể fail theo cách silent và costly, viết tests. Nếu không, đừng overthink.

---

## Thực Hành Bắt Đầu Từ Đâu?

Nếu bạn đang bắt đầu với AI-assisted development và muốn build test hygiene:

**Bước 1: Viết tests *trước* khi dispatch bất kỳ logic nào cho AI**

Dù chỉ 3-5 test cases. Better than nothing. Force yourself to answer: "Khi nào function này *đúng*? Khi nào nó *sai*?"

**Bước 2: Dùng "fix until tests pass" pattern**

```
"Implement function X. Tests là ở file test_x.py. 
Run pytest. Fix cho đến khi tất cả pass."
```

Đây là instruction rõ nhất bạn có thể cho AI. Không cần describe behavior dài dòng — tests đã describe rồi.

**Bước 3: Add test cases khi tìm ra bug**

Mỗi khi production bug xuất hiện: fix nó, rồi viết test cover case đó. Dần dần test suite của bạn sẽ catch những gì thực tế fail, không phải những gì bạn tưởng tượng có thể fail.

**Bước 4: Invest vào test speed**

Slow tests = tốn thời gian chờ = ít ai muốn run. Fast tests = feedback loop nhanh = AI có thể loop-and-fix hiệu quả. Setup test infrastructure sao cho run trong seconds, không phải minutes.

---

## Kết

Vấn đề với AI-generated code không phải là AI kém. AI khá good, thực ra rất good.

Vấn đề là chúng ta đang fly blind — không có cách systematic để biết code làm đúng thứ mình muốn hay không.

Tests là cách fix điều đó. Không phải vì tests là silver bullet, mà vì tests là thứ duy nhất cho bạn feedback loop rõ ràng: pass = đúng, fail = sai, message = biết chỗ nào sai.

Và khi bạn có feedback loop rõ ràng, AI không chỉ generate code — nó có thể *verify và self-correct*. Đó là khi agentic workflow thực sự work.

Ý tưởng của bạn đều hay. Code của AI thường tốt. Cái thiếu là cách biết hai thứ đó khớp nhau.

Đó là tests.

---

*Tôi vận hành bridge-bot system với ~15 agents, dispatch hàng chục tasks mỗi ngày. Mọi observation trong bài đến từ experience trực tiếp — không phải benchmark, không phải paper.*

---

*Nếu bài viết có ích, kết nối với mình trên [LinkedIn](https://www.linkedin.com/in/hieu-tran-data/) hoặc xem thêm các project tại [GitHub](https://github.com/hieutrtr/).*
