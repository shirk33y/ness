# LAZYCODER.md — How Lazycoder Uses Ness

Lazycoder is a GitHub project manager built on top of Ness. Ness provides the infrastructure (email, scheduling, budget, history). Lazycoder provides the domain logic as harness files.

---

## Relationship

```
lazycoder/
└── harnesses/
    ├── planner-v1.md      → loaded by Ness Runner
    ├── executor-v1.md     → loaded by Ness Runner
    └── summarizer-v1.md   → loaded by Ness Runner

ness/                      → runs the above, knows nothing about GitHub
```

Lazycoder installs Ness as a dependency, drops its harness files into `harnesses/`, and configures `config.yaml` to activate them. No Python changes to Ness needed.

---

## What Lazycoder contributes

| File | Lives in | What it is |
|---|---|---|
| `harnesses/planner-v1.md` | lazycoder repo | Prompt: read GitHub issues, decompose into checklist with cost estimates |
| `harnesses/executor-v1.md` | lazycoder repo | Prompt: take task from checklist, invoke mini-swe-agent, post ## Status |
| `harnesses/summarizer-v1.md` | lazycoder repo | Prompt: collect ## Status comments, post ## Summary |
| `github_client.py` | lazycoder repo | httpx wrapper for GitHub API — called from harnesses via MCP or subprocess |

## What Lazycoder reuses from Ness as-is

- `budget.py` — `spent_today.json`, soft/hard limits, `add_entry`, `over_hard_limit`
- `scheduler.py` — pure Python harness selection within budget
- `run_tracker.py` — `STUCK_THRESHOLD=3`, `transient=True` for rate limits
- `mail.py` — full email transport + Maildir history
- `orchestrator.py` — cron cycle, inbound reply routing
- `evaluator.py` — self-improvement proposals for lazycoder's own harness files
- `digester.py` — daily/weekly synthesis

---

## Migration path from current lazycoder

Current lazycoder has domain logic in Python (`planner.py`, `executor.py`, `summarizer.py`). Migration:

1. Extract prompts from those files → move to `harnesses/*.md`
2. Python files become thin MCP/subprocess wrappers (or deleted if harness handles it directly)
3. Wire Ness as the runner
4. Budget, scheduler, run_tracker copy as-is (same logic, tested)

The 6 open PRs (`fix/1`–`fix/5`, `fix/executor-agent-config`) should be reviewed before migration — may contain fixes that inform harness design.

---

## Config (lazycoder's config.yaml)

```yaml
harnesses:
  active:
    - planner-v1
    - executor-v1
    - summarizer-v1

github:
  token_env: GITHUB_TOKEN
  bot_username: lazycoder[bot]
  repos:
    - owner/repo-a
    - owner/repo-b
```

Ness reads `harnesses.active`, everything else is lazycoder-specific and passed as context slots.
