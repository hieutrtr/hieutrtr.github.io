---
layout: post
title: "Từ BMAD sang Claude Plugin Composition — Giảm 99% Token Burn"
description: "Câu chuyện về việc hay bị quota limit giữa tháng với multi-agent systems và phát hiện ra architecture patterns giúp token efficiency tăng 3×."
featured: true
lang: vi
ref: bmad-vs-claude-plugins
permalink: /vi/bmad-vs-claude-plugins/
date: 2026-02-27
---

# Từ BMAD sang Claude Plugin Composition — Cuộc hành trình giảm 99% token burn 🚀

*Câu chuyện về việc hay bị quota limit giữa tháng với multi-agent systems và phát hiện ra architecture patterns giúp token efficiency tăng 3×.*

---

## Mở đầu: Khi quota limit đánh bạn giữa tháng

Vài tháng trước, tui đang ở trạng thái "AI coding productivity peak" với **BMAD-METHOD** (Build More Architect Dreams). 

Workflow đỉnh:
- ✅ Planning có cấu trúc: Ideation → PRD → Architecture → Implementation
- ✅ 12+ specialized agents: PM, Architect, Developer, UX, Scrum Master...
- ✅ Party Mode: Nhiều agents cùng collaborate trong một session
- ✅ Scale-adaptive: Bug nhỏ plan nhẹ, enterprise platform plan sâu

**Cảm giác như làm việc với một team thật.**

Nhưng rồi nhận ra hai điều đau đớn:

1. **Chạy lâu** — Mỗi lần planning, agents discuss qua lại, nhiều turns → đợi mỏi tay
2. **Cháy token như đốt tiền** — 2 projects song song với BMAD, **hay bị quota limit giữa tháng** (Claude max $200/tháng nhưng token burn rate cao)

Thế là phải chờ reset quota, mất continuity, làm lại... vòng lặp vô tận.

## Chuyển qua Claude Skills — "Chắc cũng thế thôi"

Dạo gần đây công ty ai cũng xôn xao về **Claude Skills**. Sếp A viết skill meeting notes, sếp B share skill review code, team C có skill domain knowledge riêng.

Tui nghĩ "chắc cũng giống mấy tool khác" — nhưng thử vài tuần thì **shocked**.

**Cùng workload, ít khi bị quota limit giữa tháng.**

Tui phải ngồi research kỹ: **Tại sao token economy khác đến vậy?**

## Khám phá #1: BMAD và Claude Plugins không phải competitors

Lúc đầu tui cứ nghĩ đây là cuộc đua "BMAD vs Claude Code" — ai tốt hơn?

**Nhầm to.**

Chúng ở **hai layers hoàn toàn khác nhau:**

| Aspect | BMAD-METHOD | Claude Code Plugins |
|--------|-------------|---------------------|
| **Essence** | Methodology (how you work) | Infrastructure (what you build with) |
| **Focus** | Workflow structure | Architecture patterns |
| **Agents** | Share context, collaborate | Isolated context, parallel work |
| **Token model** | All-in-one context | Layered isolation |
| **Purpose** | Dạy bạn cách làm việc với AI | Cung cấp công cụ để làm |

**BMAD** dạy bạn **how to work**.  
**Claude Code** cung cấp **tools to work with**.

→ Không phải chọn một bỏ một. **Có thể dùng cả hai.**

## Khám phá #2: Separation of Concerns — Mỗi extension một trách nhiệm

Sau mấy tuần dùng, tui nhận ra Claude Code có **5 loại extensions**, mỗi thứ handle **ĐỰC MỘT** concern:

### 1. Skills — Kiến thức

**Trách nhiệm:** Claude biết gì, nên làm gì  
**Context cost:**
- Model-invocable: ~50-100 tokens/skill at startup
- Action-only với `disable-model-invocation: true`: **Zero cost**

**Khi nào dùng:** Reference knowledge (patterns, conventions, best practices)

### 2. Subagents — Isolated workers

**Trách nhiệm:** Làm việc nặng mà không làm lộn context chính  
**Context cost:** **Zero to parent** (isolated context)  
**Return:** Summary (~500 tokens), không phải cả 50K tokens output

**Khi nào dùng:** Context-heavy work (review 30 files, analyze logs, security scan)

### 3. Hooks — Enforcement

**Trách nhiệm:** Bắt buộc 100%, không cần LLM judgment  
**Context cost:** **Zero** (external script execution)  
**Khi nào dùng:** Deterministic automation (lint, test, security gates)

**Key insight:** Skill có thể bị "quên" sau compaction → Hook vẫn fire 100%

### 4. MCP — External connectivity

**Trách nhiệm:** Connect database, monitoring, APIs  
**Context cost:** ~500-1,500 tokens/server schemas  
**Optimization:** Tool Search để defer unused tools

**Khi nào dùng:** External system integration

### 5. Commands — Entry points

**Trách nhiệm:** Quick triggers (`/deploy`, `/review`)  
**Context cost:** ~100-200 tokens descriptions  
**Khi nào dùng:** User-facing workflow shortcuts

---

**Bài học đau đớn:**

Khi tui nhét logic enforcement vào skill → skill dài, context phình  
Khi tui nhét kiến thức vào CLAUDE.md → mỗi request đều load  
Khi tui để workflow phức tạp trong main context → token burn nhanh

**Giải pháp: Mỗi thứ đúng chỗ của nó.**

## Khám phá #3: Composition Patterns — Cách chúng wire lại với nhau

Đây mới là **game changer**. Không phải dùng riêng lẻ — chúng **reference lẫn nhau** tạo ra value mà standalone không có.

### Pattern 1: Skill + MCP Companion

**Problem:** Lúc đầu tui add MCP server (database connector) — Claude biết tool tồn tại nhưng dùng như shit.

**Solution:** Viết companion skill:

```yaml
# skills/database-patterns/SKILL.md
name: database-patterns
description: Production database query patterns and optimization

Content:
- Query patterns cho production DB
- Schema design conventions
- Index strategies
- Performance optimization tips
```

→ **Boom.** Claude biết tool (MCP) + biết cách dùng hiệu quả (Skill).

**MCP = capability, Skill = knowledge.**

---

### Pattern 2: Skill + Subagent (isolated context)

**Problem:** Workflow deploy phức tạp: pre-check → canary → monitor → rollout.

Lúc trước viết hết vào skill → mỗi lần deploy:
- Claude load cả workflow vào context
- Read 50 files logs
- Analyze metrics
→ Main context ăn **60K tokens**

**Solution:** Skill với `context: fork`

```yaml
# skills/deploy/SKILL.md
name: deploy
context: fork  # ← Chạy trong subagent
description: Multi-stage deployment with pre-checks and monitoring

Workflow:
1. Pre-flight checks
2. Canary deployment
3. Monitor metrics
4. Progressive rollout
```

→ Subagent spawn, đọc 50 files (40K tokens **ở context riêng**), return **500-token summary** cho parent.

**Main context: 500 tokens instead of 60K.**

---

### Pattern 3: Subagent + Preloaded Skills

**Problem:** Subagent review security — mỗi lần spawn phải dạy lại "OWASP Top 10", "SQL injection patterns"...

**Solution:** Preload skills

```yaml
# agents/security-reviewer.md
name: security-reviewer
model: sonnet
tools:
  - Read
  - Grep
  - Bash(semgrep *)
skills:
  - security-patterns
  - owasp-top-10

Instructions:
Review code for OWASP Top 10 vulnerabilities...
```

→ Subagent **start với expertise baked in**. Không phải dạy lại.

---

### Pattern 4: Skill ADVISES, Hook ENFORCES

**Problem:** Tui viết trong skill: "Hãy nhớ chạy lint trước commit"  
→ Sau 20 messages, Claude compaction → **quên**.

**Solution:** Separation

**Skill:**
```yaml
# skills/code-quality/SKILL.md
Best practices:
- Run ESLint before commit
- Fix all warnings
- Format with Prettier
```

**Hook:**
```json
// hooks/hooks.json
{
  "PostToolUse": [{
    "matcher": "Edit|Write",
    "hooks": [{
      "type": "command",
      "command": "${CLAUDE_PLUGIN_ROOT}/scripts/lint.sh",
      "async": true,
      "timeout": 30
    }]
  }]
}
```

→ Skill = advisory (probabilistic)  
→ Hook = enforcement (deterministic 100%)

**Claude có thể quên skill, hook vẫn fire.**

## Khám phá #4: Context Budget Architecture — Con số thực tế

Sau khi setup plugin đầy đủ cho project:

**Components:**
- 5 skills (3 model-invocable, 2 action-only)
- 3 subagents (security, compliance, performance)
- 4 hooks (PreToolUse block, PostToolUse lint, Stop verify, audit log)
- 3 MCP servers (GitHub, PostgreSQL, Datadog)

Tui chạy `/context` check token breakdown:

```
Skills (model-invocable):       240 tokens
MCP server schemas:           2,000 tokens
Skills (action-only):             0 tokens
Subagents:                        0 tokens
Hooks:                            0 tokens
─────────────────────────────────────────
Total startup cost:           2,240 tokens (<1.5% of 200K)
```

**Wait, what?** 5 skills + 3 MCPs + 3 subagents + 4 hooks = chỉ **2.2K tokens**?

**Vì:**
- Skills action-only: **zero** (`disable-model-invocation: true`)
- Subagents: **zero** (isolated context, zero cost to parent)
- Hooks: **zero** (external execution)
- Chỉ có 3 reference skills + 3 MCP schemas

### So với BMAD Architecture

**BMAD: Shared Context Pool**

```
[User] → [BMAD Framework]
           ↓
    [Shared Context Pool]
      ↙     ↓     ↘
   [PM]  [Arch]  [Dev]  [UX]  [QA] ...
     ↘     ↓      ↓     ↙
       [All agents see full history]
       [Each turn adds to shared context]
       
Base cost: 12 agents × 5K tokens each = 60K+ tokens
```

**✅ Pro:** Agents collaborate, discuss, reach consensus  
**❌ Con:** Context pollution — 12 agents = 12× memory footprint

---

**Claude Plugins: Layered Isolation**

```
[User] → [Main Context] ← Skills (2.2K)
           ↓                ← MCP schemas (3K)
      [Decision Layer]
           ↓
    ┌──────┴──────┐
    ↓             ↓
[Subagent 1]  [Subagent 2]  ← ISOLATED contexts
(Review)      (Analysis)
    ↓             ↓
 [Summary]    [Summary]  ← Return 500 tokens each
    └──────┬──────┘
           ↓
    [Main Context] ← Receives summaries only
           ↓
       [Hooks]  ← Enforce rules (zero cost)

Base cost: 2.2K tokens startup
Work cost: In isolated contexts
```

**✅ Pro:** Context isolation — subagents burn own tokens, return summaries  
**❌ Con:** Less "discussion" between agents (but do you need PM + Architect arguing in 60K tokens?)

## Real Example: Code Review Workflow

### Trước (BMAD-style approach)

```
Review PR:
→ Load security knowledge           5,000 tokens
→ Load compliance rules             8,000 tokens
→ Read 30 modified files           40,000 tokens
→ Run analysis in main context     15,000 tokens output
──────────────────────────────────────────────────
Total in main context:             68,000 tokens
```

**Mỗi PR review = 68K tokens.**  
Làm 10 PRs → context đầy → reset → mất continuity.

---

### Giờ (Claude Plugin composition)

```
Review PR:
→ Main: Load security-patterns skill                  80 tokens
→ Spawn security-reviewer subagent:
    ├─ Preloaded skills (security-patterns, owasp)
    ├─ Read 30 files (40K tokens in SUBAGENT context)
    ├─ Run semgrep (output in SUBAGENT context)
    └─ Return: 500-token summary to parent
→ PostToolUse hook runs ESLint (zero cost — external)
→ Main context: Receive summary, discuss fix
──────────────────────────────────────────────────
Total in main context:                              580 tokens
```

**Token savings: 68,000 → 580 = 99% reduction**

Review 100 PRs mà main context vẫn còn room.

## Plot Twist: Kết hợp cả hai!

Sau vài tuần research, tui phát hiện ra:

**BMAD ≠ competitor của Claude Code.**  
**Chúng là complementary.**

**BMAD = Methodology** (how to work)  
**Claude Plugins = Infrastructure** (what to work with)

→ **Có thể kết hợp!**

### Workflow tui đang chạy

**Phase 1: Planning (BMAD approach)**
- PM + Architect agents collaborate trong main context
- Generate PRD, Architecture Document
- Discuss trade-offs, reach consensus
- **Use case:** Cần discussion, brainstorming, alignment

**Phase 2: Implementation (Plugin composition approach)**
- `/deploy` skill → triggers deployment workflow
- Security-reviewer **subagent** → isolated review (40K tokens trong subagent, return 500-token summary)
- Perf-analyzer **subagent** → isolated metrics analysis
- PostToolUse **hooks** → enforce lint/test/format automatically
- MCP servers → connect GitHub, Datadog, PostgreSQL

**→ Best of both worlds:**
- ✅ Structured planning từ BMAD
- ✅ Token-efficient execution từ Claude Plugins

## Tại sao Composition thay đổi game

Sau khi research architecture kỹ, tui hiểu:

**Well-designed system ≠ nhét nhiều features vào.**

**Well-designed system = mỗi component handle đúng một concern, và composition tạo ra value mà standalone không có.**

### Comparison Table

| Concern | BMAD | Claude Plugins | Winner |
|---------|------|----------------|--------|
| **Multi-agent collaboration** | In shared context | Via subagent summaries | BMAD cho planning, Plugins cho execution |
| **Deterministic enforcement** | Prompt agents to check | Hooks fire 100% | Plugins (không phụ thuộc LLM) |
| **External connectivity** | LLM reads/writes manually | MCP native protocol | Plugins (built-in) |
| **Context budget** | All agents share | Isolated + zero-cost layers | Plugins (99% reduction) |
| **Team distribution** | Copy prompt files | Plugin packaging + marketplace | Plugins (versioning, updates) |
| **Workflow structure** | Built-in (Agile phases) | DIY | BMAD (proven methodology) |

## Token Economics: Con số thực tế

### Trước (BMAD-only)

```
Monthly usage với Claude max $200:
- 2 projects song song
- ~100K tokens/week/project
- 200K tokens/week total
─────────────────────────────
Quota: ~8M tokens/month ($200)
Weekly burn: 200K tokens
→ Tháng 1-2: OK
→ Tháng 3: Hay bị quota limit tuần 3-4
→ Phải chờ reset quota đầu tháng
```

### Giờ (BMAD methodology + Claude Plugin infrastructure)

```
Monthly usage với Claude max $200:
- 2 projects
- Planning phase: ~30K tokens/week (BMAD approach)
- Execution phase: ~40K tokens/week (Plugin composition)
  ├─ Main context: 10K tokens (decisions, discussions)
  └─ Subagents: 30K tokens (isolated, cheap Haiku)
─────────────────────────────
Total: ~70K tokens/week
Weekly burn giảm: 200K → 70K (65% reduction)
→ Ít khi bị quota limit giữa tháng
→ Context efficiency → less resets
→ Subagents on Haiku (1/5 cost of Sonnet)
```

**65-70% reduction in token burn rate = ít bị quota limit.**

## Những bài học đau đớn

### ❌ Lỗi #1: Nhét enforcement logic vào Skills

**Problem:** Tui viết skill dạng "Nhớ chạy lint, nhớ chạy test, nhớ check security..."  
→ Skill dài, context phình, và sau compaction → Claude **quên**.

**Fix:** Skill = advisory, Hook = enforcement.

---

### ❌ Lỗi #2: Làm context-heavy work trong main context

**Problem:** Review 30 files code, đọc hết vào main context → 40K tokens.  
→ Làm vài lần → context đầy → reset.

**Fix:** Subagent với `context: fork` — work trong isolated context, return summary.

---

### ❌ Lỗi #3: Tất cả skills đều model-invocable

**Problem:** 12 skills, tất cả model-invocable → 1,200 tokens startup.  
→ Claude "confused", activate sai skill.

**Fix:** Chỉ giữ 3-4 reference skills model-invocable, còn lại `disable-model-invocation: true`.

---

### ❌ Lỗi #4: Không dùng MCP companion skills

**Problem:** Add MCP server → Claude biết tool nhưng dùng không hiệu quả.

**Fix:** Viết companion skill document query patterns, best practices.

## Kết luận: Không phải either/or

Sau vài tuần thử nghiệm và research, kết luận của tui:

### BMAD-METHOD

**✅ Strengths:**
- Structured workflow (Ideation → Planning → Architecture → Implementation)
- Proven Agile methodology
- Multi-agent collaboration for planning
- Scale-adaptive complexity

**❌ Weaknesses:**
- Shared context → token burn
- No built-in enforcement layer
- Distribution via copy-paste prompt files

**→ Best for:** Planning phase, brainstorming, architecture discussions

---

### Claude Code Plugins

**✅ Strengths:**
- Context isolation (subagents)
- Zero-cost enforcement (hooks)
- Native external connectivity (MCP)
- Token-efficient execution
- Team distribution (plugin packaging)

**❌ Weaknesses:**
- No built-in methodology (DIY workflow)
- Learning curve (5 extension types)
- Newer ecosystem

**→ Best for:** Implementation phase, automation, team standardization

---

### The Winning Combination

**Use BMAD methodology ON TOP OF Claude Plugin infrastructure:**

1. **Planning:** BMAD agents collaborate → PRD, architecture
2. **Execution:** Claude plugins → skills, subagents, hooks, MCP
3. **Enforcement:** Hooks ensure 100% compliance
4. **Distribution:** Plugin packaging for team consistency

**Result:**
- ✅ Structured planning từ BMAD
- ✅ Token-efficient execution từ Plugins
- ✅ Deterministic enforcement từ Hooks
- ✅ Team consistency từ Plugin distribution
- ✅ 3× token efficiency improvement

## Call to Action

Nếu bạn đang:
- ❌ Cháy token nhanh với multi-agent systems
- ❌ Context window đầy hoài, phải reset liên tục
- ❌ Không enforce được rules 100% (post-compaction forgetting)
- ❌ Muốn standardize tooling cho team

→ **Thử Claude Plugin composition patterns.**

Không phải bỏ BMAD hay methodology hiện tại — dùng methodology đó **trên** Claude Plugin infrastructure.

**Planning với structure, execution với efficiency.** 🚀

---

**P/S:** Tui không phải fanboy Anthropic. Đây là kết quả sau khi đốt $200/tháng, research kỹ architecture, và phát hiện patterns giúp token efficiency tăng 3×. Chia sẻ thật với ai đang đau đầu vì token burn. 😅

---

## Resources

- **BMAD-METHOD:** [https://docs.bmad-method.org/](https://docs.bmad-method.org/)
- **Claude Code Docs:** [https://code.claude.com/docs/](https://code.claude.com/docs/)
- **GitHub:** [https://github.com/bmad-code-org/BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD)

---

*Bài viết này dựa trên kinh nghiệm thực tế sau vài tuần research và testing. YMMV (Your Mileage May Vary) — nhưng nếu token economics là concern của bạn, composition patterns này đáng thử.*

---

*Nếu bài viết có ích, kết nối với mình trên [LinkedIn](https://www.linkedin.com/in/hieu-tran-data/) hoặc xem thêm các project tại [GitHub](https://github.com/hieutrtr/).*
