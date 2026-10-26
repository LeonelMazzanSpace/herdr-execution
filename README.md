# herdr-execution

Claude Code skill for parallelizing work: from a prompt, the orchestrator builds a short task table and launches Claude Code workers in parallel, each in its own [Herdr](https://herdr.dev) pane. There is no written plan, no research wave by default and no approval stop: it starts after a single assumptions block.

## Self-contained

It does not depend on any other skill. You can install just this one. Everything it needs (Herdr commands, worker states, failure handling, evidence rules) is inside `SKILL.md`.

## Requirements

- Claude Code running inside a Herdr pane (`HERDR_ENV=1`).
- The `herdr` binary on the `PATH`.
- A git repo (build mode uses `git worktree`).
- `pnpm` for the default gates (`lint`, `test`, `build`); if the project uses other commands, the project's own are used.

Optional: the Obsidian MCP, only if the task depends on project knowledge. If it is not connected, it is silently skipped.

## Installation

Copy this folder into the skills directory:

```bash
cp -r herdr-execution ~/.claude/skills/
```

## Usage

The skill sets `disable-model-invocation: true`: Claude does not load it on its own, it must be invoked explicitly.

```
/herdr-execution <what to build or research>
```

## How it works

1. **Setup:** report folder at `~/.cache/herdr-reports/<slug>-<date>`.
2. **Read first:** the orchestrator reads `CLAUDE.md`, dependencies and the files the task touches.
3. **Assumptions block:** an `ID · goal · mode · model · files` table, then it starts. It only asks about product decisions, destructive or sensitive actions, and push/PR/`main`.
4. **Unattended workers:** every worker runs in `--permission-mode auto` with deny rules for `.env` and environment dumps. The orchestrator never asks which permission mode to use, and a worker stuck on a permission prompt is relaunched with that tool allowed instead of being handed to the user; only prompts for things outside its brief reach you.
4. **Contract first:** for parallel builds, the orchestrator writes the shared interfaces before dispatching.
5. **Workers:** at most 4 live panes, each with its own brief (`brief-template.md`) and its own report.
6. **Acceptance:** a `PASS` report file, a diff within scope, verification re-run by the orchestrator, an independent review if the work touches payments/auth/security/migrations, and the full gate after the merge.

## Modes

| Mode | When | Workers |
|---|---|---|
| Build (default) | The task splits into 2+ pieces with disjoint files | One worktree and one branch per worker |
| Research | The user asks for research, or there is a question the repo cannot answer | Read-only, with a tool allowlist |

When **not** to use it: a single question or a small change (do it directly), or pieces that touch the same files (use one worker).

## What it does not do

Running the skill authorizes local commits and merges on `feature/<slug>`. It does not authorize push, PR, a merge into `main`, deleting worktrees or branches, or deploys: each needs its own explicit yes.

## Files

| File | Contents |
|---|---|
| `SKILL.md` | Skill instructions |
| `brief-template.md` | Brief templates for research, build and review |
