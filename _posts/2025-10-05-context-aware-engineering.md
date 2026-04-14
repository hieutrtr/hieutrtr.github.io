---
layout: post
title: "From Context Anxiety to Context-Aware Engineering"
description: "How context-aware loops turn large language model limitations into reliable agent design patterns."
featured: false
lang: en
ref: context-aware-engineering
permalink: /context-aware-engineering/
banner: /assets/images/context-anxiety-banner.svg
banner_alt: "Illustration of a glowing brain surrounded by signal waves and a DNA helix"
date: 2025-10-05
---

# 🧠 From Context Anxiety to Context-Aware Engineering

Turning LLM Limits into a Practical Design Pattern

_(Inspired by Cognition’s Devin and Anthropic’s Sonnet 4.5)_

> How two engineering blogs from late 2025 quietly defined a new discipline for building long-horizon AI agents.

---

## 🏗️ Abstract

Even with context windows approaching a million tokens, large language models (LLMs) still struggle when their attention budget fills up.
Two key engineering posts published in Sept–Oct 2025 —

- **Anthropic** — “Effective Context Engineering for AI Agents” (Sept 2025)
- **Cognition AI** — “Rebuilding Devin for Sonnet 4.5: Lessons and Challenges” (Oct 2025)

— both revealed a consistent pattern: LLMs like Sonnet 4.5 start summarizing, externalizing notes, and closing tasks early when they think they’re near the edge of their context window.

That behavior, nicknamed **context anxiety**, turns out to be more feature than bug — if you know how to design around it.

---

## 1️⃣ The Observation: Context Anxiety & Underestimation

From Cognition AI’s field tests with Devin:

- Sonnet 4.5 acts as if it knows when memory is running low — but it underestimates remaining space.
- This causes premature summarization and early task termination.
- It even starts writing files like `SUMMARY.md` or `CHANGELOG.md` to dump thoughts externally.

Cognition added both start- and end-of-prompt reminders:

> “Don’t summarize until you’ve completed all acceptance checks.”

A trick that helped: enabling 1 M-token mode but capping at ≈ 200 K, giving the model the impression of more headroom.

These behaviors match Anthropic’s warning that adding more tokens doesn’t always help — LLMs degrade once the signal-to-noise per token drops.

---

## 2️⃣ The Idea: Treat Anxiety as a Signal, Not a Bug

When an LLM starts summarizing, that’s a heartbeat.
Instead of ignoring it, an orchestrator can listen to that signal and trigger a new loop:

1. **Checkpoint** – capture the current reasoning state.
2. **Compact** – summarize key knowledge into structured memory.
3. **Re-seed** – start a new, focused reasoning episode (a sub-context).

This turns context limits into natural milestones instead of silent failure points.

---

## 3️⃣ What Is a Sub-Context?

A sub-context is a bounded reasoning sandbox: one coherent module inside a larger project.

| Sub-Context | Purpose | Example Tokens |
|-------------|---------|----------------|
| `auth_module` | Build login flow | 20 K |
| `test_module` | Write tests | 15 K |
| `docs_module` | Generate API docs | 8 K |

Each sub-context starts with a structured summary seed (goals, decisions, blockers), runs a short burst of reasoning, and produces a new summary for the next stage.
Think of it as chunked cognition with continuity.

---

## 4️⃣ A Context-Aware Loop

```python
# simplified orchestrator control loop
if remaining_tokens() <= SOFT_THRESHOLD \
   or detects_summary_behavior(model_output) \
   or file_write("SUMMARY.md"):
    checkpoint()
    compact_context()
    persist_state()
    start_new_subcontext()
else:
    continue_work()
```

✅ `SOFT_THRESHOLD ≈ 20 %` of the model’s context window → leaves headroom to offset Sonnet’s tendency to underestimate available tokens.

---

## 5️⃣ Structured Summaries = Machine-Readable Memory

Free-form notes are fragile. Use schemas instead:

```yaml
summary:
  module: "UserAuth"
  goal: "Implement login flow"
  decisions:
    - "Use bcrypt for hashing"
  artifacts:
    - "auth/login.js"
  blockers:
    - "Add tests for refresh tokens"
  next_steps:
    - step: "Write test_login_refresh()"
      acceptance: "All tests pass"
```

Store this externally (database, object store, or vector memory).
Next sub-context loads only these compact fields — never the full transcript.

---

## 6️⃣ Spawning the Next Sub-Context

Seed each new context with:

- **Background:** previous goals, decisions, blockers.
- **Instructions:** next actions + acceptance criteria.
- **Retrieval handles:** file paths or issue IDs, not raw blobs.

```xml
<background>
You’re resuming work on {{module}}.
Past decisions: {{decisions}}
Outstanding blockers: {{blockers}}
</background>

<instructions>
Execute {{next_steps}}.
Fetch details via tools when needed.
Summarize only after acceptance checks pass.
</instructions>
```

---

## 7️⃣ Periodic Heartbeats

Ask the model to emit a light JSON status:

```json
{"heartbeat":{
  "phase":"normal|summarizing|wrapping_up",
  "estimated_remaining_tokens":900,
  "new_files":["SUMMARY.md"]
}}
```

When `phase == "summarizing"`, trigger a checkpoint and start a new sub-context.

---

## 8️⃣ Why This Pattern Works

| Without Management | With Context-Aware Loop |
|--------------------|-------------------------|
| Random early summaries | Predictable checkpoints |
| Context overflow | Controlled resets |
| Lost reasoning chain | Structured memory continuity |
| Token waste | Stable multi-hour sessions |

It converts the model’s “stress response” into a reliable feedback loop.

---

## 🔧 Developer Checklist

- ✅ Monitor token usage per turn
- ✅ Trigger checkpoint ≤ 20 % remaining
- ✅ Detect summary phrases / note-file writes
- ✅ Persist structured YAML/JSON summaries
- ✅ Re-seed minimal sub-contexts
- ✅ Insert start + end reminders to fight premature wrap-ups

---

## ⚠️ Sources & Credit

- Cognition AI, _Rebuilding Devin for Sonnet 4.5: Lessons and Challenges_, Oct 2025
- Anthropic, _Effective Context Engineering for AI Agents_, Sept 2025

This article re-expresses their insights as a reproducible engineering pattern, not as proprietary implementation details.

---

## 🧭 Final Thought

> “Every token is precious.” — Anthropic

When your model starts summarizing, it’s not panicking — it’s signaling.
Listen, checkpoint, and let it breathe.
That’s how context anxiety becomes context awareness.

---

**Tags:** #AIEngineering #LLMAgents #ContextWindow #Anthropic #CognitionAI #Devin #PromptEngineering


---

*If you found this useful, feel free to connect on [LinkedIn](https://www.linkedin.com/in/hieu-tran-data/) or check out my projects on [GitHub](https://github.com/hieutrtr/).*
