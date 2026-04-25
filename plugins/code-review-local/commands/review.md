---
allowed-tools: Bash(jj log:*), Bash(jj diff:*), Bash(jj show:*), Bash(jj file annotate:*), Bash(jj status:*), Bash(jj root:*), Bash(git log:*), Bash(git blame:*), Bash(git show:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git status:*), Read, Grep, Glob
argument-hint: "[--git] [revset-or-refspec]"
description: Read-only local code review of a jj revset or git refspec
disable-model-invocation: false
---

# Local code review

Run a local-only code review on the change identified by `$ARGUMENTS`. Output is plain text in the chat. Nothing is committed, sent, or modified.

## Read-only constraint

This command must never invoke any jj or git subcommand that mutates state. The `allowed-tools` frontmatter restricts the bash surface to a small set of read-only commands, and the model must not attempt to circumvent it. If at any point the review approach seems to require a write command, abandon that approach — do not attempt the write.

For clarity, the following are explicitly out of scope:

- jj writes: `new`, `commit`, `describe`, `abandon`, `squash`, `split`, `rebase`, `edit`, `restore`, `move`, `absorb`, `resolve`, `bookmark`, `workspace`, `undo`, `op restore`, `op abandon`, `git push`, `git fetch`, `git clone`, `git init`, `fix`, `sign`, `parallelize`, `metaedit`, `duplicate`, `backout`, `config set`, and any interactive (`-i` / `--interactive`) variants
- git writes: `commit`, `add`, `rm`, `mv`, `push`, `fetch`, `pull`, `clone`, `checkout`, `switch`, `restore`, `merge`, `rebase`, `cherry-pick`, `revert`, `branch`, `tag`, `reset`, `stash`, `clean`, `gc`, `prune`, `notes`, `config`, `remote`, `submodule`, `worktree`

Even when one of these would be convenient, it stays off the table.

## Argument

`$ARGUMENTS` may contain:

1. An optional `--git` flag (anywhere in the string) which forces git mode.
2. A revset (jj) or refspec (git). Optional. Defaults below.

Examples:

- *(empty)* — review the current change (`@` in jj, falling back to `@-` if `@` has no file changes; `HEAD` in git)
- `@-` — jj: parent of the working copy
- `HEAD~2` — git: two commits back
- `trunk()..@` — jj: every change above trunk
- `main..HEAD` — either: changes on the current branch relative to main
- `--git HEAD~3..HEAD` — force git mode, review the last three commits
- `--git abc1234` — force git mode, review the commit with that SHA prefix

## Pipeline

### A. Set up

1. Parse `$ARGUMENTS`. Strip a `--git` token if present and remember whether it was there. Whatever remains is the **target**.
2. Decide the VCS:
   - If `--git` was present: VCS is git. Verify with `git rev-parse --show-toplevel`. If that fails, stop and reply: "`--git` was specified but no git repository is present here."
   - Else, try `jj root`. Success → VCS is jj. (This includes jj-colocated-with-git repos. jj is the preferred view when both exist.)
   - Else, try `git rev-parse --show-toplevel`. Success → VCS is git.
   - Else, stop and reply: "This directory is not inside a jj or git repository."
3. Apply the default target if the user did not supply one:
   - jj: default to `@`. If `jj diff -r "@" --name-only` is empty, fall back to `@-` and note this in the final report header (e.g., `jj: @- (fell back from empty @)`). If `@-` is also empty, stop and reply: "No file changes in `@` or `@-`."
   - git: default to `HEAD`.
4. Validate the target exists and contains changes:
   - jj: `jj log -r "<target>" --no-graph --template 'change_id ++ "\n"'` should succeed and return at least one row. Then `jj diff -r "<target>" --name-only` should be non-empty. (When the target was chosen by the default-fallback in step 3, this is already known.)
   - git: if the target contains `..`, treat it as a range; validate with `git log <target> --oneline -1`. Otherwise treat it as a single commit-ish, validate with `git rev-parse --verify <target>`, and define the effective range as `<target>^..<target>`. Then `git diff --name-only <range>` should be non-empty.
   - On any failure, stop and reply with the precise reason ("Target `xyz` does not resolve.", "No file changes in `xyz`.", etc.).
5. Locate `CLAUDE.md` files. Walk from the repo root and from each directory containing a touched file. Collect the set of CLAUDE.md paths encountered. If none exist, that's fine — note it for the rules pass.

### B. Gather intent

Spawn one Haiku agent that returns a 2–4 sentence summary of the change. The agent reads:

- The change descriptions (`jj log -r "<target>" --no-graph` for jj, `git log <range> --no-merges` for git)
- The shape of the change (`jj diff -r "<target>" --stat` or `git diff <range> --stat`)
- The diff itself (`jj diff -r "<target>"` or `git diff <range>`), truncated if very large

The summary should cover *what* changed and, if discernible, *why*. It is the only context every downstream agent receives about author intent.

### C. Three parallel review passes

Spawn three Sonnet agents simultaneously. Each receives: the intent summary, the diff, and the list of CLAUDE.md paths from step A.5. Each returns a list of candidate findings using this shape:

```
{
  "file": "path/to/file.py",
  "lines": "L42-L48",
  "kind": "rule" | "defect" | "history" | "comment",
  "summary": "<one sentence describing the issue>",
  "evidence": "<the specific code or quoted rule that justifies flagging>"
}
```

The `evidence` field is required — a finding without concrete evidence is dropped before validation.

#### Pass 1 — Rules

Audit the change against the collected CLAUDE.md files and any conventions visible in unchanged neighboring code.

For each touched file, only consider CLAUDE.md files in that file's directory or one of its ancestors. CLAUDE.md is meant to guide code *generation*; not every line of it is a review criterion. Flag a candidate only when the change visibly deviates from a rule the agent can quote. Set `evidence` to the quoted rule plus the conflicting code snippet.

#### Pass 2 — Defects

Read the diff (only the diff — no extra context) and look for concrete defects: off-by-one errors, null/undefined hazards, race conditions, resource leaks, swapped arguments, miscalculated indices, missing await, unhandled exceptions where the calling pattern requires handling.

Flag only what the agent can confirm from the diff alone. If verifying a candidate would require reading code not in the diff, skip it.

Set `evidence` to the relevant snippet from the diff.

#### Pass 3 — Context

Two sub-tasks, run together:

1. *History.* For each touched file, run `jj file annotate <file>` (jj) or `git blame <file>` (git) on the modified line ranges, plus `jj log <file>` or `git log --follow <file>`. Look for: regressions (a fix being undone), broken invariants (a contract this file historically maintained), or repeated past mistakes (a pattern previously corrected and now reintroduced).
2. *Comments.* Read inline code comments (`//`, `#`, `/* */`, etc.) inside or immediately adjacent to the modified ranges. Comments often encode constraints the change might violate ("don't call this from outside the lock", "must precede `init()`", "leave this `+ 1` — it accounts for the header byte").

Use `kind` of `history` or `comment` accordingly.

### D. Validate and score

For each candidate finding, spawn a Haiku validator agent. The validator receives:

- The intent summary
- The candidate finding (including its `evidence`)
- The list of CLAUDE.md paths

The validator's job is to score the finding 1–10 by how confident it is that the finding is real *and* worth surfacing. Use this rubric:

- **1–2** — false positive on inspection. The cited code does not actually have the claimed problem, or the claimed rule does not say what the finding claims.
- **3–4** — stylistic only. The finding describes a preference not grounded in any rule the validator can locate, and not rising to the level of a defect.
- **5–6** — possibly real, partially verified. The validator can see how the finding might apply but cannot conclusively confirm it from available information.
- **7–8** — verified and material. The validator confirmed the cited evidence supports the finding, and the issue is meaningful in practice.
- **9–10** — certain and high-impact. The validator independently re-derived the issue and is confident it will affect behavior.

For findings with `kind: rule`, the validator must verify that the cited CLAUDE.md actually contains the rule attributed to it.

Filter to findings with score ≥ 7.

### E. Report

Emit the result as the assistant's final message in the chat. Do not run any external posting tool. Format:

When findings remain after filtering:

```
### Local code review — <vcs>: `<target>`

Found N finding(s):

**`path/to/file.ext:Lstart-Lend`** — <kind>
<summary>
> <evidence>

**`other/path.ext:Lstart-Lend`** — <kind>
<summary>
> <evidence>
```

When all findings filter out, or there were no candidates:

```
### Local code review — <vcs>: `<target>`

Nothing flagged. The pipeline ran rules, defects, and context passes; no finding scored ≥ 7.
```

Substitute `<vcs>` with literal `jj` or `git`.

## Notes

- Don't try to build, lint, type-check, or run tests. Those are separate from review.
- `path:Lstart-Lend` line references render as clickable links in Claude Code's terminal and survive history rewrites (no commit SHA in the path).
- For jj targets that span multiple changes, line numbers refer to the file state at `heads(<target>)`.
- For git ranges, line numbers refer to the file state at the second ref of the range (or the single ref if the target was not a range).
- Do not stack. A given line range should appear under one finding kind, owned by the first pass to identify it.

## False positive guidance

Skip findings that match any of:

- The flagged line is unchanged by the target
- The issue would be caught by the project's linter, type checker, formatter, or compiler if those exist
- The "rule" being cited is the agent's own preference rather than text in CLAUDE.md
- The flagged behavior is the entire purpose of the change (it's the feature, not a bug)
- The flagged code has an explicit silencing marker (e.g., `// noqa`, `/* eslint-disable */`, `// CLAUDE.md exception: …`)
- The agent cannot describe the harm in concrete terms — "this could potentially be a problem" is not enough
- The finding restates a tautology of the change ("you removed the null check" — yes, that's what the change is)
