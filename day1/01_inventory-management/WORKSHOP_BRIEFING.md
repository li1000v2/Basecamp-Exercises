# Workshop Briefing — Inventory Management Session

> Read this first. It's a handoff doc from the user's parallel Claude Code session.

## Context

- **User**: `li1000v2` (xinran.c.li@accenture.com) · GitHub: https://github.com/li1000v2
- **Today**: 2026-05-20
- **Event**: Anthropic Partner Basecamp · Day 1 (in-person workshop)
- **Dual purpose**: The user is also preparing for the **Claude Certified Architect — Foundations (CCA-F)** exam on 2026-05-28. This workshop session is being used as hands-on practice for CCA-F Domain 3 (Claude Code Configuration & Workflows, 20%) and Domain 1 (Agentic Architecture, 27%).
- **The other Claude Code window** (workspace root: `c:\Python\ccaf-prep-agent`) is where the user reflects against CCA-F domains AFTER completing workshop steps here. You do NOT need to mirror that other project's strict conventions (TDD, R0-R6 reviews, STATE.md updates) — those are out of scope here. Follow THIS project's `CLAUDE.md` only.

## Workshop progress so far

| Step | Status | Notes |
|---|---|---|
| 1. Fork + clone + branch | ✅ done | Repo at `C:\Basecamp-Exercises-main\` (this is the `day1/01_inventory-management/` subdir). Remote: `https://github.com/li1000v2/Basecamp-Exercises.git`. Branch: `new_features`. |
| 2. Start Claude Code | (skipped — meta step) | |
| 3. Install deps + start dev servers | ✅ done | **Servers are already running. Do NOT restart them — see below.** |
| 4. CLAUDE.md walkthrough | ✅ done | This briefing is the artifact. |
| 5+ | ⏳ pending | Workshop will direct the user to fix the planted bugs across pages. |

## Environment (already configured — verify, don't reinstall)

- **Python**: 3.14.3
- **Node**: 24.14.0, **npm**: 11.9.0
- **Git identity** (global): `li1000v2 <li1000v2@gmail.com>`
- **Backend venv**: `server/.venv/` with `fastapi 0.136.1`, `uvicorn 0.47.0`, `pydantic 2.13.4`, plus `pytest`, `pytest-asyncio`, `httpx`, `pytest-cov`.
- **Frontend deps**: `client/node_modules/` already installed (Vue 3, Vite 5.4, axios, vue-router).
- **`uv` is NOT installed.** The official `CLAUDE.md` says `uv run python main.py` — substitute `.\.venv\Scripts\python.exe main.py` from `server/` instead.
- **`.env` for `ANTHROPIC_API_KEY`** lives at `C:\Basecamp-Exercises-main\.env` (workspace-root level, the directory ABOVE this one). The inventory app itself does NOT call the Claude API (it's pure FastAPI+Vue), so this is only relevant for unrelated notebook sessions in sibling folders. Your Claude Code extension authenticates separately.

## Running dev servers — important

Both dev servers are running, but they were started by a **different Claude Code session** (the user's parallel window). They will NOT show up in your background-task list. Endpoints:

- Frontend: http://localhost:3000  ← the dashboard
- Backend: http://localhost:8001
- API docs: http://localhost:8001/docs

**If you need to restart them**: do NOT just `npm run dev` — port 3000 will conflict. Either:
1. Ask the user to kill them from the other window (preferred), OR
2. Kill by port from PowerShell:
   ```powershell
   Get-NetTCPConnection -LocalPort 8001 -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
   Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
   ```

## Local workspace files committed at repo root

The repo root (`C:\Basecamp-Exercises-main\`, parent of this folder) contains four locally-added files NOT visible in this VS Code workspace explorer but tracked in git on `new_features`:
- `SETUP.md` — Chinese-language quickstart for all 8 Basecamp sessions
- `.gitignore` — top-level rules covering venvs / node_modules / .env / IDE
- `.env.example` — `ANTHROPIC_API_KEY` template
- `Basecamp-Exercises.code-workspace` — VS Code multi-root workspace (not used for this Claude session)

You can ignore them — they're for the other window's navigation.

## Conventions you MUST follow (from this project's CLAUDE.md)

- **ANY `.vue` file edit → mandatory delegation to `vue-expert` subagent** (Task tool).
- Significant code changes → run `code-reviewer` subagent afterwards.
- GitHub operations → use `mcp__github__*` MCP tools. Exception: local-only `git checkout -b` is fine via Bash.
- Browser testing → use `mcp__playwright__*` MCP tools against `http://localhost:3000` and `http://localhost:8001`.
- Backend tests in `tests/backend/` → invoke the `backend-api-test` skill.
- Common bugs to watch for (the app has planted bugs):
  - `v-for` using `index` as key instead of `sku` / `month`
  - Missing date validation before `.getMonth()`
  - Pydantic models drifting from JSON data structure
  - Inventory filters incorrectly supporting `month` (no time dimension exists)
  - Revenue goals: $800K/month single, $9.6M YTD

## User communication preferences

- High-level discussions / explanations: **Chinese**
- Code, commit messages, file contents: **English**
- The user prefers being asked clarifying questions over having you guess. When ambiguous, ask.
- The user is new-ish to GitHub; explain git operations clearly when you do them.

## What "submission" means here

This is a personal training run — no PR back to upstream `victorsteeb/Basecamp-Exercises` or `beck-source/inventory-management`. "Submission" = commits on `new_features` pushed to the user's fork. Push when natural; don't auto-push without telling the user.

## Smoke test you can run now to verify everything

```powershell
Invoke-WebRequest http://localhost:8001/api/dashboard/summary -UseBasicParsing | Select-Object StatusCode
Invoke-WebRequest http://localhost:3000 -UseBasicParsing | Select-Object StatusCode
git status
git log --oneline -3
```

Expected: both 200, branch is `new_features`, last commit is `chore: add local workspace setup files` (or this briefing if it's been committed).

---

You're now caught up. Wait for the user's next workshop step.
