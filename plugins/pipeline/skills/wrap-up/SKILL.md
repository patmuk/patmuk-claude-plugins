---
name: wrap-up
description: Phase 4 of the gated task protocol - reviews a finished stack on two separate axes with fresh subagents through jj, fixes the findings, updates the plan doc and todo index, cleans the workspace and reports. Use when all slices are built, or when the user asks to wrap up, finish or close out work they or an ad-hoc session built.
---

# Wrap up

The rules are the project's general rules, `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1.2 (Phase 4), §2 and §3; this is the procedure. Two **axes**, reviewed apart and reported apart: a stack can follow every rule and build the wrong thing, or the reverse.

## Steps

1. **Pin the stack.** The revset of the work: `<base>..<tip>` in the task workspace, or what the user names. `jj st` first, so the review reads what is on disk. `jj log -r '<revset>'` is non-empty.
2. **Find the spec.** The plan doc's `## Plan` (or the user's step list). None → the spec axis is skipped and the report says so.
3. **Two fresh subagents, in one message, read-only** (`jj log`, `jj diff -r`, `jj file show` only; they change nothing and run no build):
   - **Spec axis** — given the revset and the plan doc's path: "Report (a) what the spec asks for that is missing or partial, (b) behaviour in the diff nobody asked for, (c) what looks implemented but wrong. Quote the spec line for each. Under 400 words."
   - **Rules axis** — given the revset and the paths of `LLM_GENERAL/LLM_CODING_STYLE.md`, `LLM_GENERAL/LLM_DESIGN_PRINCIPLES.md`, the touched languages' `stacks/` files, `LLM/LLM_REPO_MAP.md` and the touched sections of `LLM/LLM_AREA_PATTERNS.md`: "Report every place the diff breaks a documented rule; cite file and rule, quote the hunk. Separate hard breaches from judgement calls. Skip what a linter enforces. Under 400 words."
   Nobody else edits the tree while they run.
4. **Fix the findings.** Each fix goes into the own-session commit that introduced the thing (`jj squash --into <change> --use-destination-message`, or edit and absorb); a finding about a commit that is not yours becomes a new commit on top that cites it. A finding you reject is listed in the report with the reason. A finding that needs a design change goes back to `pipeline:spec`.
5. **Final verify.** The last row of the project's verify table, formatting of touched files, `jj log -r "conflicts()"` empty.
   Done when every command exits clean; paste the commands with their results into the report.
6. **Bookkeeping.** The plan doc: slices ticked, status updated; it moves to `plan/done/` when nothing but a device test is left, and `./todo.md` follows. A todo the review found and you did not fix gets a doc or a line with `!triage`. A lesson that holds beyond this task goes to its owning doc (§3).
7. **Workspace.** Forget and remove the task workspace and its outside build caches (§2.3), from the default directory. `jj log` shows the commits.
8. **Report**, in this order: the commits (change id + what each delivers); the verify commands and results; `## Spec` findings and `## Rules` findings as the subagents gave them, each marked fixed / rejected / open; every deviation from the spec; what the user must do by hand (`!human`).
