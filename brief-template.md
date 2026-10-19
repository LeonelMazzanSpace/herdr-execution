# Worker brief templates

Fill one per worker and save it as `<report-dir>/<id>.brief.md`. Every field is required; write `none` rather than deleting a line. Workers read the project and global `CLAUDE.md` themselves, so do not restate company rules.

## Research brief

````markdown
# <ID> — <question as a title>

**Mode:** research, read-only. Do not edit, create or delete anything outside `<report-dir>`.
**Question:** <one question, answerable on its own>
**Context:** <what the orchestrator already knows; paths and vault excerpts quoted, not summarized>
**Scope:** <where to look: directories, docs, URLs, or "web">
**Out of scope:** <what to leave to other workers>

## Report (`<report-dir>/<ID>.md`)
1. Answer in three lines
2. Findings, each with a source (URL or `file:line`)
3. Alternatives and tradeoffs
4. Risks
5. Recommendation
6. Open questions

End the report with `STATUS: DONE | BLOCKED (<reason>)` and `REPORT: <path>`.
````

## Build brief

````markdown
# <ID> — <goal as a title>

**Mode:** build. Worktree `<path>`, branch `feature/<slug>-<id>`. Work only there.
**Goal:** <the observable result>
**Contract (fixed, do not change):** <interfaces, DTOs, tokens, endpoints, quoted verbatim from the coordination branch>
**Files you own:** <list or globs. Anything else is out of scope; if you need it, stop with BLOCKED>
**Do not touch:** <shared registration files owned by the orchestrator, `.env`, lockfile, other tasks' files>
**Libraries:** <only these; a new dependency means BLOCKED>
**Environment:** <test DB variable, port, shared services>

## Verify (run all, paste the real output in the report)
```bash
<exact commands, e.g. pnpm lint && pnpm test <path> && pnpm build>
```

## Report (`<report-dir>/<ID>.md`)
What changed (files), what you verified (commands and results), deviations, anything you could not do.

Commit on your branch with conventional messages. End with `STATUS: PASS | FAIL | BLOCKED (<reason>)` and `REPORT: <path>`.
````

## Review brief

````markdown
# <ID>-r<N> — review of <ID>

**Mode:** review, read-only. Do not edit the worktree.
**Diff:** `git -C <wt> diff $(git -C <wt> merge-base feature/<slug> HEAD)...HEAD`
**Brief and report under review:** `<report-dir>/<ID>.brief.md`, `<report-dir>/<ID>.md`
**Review focus:** <correctness against the contract, input validation, error leakage, idempotency, secrets, transactions, migrations; pick what applies>

Write `<report-dir>/<ID>.review.md`: findings graded Critical, Important, Minor, each with `file:line` and a failing scenario; then `VERDICT: APPROVED | CHANGES_REQUIRED`. End with `STATUS` and `REPORT` lines.
````
