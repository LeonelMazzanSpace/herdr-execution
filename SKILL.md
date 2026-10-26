---
name: herdr-execution
description: Use when the user asks for research or an implementation to be carried out by orchestrating several Claude Code workers in Herdr panes (HERDR_ENV=1) straight from a prompt, with no written plan or spec step.
disable-model-invocation: true
---

# Herdr Execution

## Overview

The orchestrator turns a prompt into a short task table and runs independent Claude Code workers in parallel in Herdr panes. The point is parallelizing the work that was asked for. There is no plan document, no research wave by default and no approval stop: it starts after one assumptions block.

**Core principle:** fast start, same evidence bar. A worker's report is a claim; only what you read, diffed or ran yourself is fact.

**Workers run unattended.** Every worker starts in `--permission-mode auto` with the secret deny rules below. The orchestrator never asks the user which permission mode to use, and a worker that stops on a permission prompt is relaunched with the tool it needed allowed, not handed to the user (see "When a Worker Stops or Breaks"). The user is asked only for the three things in step 3 of Run.

**Self-contained:** this skill needs no other skill. Everything required (CLI, worker states, failure handling, evidence rules) is below. Run `herdr agent --help` or `herdr pane --help` for flags not listed here.

## Herdr essentials

- **Agent states** (`herdr agent get <id>`, `herdr agent wait <id> --until <state> --timeout <ms>`): `working`, `idle`, `done`, `blocked` (waiting on a permission prompt or input), `unknown`. A prompt sent to a `blocked` agent is rejected with `agent_blocked`.
- **Read output:** `herdr agent read <id> --source recent-unwrapped --lines 120` (or `--source visible` for the screen). `herdr agent explain <id>` says why a state was detected.
- **Panes:** `herdr pane layout`, `herdr pane split`, `herdr pane close`, `herdr pane list`.
- **Pane ids** come from JSON output (`.result.pane.pane_id`); use `jq`.

## When to use

| Situation | Do |
|---|---|
| A task that splits into 2+ pieces with disjoint files | Build mode: execute what was asked (default) |
| The user asks for research, or a question you cannot answer by reading the repo yourself | Research mode, only for that question |
| One question, or one small change | Do it yourself; panes cost more than they save |
| Every piece touches the same files | One worker, not several |

`test "$HERDR_ENV" = 1` first. If it fails, ask the user to start the session inside a Herdr pane.

## Run

1. **Setup.** `report_dir=~/.cache/herdr-reports/<slug>-<yyyymmdd>`, `mkdir -p -m 700`. Check `herdr agent`, `herdr pane layout --pane "$HERDR_PANE_ID"` and `claude --help` for the flags you use. Read the scratchpad if one exists (`docs/scratchpad.md`; see Scratchpad).
2. **Read before delegating, yourself.** Project `CLAUDE.md`, dependency keys of `package.json`, folder layout, and the files the task touches. This replaces a research wave: launch research workers only if the user asked for research or a decision truly cannot be made from what you read. A request to implement is executed, not researched first. **Obsidian is optional.** Only if the Obsidian MCP (`mcp__obsidian__*`) is connected and the task depends on project knowledge, query the vault through it; never read vault files from disk. No MCP, no vault, or an MCP error: skip it silently and continue. Do not ask the user to connect it and do not queue vault updates.
3. **Assumptions block, then go.** One message: the task table (`ID · goal · mode · model · files owned`), the assumptions you made, and the decisions you took. Do not wait for approval. Stop and ask only for:
   - product decisions that change scope;
   - destructive or security-sensitive actions (credentials, production data);
   - push, PR, or anything touching `main`.
4. **Contract first.** If workers will build in parallel, you write the shared contract yourself on the coordination branch before dispatching: interfaces, DTOs, injection tokens, file ownership. Parallel workers never invent a shared interface.
5. **Dispatch** (see Workers). Update the scratchpad after each milestone.
6. **Accept** each result (see Accepting), then consolidate and report.

## Workers

- **Briefs.** Write `<report-dir>/<id>.brief.md` from `brief-template.md`. One worker, one goal, one report.
- **Cap:** 4 live worker panes unless the user says otherwise. Reviewers count; a `caffeinate` pane does not.
- **Names** are unique among live agents: `r01…` research, `t01…` build, `t01-r1` reviewer, `t01-f1` fix.
- **Layout:** `herdr pane split --pane "$HERDR_PANE_ID" --direction right --cwd <dir> --no-focus` for the first worker, later `--direction down` in the column with fewer panes. The pane id is `.result.pane.pane_id`. Reuse a pane freed by `/exit` before splitting. Close only panes this run created, and only after the report is accepted.
- **Launch and prompt:**
  ```bash
  herdr agent start <id> --kind claude --pane <pane> --timeout 120000 -- --model <model> <mode flags> --add-dir <report-dir> -n "<ID> <label>"
  herdr agent prompt <id> "Read <report-dir>/<id>.brief.md and execute it exactly. Write your report to <report-dir>/<id>.md and end with STATUS and REPORT lines." --wait --timeout 3600000
  ```
  Run the prompt as a background command. Never resend a prompt; inspect `herdr agent read <id> --source recent-unwrapped --lines 120` first.
- **Verify the effective model** with `herdr agent read <id> --source visible` (status line) and record it. Never assume a model alias is available.
- Do not duplicate an agent that already exists for the task. Never answer another agent's permission prompt: relaunch it with the right flags instead.
- **Secret deny rules, on every launch** (they win over `auto`): `DENY_SECRETS=(--disallowedTools "Read(./.env*)" "Read(**/.env*)" "Bash(printenv:*)" "Bash(env:*)" "Bash(export -p:*)" "Bash(declare -p:*)" "Bash(cat .env*)" "Bash(head .env*)" "Bash(tail .env*)" "Bash(grep * .env*)")`, passed as `"${DENY_SECRETS[@]}"` after the mode flags. Check once per run that `claude --help` lists `--disallowedTools`.

| Work | Model | Mode flags |
|---|---|---|
| File exploration, mapping, simple lookups | `haiku` | research |
| Research synthesis, standard implementation, tests | `sonnet` | research or build |
| Architecture, payments, auth, migrations, hard debugging, reviews of those | `opus` | research or build |

**Research mode** is read-only on the repo. Launch with `--permission-mode dontAsk --allowedTools "<Read, Grep, Glob, WebSearch, WebFetch, read-only git commands, and Write only under the report dir>"`. `dontAsk` denies anything outside the allowlist without prompting, so the worker never blocks. Check the exact rule syntax in `claude --help` the first time. Backstop: after each research worker, `git status --porcelain` must show nothing new. If a worker changed the repo, discard nothing; tell the user.

**Build mode** gives each worker its own worktree and branch off the coordination branch:
```bash
git worktree add <root>.worktrees/<id> -b feature/<slug>-<id> feature/<slug>
(cd <root>.worktrees/<id> && pnpm install --frozen-lockfile)
```
Launch with `--permission-mode auto "${DENY_SECRETS[@]}"`, always, without asking. `auto` runs the brief's commands (`pnpm`, `git` in the worktree, the project's test and build tools) with a safety check that blocks risky ones; the deny rules keep `.env` out of reach even then. If the harness refuses `auto` (the `herdr agent start` call is denied, or the pane shows the refusal in `herdr agent read <id> --source visible` right after start), fall back **on your own** to `--permission-mode acceptEdits --allowedTools "Bash(pnpm:*)" "Bash(npx:*)" "Bash(git:*)" "Bash(node:*)"` plus the commands the brief's Verify section names, record the fallback in the scratchpad, and go on. Never use bypass flags, and never ask the user which mode to use. Workers never touch `.env` secrets, the vault, or the scratchpad.

## Research (only when asked)

- Ask for independent questions in parallel, each with its own report: findings, sources (URL or `file:line`), alternatives, tradeoffs, risks, recommendation.
- Consolidate into `<report-dir>/synthesis.md`: where reports agree, where they contradict, and how you resolved each contradiction (primary source, or one follow-up worker). Do not paste reports together.

## Accepting a build task

All of these, in order:
1. Worker settled (`idle` or `done`) and the **report file** says PASS. Chat text does not count.
2. `git -C <wt> diff --stat $(git -C <wt> merge-base feature/<slug> HEAD)...HEAD` touches only the files the brief allows.
3. You re-ran the brief's Verify commands in the worktree: green.
4. **Review.** Payments, auth, security, migrations, or anything the user flagged: independent reviewer worker (`opus`, read-only, never weaker than the implementer) and no Critical or Important findings. Anything else: you read the diff yourself.
5. `git merge --no-ff` into `feature/<slug>`, then the full gate (`pnpm lint && pnpm test && pnpm build`, or the project's own). Only a green gate makes it `verified`. A red gate stops further merges.

**Red gate:** read the full failure output, find the root cause (which merged task, which file, which contract mismatch) before touching anything, and fix it with one `t01-f1` worker on that cause. Never patch by guessing, never retry the same command hoping it passes, never skip the gate.

## When a Worker Stops or Breaks

| Symptom | Do |
|---|---|
| `blocked` on a permission prompt | Do not answer it. Read it (`herdr agent read <id> --source visible`). If the tool or command is one the brief needs (a test runner, a build step, a git command in its worktree, a file under its allowed list), `/exit` the worker and relaunch it with the same name and brief plus `--allowedTools "<that rule>"`; record the rule in the scratchpad and add it to later launches. Only a prompt for something outside the brief (a destructive command, a secret, a host or path the brief never named) goes to the user, as a finding, with the worker left exited |
| `idle` but no report file | Read `recent-unwrapped` output once; send one short follow-up asking for the report. Never resend the original prompt |
| `working` far past the expected time | `herdr agent read` to see whether it progresses; if it loops or is silent, stop it and relaunch with a narrower brief |
| Edited files outside its allowed list | Do not merge. Report the files to the user; discard nothing without their yes |
| Report says `BLOCKED` | Resolve the cause yourself (contract, missing file) and relaunch; escalate to the user only for scope or product decisions |
| `FAIL` or your Verify run is red | One fix worker `t01-f1` with the failing output in its brief; a second failure goes to the user |
| Agent exited or pane died | Check `herdr agent list`, start a fresh agent in a new pane, reuse the same brief |

The user's request to execute authorizes local commits and merges on `feature/<slug>`, which you create from the current branch. It never authorizes push, PR, a merge into `main`, deleting worktrees or branches, or deploys. Commits are conventional (`feat:`, `fix:`, `test:`).

## Scratchpad

`docs/scratchpad.md`, one writer (you), never committed and never copied into worktrees. Review what is there before editing; append or amend, never overwrite progress. Sections: status, in progress, completed, decisions, blockers, outstanding, next actions. Update after each milestone, with date and time.

If the Obsidian MCP was connected for this run, update the vault at the end, only with verified results and preserving existing decisions. Otherwise there is nothing to sync.

## Red Flags

| Thought | Reality |
|---|---|
| "Start with a research wave to be safe" | Execute what was asked; research only on request or a real gap |
| "Ask the user to approve the task table first" | State assumptions and go; ask only for scope, destructive, or push |
| "The workers can agree on the interface themselves" | You write the contract before parallel build |
| "The research worker is read-only because I said so" | Enforce with the allowlist and check `git status` |
| "PASS in chat, merge it" | Report file, scope diff, your own Verify run |
| "Payments code is small, skip the review" | Payments, auth, migrations always get an `opus` reviewer |
| "Use opus everywhere to be safe" | Match model to the table; cost is a decision |
| "I'll ask which permission mode they want" | `auto` with the secret deny rules, always; `acceptEdits` plus an allowlist is the automatic fallback. The user is never asked about modes |
| "A worker is blocked on a prompt, I'll show it to the user" | Read the prompt. If the brief needs that tool, relaunch with it allowed. The user sees only prompts for things the brief never asked for |
| "Commit the scratchpad with the feature" | It stays untracked |
| "The user said run it, so push too" | Push, PR and `main` need their own explicit yes |
