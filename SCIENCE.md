# SCIENCE.md — Research Foundations for Ness

Sources: `cah-first-task-research-scout.md`, `conversation-t3W5.md`, web research (May 2026).

---

## 1. Harness quality is the primary lever

**Finding:** SWE-bench Pro shows 4-10 percentage point spread from scaffolding alone, same model.
- Opus 4.5: 50.2–55.4% depending on agent system.
- Stanford Meta-Harness reached #1 on TerminalBench-2 with **Haiku 4.5** (37.6%), beating larger models.

**Conclusion for Ness:** architecture and prompting matter more than model choice. Start with Haiku for filtering/triage, Sonnet only for synthesis. Don't reach for Opus first.

---

## 2. Three temporal loops pattern

From research-scout design + conversation analysis. Prevents "mail avalanche" — the most common fail mode of personal research/PM assistants.

| Loop | Frequency | LLM use | Output |
|---|---|---|---|
| Scan | Hourly / per schedule | Zero | Raw findings in SQLite |
| Digest | Daily | Haiku (binary filter + short summary) | 1 email, max 5 items |
| Synthesis | Weekly | Sonnet | 1 email, patterns + 1 question |
| Trend report | Monthly | Sonnet | Scope adjustment proposal |

**Key:** hourly scan costs nothing. LLM only touches filtered items. Daily cost estimate: $0.50–$1.50 at Haiku rates. Monthly cap: $60.

---

## 3. Email as human-agent interface: good. As inter-agent transport: neutral.

From conversation analysis (the "anti-pattern" objection):

- **Anti-pattern argument:** SMTP has 1-30s latency, no ordering guarantees, no transactions. For tight inter-agent loops, use queues or HTTP.
- **Counter-argument (accepted):** For Ness, inter-agent loops are *not tight*. Agents run on schedules (hours apart), not in real-time. Email gives: human-readable audit trail, archivability, trivial human-in-the-loop (just reply). SMTP latency is irrelevant at hourly cadence.
- **Conclusion:** Email is appropriate here. The "anti-pattern" critique applies to real-time agent choreography, not schedule-driven PM systems.

---

## 4. Role-based agent separation (evolved from gh-tribunal → lazycoder)

Original gh-tribunal model: Legislative / Judicial / Executive. **This was refuted and simplified** in practice (lazycoder project):

- `legislative.py` → **deleted**, replaced by `planner.py` (Sonnet, smaller scope)
- `judicial.py` → **deleted**, replaced by `scheduler.py` (**pure Python, zero LLM** — sorts by priority labels, accumulates estimates, cuts at budget)
- `executive.py` → **deleted**, replaced by `executor.py` (mini-swe-agent wrapper)

**Evolved model (used in Ness):**
- **Planner (Sonnet):** reads issues, decomposes into task checklists with cost estimates, posts `## Plan` and `## Summary` comments.
- **Scheduler (pure Python):** reads plans + `spent_today.json`, checks `run_counts.json` for stuck tasks, selects tasks within budget. Zero LLM calls — most testable component.
- **Executor (Sonnet via mini-swe-agent):** executes selected tasks, commits to `bot/run-{date}` branch, opens PRs, posts `## Status`.

**Key insight:** Scheduler as pure Python eliminates an entire LLM call per run and makes prioritization logic deterministic and fully testable (19 unit tests with no mocking needed in lazycoder).

---

## 5. MCP ecosystem maturity (May 2026)

- Docker MCP Catalog: 270+ servers.
- `rmcp` (official Rust SDK): 3.4k stars, actively maintained, has macro-based tool definitions.
- MCP donated to Linux Foundation (Dec 2025): inflection point for ecosystem permanence.
- 97M cumulative downloads (Python + TS SDK).

**Conclusion for Ness:** MCP is stable infra. PM agents can use MCP servers for GitHub API access, web search, etc. Don't implement GitHub client from scratch — use or wrap an MCP server.

---

## 6. Agent observability tools exist now

- **Manifest:** token cost tracking per agent.
- **traceAI:** OpenTelemetry-based agent tracing.
- **Letta:** memory framework for persistent agent state.

**Conclusion:** Ness should track costs per agent per run from day one (SQLite `runs` table). Don't retrofit observability later.

---

## 7. Tech stack decision: Python over Rust for MVP

From conversation (user explicitly chose Python + aiosmtpd):

- Python: faster MVP, Anthropic SDK official + up-to-date, easier harness iteration.
- Rust: better for long-running daemons, single binary, official `rmcp`. Recommended for CAH-core long-term.
- **For Ness specifically:** schedule-driven, not performance-critical. Python wins. Rust adds weeks with zero end-user benefit for a tool running hourly scans.

**Chosen stack:** Python + aiosmtpd (receive) + aiosmtplib (send) + Maildir + SQLite.

---

## 8. Speed control and token budget: non-negotiable from day one

From research-scout success criteria and conversation:

- Budget without teeth is no budget. Enforce with hard stop, not just logging.
- Start conservative: 1-2 tasks/day, $0.10 soft limit, $0.50 hard limit.
- Escalate to human on: overrun, stuck task 3+ runs, new milestone release.
- Human (Mat) controls speed via config YAML — no code changes needed to slow down or speed up.

---

## 9. GitHub comment conventions as state (from gh-tribunal)

All state lives in GitHub comments using markdown headers as identifiers:
- `## Plan` — legislative output, subtask checklist with cost estimates
- `## Status` — executive output per run, what happened, cost, what's remaining
- `## Summary` — legislative post-run, cross-issue overview
- `## Paused` — escalation marker with @mention

**Finding:** no external DB needed for GitHub state. GitHub itself is the state store. Local SQLite tracks cost/budget/scheduling only.

---

## 10. What not to build in v1

Explicitly out of scope based on research:

- Twitter/X monitoring (API unstable, signal/noise terrible)
- Active code replication (testing harnesses from papers)
- Multi-language sources
- Vector DB / semantic memory (overkill)
- Web UI or dashboard
- Parallel execution across repos (sequential avoids rate limits)
- Real-time agent choreography
