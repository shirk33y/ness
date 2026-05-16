# Ness

Autonomous GitHub project manager. Manages multiple hobby repos via email — no dashboard, no web UI, just inbox.

Ness monitors your repos, finds issues and opportunities, decomposes them into tasks, executes them within a daily token budget, and sends you concise digests. You reply to redirect it.

---

## How it works

Five agents run on a schedule. Each has its own email address.

| Agent | Does |
|---|---|
| Scanner | Polls GitHub repos, writes findings to SQLite |
| Legislative | Decomposes issues into task checklists, writes `## Plan` comments |
| Judicial | Selects tasks within daily budget |
| Executive | Executes tasks: reads code, writes code, commits, posts `## Status` |
| Digester | Sends daily/weekly email summaries to you |

You receive mail from `digest@ness.local`. You direct the system by replying to `orchestrator@ness.local`. Agents CC you on escalations.

---

## Cadence

- **Every 6h** (configurable): scan → plan → select → execute → summarize
- **Daily 18:00:** digest email, max 5 items or "nothing notable"
- **Sunday 10:00:** weekly synthesis with patterns and one question for you
- **First Monday:** monthly trend report — awaits your APPROVE/MODIFY/REJECT

Speed is fully config-driven. Start conservative (`max_tasks_per_run: 1`, `soft_limit_daily: $0.10`), scale up as trust builds.

---

## Stack

Python + aiosmtpd + Maildir + SQLite + litellm. No external services needed beyond GitHub API and Anthropic API.

Models: Haiku for filtering/triage, Sonnet for planning and synthesis.

---

## Docs

- [ARCHITECTURE.md](ARCHITECTURE.md) — agents, email protocol, data model, file structure, build order
- [SCIENCE.md](SCIENCE.md) — research foundations: what we know about harness quality, agent roles, temporal loops, tech stack decisions
- [conversation-t3W5.md](conversation-t3W5.md) — prior design conversation (CAH architecture, email transport, role separation)
