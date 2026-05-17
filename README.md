# Ness

General-purpose email-driven agent harness. Provides scheduling, budget control, email transport, conversation history, and self-improvement loops. Domain logic lives in harness files supplied by consumers.

---

## What it is

Ness is infrastructure. It does not know what your agents do — that's defined by harness files (Markdown prompt templates) loaded at runtime.

```
harness files (supplied by consumer)
        +
   Ness (scaffold)
        =
   autonomous agents
```

**Example consumer:** [lazycoder](../lazycoder) — GitHub project manager that runs on Ness, supplies its own harness files.

---

## What Ness provides

- **Email transport** — aiosmtpd receive + aiosmtplib send + Maildir (full history in files)
- **Scheduler** — pure Python, selects which harnesses run each cycle within budget
- **Budget control** — soft/hard daily limits, cost tracking per run, escalation on overrun
- **Run tracking** — consecutive failure detection, stuck harness escalation
- **Self-improvement loop** — Evaluator reads run history, proposes harness diffs, A/B testing, human APPROVE/MODIFY/REJECT

## What Ness does NOT provide

- Any domain logic (that's the harness)
- External API clients (harnesses bring their own via MCP or subprocess)
- Web UI or dashboard

---

## Harness contract

A consumer provides one or more `.md` files in `harnesses/`. Ness loads them, fills context slots, invokes LLM, routes output.

```markdown
# harnesses/my-agent-v1.md

## Role
...

## Context
Today: {{date}}
Budget remaining: {{budget_remaining}}
Recent runs: {{run_history}}

## Task
{{task}}

## Output
Send email to orchestrator@ness.local with subject: [ness] result | {{date}}
```

---

## Cadence

- **Configurable interval** (default 6h): Scheduler selects harnesses → Runner invokes LLM → result to Maildir
- **Daily 18:00:** Digester synthesizes Maildir → digest email, max 5 items
- **Sunday 10:00:** weekly synthesis + one question
- **First Monday:** Evaluator sends harness improvement proposals

---

## Stack

Python + aiosmtpd + Maildir + SQLite + litellm.

---

## Docs

- [ARCHITECTURE.md](ARCHITECTURE.md) — harness system, agents, email protocol, data model, build order
- [SCIENCE.md](SCIENCE.md) — research foundations: harness-driven design, temporal loops, budget enforcement
- [LAZYCODER.md](LAZYCODER.md) — how lazycoder uses Ness
