# ARCHITECTURE.md — Ness

Email-driven autonomous GitHub project manager.

---

## Overview

Ness runs autonomously on a schedule. It monitors your GitHub repos, finds issues and opportunities, manages tasks, and communicates with you via email. You reply to redirect it. No dashboard, no web UI — just email.

```
GitHub repos
     │
     ▼
 [Scanner]   ─────────────────────────────────────────────────┐
     │ raw findings → SQLite                                   │
     ▼                                                         │
 [Planner]  ── plans, decomposed tasks ──▶ GitHub comments    │
     │                                                         │
     ▼                                                         │
 [Scheduler] ── selected tasks (pure Python, no LLM) ─────────┤
     │                                                         │
     ▼                                                         │
 [Executor]  ── code, commits, PRs ──▶ GitHub + status comment│
     │                                                         │
     ▼                                                         │
 [Digester]  ── daily/weekly summaries ──▶ your inbox ◀───────┘
     │
     ▼
  you (reply to redirect, CC'd on escalations)
```

---

## Agents

Each agent has its own email address. This makes routing, filtering, and audit trivial.

| Agent | Email | Model | Role |
|---|---|---|---|
| Orchestrator | `orchestrator@ness.local` | — | Schedules, routes inbound mail, manages run cycle |
| Scanner | `scanner@ness.local` | — | Fetches GitHub state, writes findings to SQLite. No LLM. |
| Planner | `planner@ness.local` | Sonnet | Decomposes issues into task checklists, posts `## Plan` comments |
| Scheduler | `scheduler@ness.local` | — | Reads plans + budget, selects tasks. Pure Python, no LLM. |
| Executor | `executor@ness.local` | Sonnet | Executes tasks via mini-swe-agent, commits, posts `## Status` |
| Digester | `digest@ness.local` | Haiku/Sonnet | Produces daily/weekly email summaries for human |

**You receive mail from:** `digest@ness.local`, `orchestrator@ness.local` (escalations).

**You send mail to:** `orchestrator@ness.local` (directives, APPROVE/MODIFY/REJECT replies).

---

## Run cycle

Triggered by cron (default: every 6h, configurable).

```
1. Scanner    → poll GitHub repos → write findings to SQLite
2. Planner    → read new issues without Plan → post ## Plan comments
3. Planner    → update existing Plans from ## Status comments
4. Scheduler  → read Plans + spent_today.json → select tasks (pure Python)
5. Executor   → work selected tasks sequentially → commit + post ## Status
6. Executor   → check hard budget between tasks; stop if exceeded
7. Planner    → collect ## Status → post ## Summary per repo
8. Scheduler  → flag stuck tasks (3+ consecutive failures) as needs-human
9. Orchestrator → log costs to spent_today.json
```

At 18:00 daily: Digester reads SQLite + GitHub comments, sends daily digest.
On Sunday 10:00: Digester sends weekly synthesis.
On first Monday: Digester sends monthly trend report (waits for APPROVE/MODIFY/REJECT reply).

---

## Email protocol

### Subject format

```
[ness] <agent> | <repo> | <issue#> | <action>
[ness] digest | daily | 2026-05-16
[ness] ESCALATE | <reason>
```

### Custom headers

```
X-Ness-Agent: legislative
X-Ness-Repo: owner/repo-name
X-Ness-Issue: 42
X-Ness-Run: 2026-05-16T18:00:00Z
X-Ness-Cost: 0.04
```

### GitHub comment conventions

```markdown
## Plan
- [ ] Add OAuth middleware (~$0.05)
- [ ] Write integration tests (~$0.03)

## Status
Task: Add OAuth middleware
Branch: ness/run-2026-05-16
Result: Tests passing. Linting fails on line 47.
Cost: $0.04 (estimate: $0.05)

## Summary
Run 2026-05-16: 2 tasks, 1 done, 1 partial.
Total: $0.08 / $0.50 budget.

## Paused
Estimate: $0.05, spent: $0.11 (2.2x over).
@shirk33y review needed.
```

---

## Speed control

All in `config.yaml`. No code changes to throttle.

```yaml
schedule:
  run_interval_hours: 6        # how often the cycle runs
  digest_time: "18:00"
  weekly_synthesis_day: Sunday
  monthly_report_day: 1        # first of month

budget:
  soft_limit_daily: 0.50       # judicial uses this to select tasks
  hard_limit_daily: 1.00       # executive checks before every LLM call
  cost_overrun_multiplier: 2.0 # pause task if actual > estimate × this

pace:
  max_tasks_per_run: 3         # executive hard cap per cycle
  max_repos_per_run: 5         # scanner hard cap
```

Start settings for day 1 (conservative):
- `run_interval_hours: 24` (daily only)
- `max_tasks_per_run: 1`
- `soft_limit_daily: 0.10`

---

## Data model

### SQLite tables

```sql
-- what scanner found
CREATE TABLE findings (
    id TEXT PRIMARY KEY,           -- ness-2026-05-16-0042
    repo TEXT NOT NULL,
    issue_number INTEGER,
    type TEXT,                     -- new_issue, pr_opened, pr_merged, etc.
    url TEXT,
    discovered_at TEXT,
    processed INTEGER DEFAULT 0
);

-- cost tracking (resets daily)
CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    run_at TEXT,
    agent TEXT,
    repo TEXT,
    issue_number INTEGER,
    estimated REAL,
    actual REAL,
    result TEXT                    -- completed, partial, paused, skipped
);

-- per-repo config overrides
CREATE TABLE repos (
    full_name TEXT PRIMARY KEY,    -- owner/repo
    enabled INTEGER DEFAULT 1,
    priority INTEGER DEFAULT 2,    -- 1=critical, 2=high, 3=medium, 4=low
    budget_share REAL DEFAULT 1.0  -- relative budget weight
);
```

### spent_today.json

```json
{
  "date": "2026-05-16",
  "tasks": [
    {"repo": "owner/repo-a", "issue": 12, "estimated": 0.05, "actual": 0.04}
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
├── pyproject.toml
├── uv.lock
├── justfile
├── config.yaml
├── config.example.yaml
│
├── ness/
│   ├── __init__.py
│   ├── __main__.py          # python -m ness orchestrator | worker | cli
│   ├── config.py            # pydantic-settings
│   ├── mail.py              # aiosmtpd handler, aiosmtplib sender, Maildir
│   ├── protocol.py          # pydantic: RunEnvelope, DigestEnvelope, Escalate
│   ├── state.py             # SQLite schema + read/write
│   ├── github_client.py     # httpx wrapper: issues, comments, labels, branches
│   ├── budget.py            # spent_today.json, limit checks, cost tracking
│   ├── run_tracker.py       # run_counts.json, consecutive failure tracking
│   ├── scanner.py           # fetch repos → write findings (no LLM)
│   ├── planner.py           # plan decomposition, summary generation (Sonnet)
│   ├── scheduler.py         # task selection within budget (pure Python, no LLM)
│   ├── executor.py          # task execution via mini-swe-agent
│   ├── digester.py          # daily/weekly/monthly email synthesis
│   └── orchestrator.py      # cron cycle, inbound mail routing
│
├── harnesses/               # NL harness definitions (data, not code)
│   ├── pm-v1.md             # project manager harness
│   └── _archive/
│
├── data/                    # gitignored, runtime state
│   ├── ness.db
│   ├── spent_today.json
│   └── maildir/
│
└── tests/
    ├── conftest.py
    ├── test_budget.py
    ├── test_scheduler.py
    └── test_protocol.py
```

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | Python 3.12+ | Faster MVP, official Anthropic SDK, easier prompt iteration |
| SMTP receive | aiosmtpd | Lightweight local SMTP listener, no full mail server needed in v1 |
| SMTP send | aiosmtplib | Async SMTP client |
| Mail storage | Maildir | Simple, auditable, no daemon needed |
| State | SQLite (via aiosqlite) | Zero-infra, sufficient for ~10 repos |
| LLM | litellm | Model-agnostic, has `completion_cost()`, works with Anthropic SDK |
| GitHub API | httpx + PyGithub | Direct REST, no magic |
| Config | pydantic-settings | Validates config.yaml, env var overrides |
| Task runner | just | Simpler than make for this use case |
| Packaging | uv | Fast, good lockfile, replaces pip-tools |

---

## Dependencies

```toml
[project]
dependencies = [
    "aiosmtpd",
    "aiosmtplib",
    "aiosqlite",
    "litellm",
    "PyGithub",
    "httpx",
    "pydantic-settings",
    "pyyaml",
    "click",
]
```

---

## What to build first (order matters)

1. `state.py` + `budget.py` + `run_tracker.py` — schema, cost tracking, stuck detection
2. `github_client.py` — fetch issues, read bot comments by header, post comments
3. `scanner.py` — repo scan → findings (no LLM, easiest to test)
4. `scheduler.py` — pure Python task selection, no LLM, most testable logic
5. `planner.py` — plan decomposition (Sonnet)
6. `executor.py` — task execution via mini-swe-agent (stub first)
7. `digester.py` — daily email
8. `mail.py` + `orchestrator.py` — wire everything together
9. Tests for budget and scheduler (no mocking needed, pure functions)

---

## Escalation triggers (bypass schedule)

Send to human immediately:
- Task actual cost > 2× estimate → `## Paused` comment + @mention + `needs-human` label
- Hard budget exceeded for the day
- Task stuck 3+ consecutive runs → `needs-human` label
- GitHub API rate limit sustained > 30 min

---

## Non-goals (v1)

- Web UI, dashboard
- Parallel task execution across repos
- Twitter/X or Discord monitoring
- Vector DB / semantic memory
- Multi-machine deployment
- Docker per service (single compose.yml maximum)
- Real-time agent choreography
