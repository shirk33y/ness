# ARCHITECTURE.md — Ness

Email-driven autonomous agent harness. Domain-agnostic — behavior defined by harness files, not hardcoded Python.

---

## Overview

Ness is a thin Python scaffold that:
1. Receives email → routes to correct harness
2. Loads harness (prompt + context) → invokes LLM
3. Sends result via email → stores in Maildir
4. Tracks cost + run history in SQLite
5. Periodically evaluates its own performance → proposes prompt improvements

What the agents *do* is defined in `harnesses/` as Markdown files. Python knows nothing about domains.

```
inbound email
     │
     ▼
 [Orchestrator]  ── route by subject/headers ──▶ [Runner]
                                                      │
                                              load harness file
                                              inject context
                                              invoke LLM
                                                      │
                                              ┌───────┴────────┐
                                              ▼                ▼
                                         Maildir          SQLite
                                     (full history)   (cost + runs)
                                              │
                                              ▼
                                      outbound email
                                      → you / next agent
```

At scheduled intervals: [Evaluator] reads run history → proposes harness improvements → you APPROVE/MODIFY/REJECT.

---

## Core concept: harness files

A harness is a Markdown file that defines an agent's behavior. Python reads it, fills in context slots, sends to LLM.

```markdown
# harnesses/research-scout-v1.md

## Role
You are a research scout monitoring AI/ML papers and tools.

## Context
Today: {{date}}
Recent runs: {{run_history}}
Budget remaining: {{budget_remaining}}

## Task
{{task}}

## Output format
Send email to digest@ness.local with subject: [ness] finding | {{date}}
Body: 3 bullet points max. One sentence each. Cost estimate for follow-up.

## Constraints
- Max 3 findings per run
- Skip anything older than 7 days
- If unsure relevance: skip, don't include
```

Swapping harness = different agent behavior, zero Python changes.

---

## Self-improvement loop

After N runs, Evaluator reads Maildir history + SQLite metrics and proposes changes:

```
[Evaluator] reads:
  - last 30 run outputs (Maildir)
  - cost per run, result quality ratings (SQLite)
  - current harness file

proposes:
  - harness diff (specific prompt changes)
  - A/B variant (runs both, tracks which performs better)
  - deprecation (remove harness that consistently underperforms)

→ email to you: APPROVE / MODIFY / REJECT
→ on APPROVE: harness file updated, old version moved to harnesses/_archive/
```

A/B testing: Orchestrator alternates between `harness-v1.md` and `harness-v2.md` for N runs, Evaluator compares metrics, picks winner.

---

## Agents

Each agent is an email address + a harness file. Orchestrator maps addresses to harnesses.

| Agent | Email | Model | Harness |
|---|---|---|---|
| Orchestrator | `orchestrator@ness.local` | — | `harnesses/_orchestrator.md` (routing logic) |
| Runner | `runner@ness.local` | Sonnet | whichever harness was routed to it |
| Digester | `digest@ness.local` | Haiku | `harnesses/_digester.md` |
| Evaluator | `evaluator@ness.local` | Sonnet | `harnesses/_evaluator.md` |
| Scheduler | `scheduler@ness.local` | — | Pure Python, no LLM — budget + run selection |

**You receive mail from:** `digest@ness.local`, `evaluator@ness.local` (proposals), `orchestrator@ness.local` (escalations).

**You send mail to:** `orchestrator@ness.local` — directives, APPROVE/MODIFY/REJECT replies.

---

## Run cycle

Triggered by cron (configurable interval).

```
1. Orchestrator  → check inbound mail → process any human replies first
2. Scheduler     → read spent_today.json + run_counts.json → select which harnesses run (pure Python)
3. Runner        → load harness → inject context → invoke LLM → send output email + store in Maildir
4. Orchestrator  → log costs to SQLite + spent_today.json
5. Scheduler     → flag stuck harnesses (3+ consecutive failures) → escalate to human
```

At 18:00 daily: Digester synthesizes Maildir → sends daily digest (max 5 items).
On Sunday 10:00: Digester sends weekly synthesis + one question for you.
On first Monday: Evaluator sends harness improvement proposals.

---

## Email protocol

### Subject format

```
[ness] <harness> | <action> | <date>
[ness] digest | daily | 2026-05-17
[ness] ESCALATE | <reason>
[ness] PROPOSE | <harness-name> | v2
```

### Custom headers

```
X-Ness-Agent: runner
X-Ness-Harness: research-scout-v1
X-Ness-Run: 2026-05-17T18:00:00Z
X-Ness-Cost: 0.04
X-Ness-Variant: a          # for A/B runs
```

---

## Speed control

All in `config.yaml`. No code changes to throttle.

```yaml
schedule:
  run_interval_hours: 6
  digest_time: "18:00"
  weekly_synthesis_day: Sunday
  evaluator_day: 1           # first of month

budget:
  soft_limit_daily: 0.50
  hard_limit_daily: 1.00
  cost_overrun_multiplier: 2.0

pace:
  max_runs_per_cycle: 3      # how many harnesses run per cron tick
  stuck_threshold: 3         # consecutive failures before escalation

harnesses:
  active:
    - research-scout-v1
    - github-pm-v1           # optional, if you want GitHub PM as one harness
  ab_tests:
    - [research-scout-v1, research-scout-v2]
```

Start settings for day 1 (conservative):
- `run_interval_hours: 24`
- `max_runs_per_cycle: 1`
- `soft_limit_daily: 0.10`

---

## Data model

### SQLite tables

```sql
-- one row per LLM invocation
CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    run_at TEXT,
    harness TEXT,              -- harness filename (without .md)
    variant TEXT,              -- 'a', 'b', or null
    estimated REAL,
    actual REAL,
    result TEXT,               -- completed, partial, failed, skipped
    quality INTEGER            -- 1-5, set by evaluator or human reply
);

-- budget tracking (resets daily)
CREATE TABLE budget (
    date TEXT PRIMARY KEY,
    spent REAL DEFAULT 0
);

-- harness metadata
CREATE TABLE harnesses (
    name TEXT PRIMARY KEY,
    version INTEGER DEFAULT 1,
    enabled INTEGER DEFAULT 1,
    promoted_at TEXT,          -- when last version was approved
    ab_active INTEGER DEFAULT 0
);
```

### spent_today.json

```json
{
  "date": "2026-05-17",
  "runs": [
    {"harness": "research-scout-v1", "estimated": 0.05, "actual": 0.04}
  ],
  "total": 0.04
}
```

---

## File structure

```
ness/
├── README.md
├── ARCHITECTURE.md          ← this file
├── SCIENCE.md               ← research foundations
├── LAZYCODER.md             ← comparison with lazycoder
├── pyproject.toml
├── uv.lock
├── justfile
├── config.yaml
├── config.example.yaml
│
├── ness/
│   ├── __init__.py
│   ├── __main__.py          # python -m ness run | cli
│   ├── config.py            # pydantic-settings
│   ├── mail.py              # aiosmtpd handler, aiosmtplib sender, Maildir
│   ├── protocol.py          # pydantic: Envelope, Proposal, Escalation
│   ├── state.py             # SQLite schema + read/write
│   ├── budget.py            # spent_today.json, limit checks, cost tracking
│   ├── run_tracker.py       # run_counts.json, consecutive failure detection
│   ├── harness.py           # load .md, fill context slots, invoke LLM
│   ├── scheduler.py         # select which harnesses run (pure Python, no LLM)
│   ├── orchestrator.py      # cron cycle, inbound mail routing, reply handling
│   └── evaluator.py         # run history analysis, propose diffs, A/B tracking
│
├── harnesses/               # agent behavior lives here, not in Python
│   ├── _orchestrator.md     # routing + escalation logic
│   ├── _digester.md         # daily/weekly synthesis
│   ├── _evaluator.md        # self-improvement proposals
│   ├── research-scout-v1.md # example domain harness
│   └── _archive/            # superseded harness versions
│
├── data/                    # gitignored, runtime state
│   ├── ness.db
│   ├── spent_today.json
│   ├── run_counts.json
│   └── maildir/             # full conversation history in files
│       ├── new/
│       ├── cur/
│       └── tmp/
│
└── tests/
    ├── conftest.py
    ├── test_budget.py
    ├── test_scheduler.py
    └── test_harness.py
```

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | Python 3.12+ | Faster MVP, official Anthropic SDK, easier prompt iteration |
| SMTP receive | aiosmtpd | Lightweight local SMTP listener |
| SMTP send | aiosmtplib | Async SMTP client |
| Mail storage | Maildir | Files, human-readable, full history, no daemon |
| State | SQLite (via aiosqlite) | Zero-infra, sufficient for any harness count |
| LLM | litellm | Model-agnostic, `completion_cost()`, swap models without code changes |
| Config | pydantic-settings | Validates config.yaml, env var overrides |
| Task runner | just | Simple |
| Packaging | uv | Fast lockfile |

Removed: PyGithub, httpx — no domain-specific clients in core. Harnesses that need external APIs bring their own tools via MCP or subprocess.

---

## Dependencies

```toml
[project]
dependencies = [
    "aiosmtpd",
    "aiosmtplib",
    "aiosqlite",
    "litellm",
    "pydantic-settings",
    "pyyaml",
    "click",
]
```

---

## What to build first (order matters)

1. `state.py` + `budget.py` + `run_tracker.py` — schema, cost tracking, stuck detection
2. `harness.py` — load .md, fill `{{slots}}`, invoke LLM, return result
3. `scheduler.py` — pure Python: which harnesses run this cycle, within budget
4. `mail.py` — receive + send + Maildir write
5. `orchestrator.py` — wire cron cycle + reply routing
6. `evaluator.py` — read history, propose diffs (stub first)
7. `digester.py` — daily email synthesis
8. Tests for budget + scheduler (pure functions, no mocking)

---

## Escalation triggers

Send to human immediately:
- Actual cost > 2× estimate for any single run
- Hard daily budget exceeded
- Harness stuck 3+ consecutive runs
- Evaluator proposes harness change (always requires human APPROVE)

---

## Non-goals (v1)

- Web UI, dashboard
- Parallel harness execution
- Vector DB / semantic memory
- Multi-machine deployment
- Real-time agent choreography
- Any hardcoded domain logic in Python (that's what harnesses are for)
