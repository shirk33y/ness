# SCIENCE.md — Research Foundations for Ness

Sources: `cah-first-task-research-scout.md`, `conversation-t3W5.md`, web research (May 2026).

---

## 1. Harness quality is the primary lever

**Finding:** SWE-bench Pro shows 4-10 percentage point spread from scaffolding alone, same model.
- Opus 4.5: 50.2–55.4% depending on agent system.
- Stanford Meta-Harness reached #1 on TerminalBench-2 with **Haiku 4.5** (37.6%), beating larger models.

**Conclusion:** prompts and architecture matter more than model choice. Ness is built around this — behavior lives in harness files, Python is scaffolding only. Don't reach for Opus first; improve the harness first.

---

## 2. Harness-driven design: code is scaffolding, prompts are behavior

**Finding:** the more domain logic that lives in code, the harder it is to iterate and self-improve. When behavior is in Markdown files (prompts + context slots), you can:
- Swap agent behavior without touching Python
- Version and A/B test prompts like code
- Let the system propose its own improvements (diff to a .md file, not a .py file)
- Archive old versions automatically

**Conclusion:** Python in Ness handles routing, scheduling, budget, and LLM invocation. Everything else — what to look for, how to reason, what to output — lives in `harnesses/*.md`. The system can improve itself because prompt files are first-class artifacts, not buried in code strings.

---

## 3. Self-improvement via prompt A/B testing

**Finding:** LLM agents can evaluate their own outputs if given structured run history and a rubric. This is more reliable than intuition-based prompt tuning.

**Pattern for Ness:**
- Evaluator reads last N run outputs + cost metrics from Maildir + SQLite
- Generates a harness diff (specific line changes, not rewrites)
- Optionally creates a v2 variant for A/B testing
- Orchestrator alternates v1/v2 for M runs, tags each with `X-Ness-Variant`
- Evaluator compares metrics, picks winner, sends PROPOSE email to human
- Human replies APPROVE → harness promoted, old version archived

**Key constraint:** human always approves harness changes. System proposes, human decides.

---

## 4. Three temporal loops pattern

Prevents "mail avalanche" — the most common fail mode of personal agent assistants.

| Loop | Frequency | LLM use | Output |
|---|---|---|---|
| Run | Per schedule (hours) | Sonnet (harness) | Result stored in Maildir |
| Digest | Daily | Haiku (filter + summary) | 1 email, max 5 items |
| Synthesis | Weekly | Sonnet | 1 email, patterns + 1 question |
| Evaluation | Monthly | Sonnet | Harness improvement proposals |

**Key:** LLM only touches filtered items at digest stage. Raw run outputs accumulate in Maildir (free). Daily cost estimate: $0.50–$1.50. Monthly cap: $60.

---

## 5. Email as human-agent interface: correct transport for this use case

From conversation analysis (the "anti-pattern" objection):

- **Anti-pattern argument:** SMTP has 1-30s latency, no ordering guarantees, no transactions. For tight inter-agent loops, use queues or HTTP.
- **Counter-argument (accepted):** For Ness, inter-agent loops are *not tight*. Agents run on schedules (hours apart), not in real-time. Email gives: human-readable audit trail, Maildir = history in files, trivial human-in-the-loop (just reply). SMTP latency is irrelevant at hourly cadence.
- **Conclusion:** Email is appropriate here. The anti-pattern critique applies to real-time agent choreography, not schedule-driven systems.

---

## 6. Scheduler as pure Python: key architectural insight

From lazycoder (prior project):

- Original design: Judicial agent (Haiku) selected tasks — one LLM call per run cycle.
- Refuted: selection logic is deterministic (sort by priority, accumulate estimates, cut at budget). No LLM needed.
- **Pure Python scheduler:** zero LLM calls, fully testable (19 unit tests, no mocking), deterministic output.

**Applied to Ness:** `scheduler.py` selects which harnesses run each cycle — pure Python, no LLM. Most testable component in the system.

---

## 7. History in files: non-negotiable

**Requirement:** all conversation history stored in files, human-readable, no proprietary format.

**Maildir satisfies this out of the box:** one file per message, RFC 822 format, `grep`/`find`/`cat` work without any tooling. No export step needed.

**Conclusion:** Maildir is not just a transport detail — it's the audit trail and history store. SQLite tracks cost/runs only. Maildir is the source of truth for what agents said and did.

---

## 8. MCP ecosystem maturity (May 2026)

- Docker MCP Catalog: 270+ servers.
- `rmcp` (official Rust SDK): 3.4k stars, actively maintained.
- MCP donated to Linux Foundation (Dec 2025): stable infra.
- 97M cumulative downloads (Python + TS SDK).

**Conclusion:** harnesses that need external APIs (GitHub, web search, etc.) should use MCP servers, not custom clients. Don't implement API clients in Python — write a harness that invokes the right MCP tool.

---

## 9. Speed control and token budget: non-negotiable from day one

- Budget without teeth is no budget. Enforce with hard stop, not just logging.
- Start conservative: 1 run/day, $0.10 soft limit, $0.50 hard limit.
- Escalate to human on: cost overrun (>2× estimate), stuck harness (3+ failures), evaluator proposals.
- Human controls speed via `config.yaml` — no code changes needed.

---

## 10. Tech stack decision: Python over Rust for MVP

- Python: faster MVP, official Anthropic SDK, easier harness iteration.
- Rust: better for long-running daemons, single binary, official `rmcp`. Consider for CAH-core long-term.
- **For Ness:** schedule-driven, not performance-critical. Python wins. Rust adds weeks with zero benefit at this scale.

**Chosen stack:** Python + aiosmtpd (receive) + aiosmtplib (send) + Maildir + SQLite + litellm.

---

## 11. What not to build in v1

- Any domain-specific Python code (belongs in harness files)
- Web UI or dashboard
- Vector DB / semantic memory
- Parallel harness execution
- Real-time agent choreography
- Twitter/X monitoring
- Multi-language sources
- Docker per service
