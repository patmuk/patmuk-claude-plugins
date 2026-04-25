# code-review-local

A read-only local code review for Claude Code, driven by a jj revset or a git refspec, with findings posted into the chat.
The first iteration was vibe coded.

## What it does

The `/code-review-local:review` command takes a target (a jj revset, a git refspec, or the literal keyword `all` for the whole tracked codebase; defaults to the current change), runs a multi-pass review pipeline against it, and prints findings back into your conversation. It never modifies the repository.

The pipeline:

1. Detects the VCS (`jj root`, falling back to `git rev-parse`; `--git` flag forces git).
2. Validates the target and collects the set of files it touches.
3. Has a small Haiku agent summarize the change's intent, so downstream agents can distinguish "the bug" from "the feature."
4. Runs three Sonnet review passes in parallel: **rules** (CLAUDE.md and conventions), **defects** (bugs visible from the diff alone), and **context** (file history plus inline code-comment guidance).
5. Has a Haiku validator score each candidate finding 1–10 and filters to ≥ 7.
6. Reports the surviving findings as the chat reply.

Each finding carries a kind (`rule`, `defect`, `history`, or `comment`), a summary, and an `evidence` field. Requiring concrete evidence per finding pushes back on hallucinated reviews — a finding without quotable supporting code or quoted rule is dropped before scoring.

## Why this exists

Anthropic's official `/code-review` plugin is excellent but built around the GitHub PR lifecycle: it expects the `gh` CLI, an open PR, and posts findings as PR comments. That doesn't fit when you:

- work primarily with jj (jujutsu), shaping history locally before — or instead of — pushing,
- want to review a change, or a stack, *before* it becomes a PR,
- hop between commits in a Claude-generated stack with `jj edit` and want a review of the one you're sitting on,
- use plain git and want a review of the last commit (or a branch, or a range) without opening a PR.

`code-review-local` covers that use case. It is not a port of the official plugin — it's a different tool inspired by the same idea (multi-pass review with confidence scoring) and shaped for local work.

| Concern                   | Official `/code-review`                  | `code-review-local`                          |
|---------------------------|------------------------------------------|----------------------------------------------|
| Requires                  | `gh` CLI + an open GitHub PR             | `jj` **or** `git`                            |
| Scope identifier          | the PR's branch                          | a jj revset or git refspec (argument)        |
| Findings posted to        | the PR as inline comments                | the Claude Code chat                         |
| Works on local changes    | no                                       | yes                                          |
| Works on feature branches | yes                                      | yes                                          |

Despite the "local" in the name, the plugin is not tied to trunk-based development. Any revset or refspec works: a feature branch (`main..feature-branch`), a whole local stack (`trunk()..@` in jj), a range in git (`HEAD~3..HEAD`), or a single change.

## Read-only guarantee

The plugin's `allowed-tools` whitelist permits only read-only jj and git subcommands: `jj log`, `jj diff`, `jj show`, `jj file annotate`, `jj file list`, `jj status`, `jj root`, `git log`, `git blame`, `git show`, `git diff`, `git rev-parse`, `git ls-files`, `git status`. Nothing on this list can change the repository, the working copy, the operation log, or any remote.

The command's prompt also explicitly forbids the corresponding write commands by name, so even if the model imagined a write would help (it shouldn't), the permission layer would reject it. Run the review against any change you like, including unfinished or experimental ones, without worry that it might rewrite history or push something.

## Installation

From this marketplace:

```text
/plugin marketplace add patmuk/patmuk-claude-plugins
/plugin install code-review-local@patmuk-plugins
```

## Usage

In a jj repository:

```text
/code-review-local:review              # reviews @ (the current change), or @- if @ has no file changes
/code-review-local:review @-           # reviews the parent change
/code-review-local:review trunk()..@   # reviews the whole local stack
/code-review-local:review kpqxywon     # reviews a specific change id
/code-review-local:review main..@      # reviews the current branch vs main
/code-review-local:review all          # reviews every tracked file in the working copy
```

In a plain git repository:

```text
/code-review-local:review              # reviews HEAD
/code-review-local:review HEAD~1       # reviews the second-to-last commit
/code-review-local:review HEAD~3..HEAD # reviews the last three commits
/code-review-local:review main..HEAD   # reviews the current branch vs main
/code-review-local:review abc1234      # reviews a specific commit
/code-review-local:review all          # reviews every tracked file in the working copy
```

The plugin auto-detects which VCS is in use and interprets the argument accordingly.

### Whole-codebase mode (`all`)

`all` is a special keyword (not a revset or refspec). Instead of a diff, the pipeline enumerates every tracked file (`jj file list` or `git ls-files`) and runs the review passes against the file contents directly:

- **Rules** still audits each file against its applicable `CLAUDE.md`.
- **Defects** reads the file (not a diff). Cross-file calls are fair game because the agent can `Read` the callee.
- **Context** keeps the inline-comments sub-pass and **skips** the history sub-pass — annotating every line of every file is not what `all` is for.
- **Intent** becomes a short codebase summary (from the top-level `README` and root `CLAUDE.md`) rather than a change description.

There is a **soft cap of 200 tracked files**. Above that, the command stops and asks you to narrow the scope, because the pipeline cost grows linearly with files and a whole-repo audit on a large codebase is better split into focused passes. Use a revset/refspec target instead — e.g., `trunk()..@` for everything you've added, or `main..HEAD` for a feature branch.

`--git all` works the same way and forces git mode.

### Auto-detection and the `--git` override

The VCS is decided *before* the argument is interpreted, in this order:

1. If `--git` appears anywhere in the argument string, it is stripped and the command is forced into git mode. If the repo has no git, the command errors out.
2. Else, `jj root` is tried. If it returns a path, the repo is treated as jj. (jj-colocated-with-git repos hit this branch — jj wins when both are present.)
3. Else, `git rev-parse --show-toplevel` is tried. If it returns a path, the repo is treated as plain git.
4. Else, the command stops with an error.

The argument is only interpreted once the VCS is known. That is why the default (`@` or `HEAD`) does not need to disambiguate — it is picked after detection.

In a jj-colocated repo, jj is preferred by default. Pass `--git` to force git when you specifically need it — for example, to exercise the git code path for testing, or to review a range expressed in pure git terms (`abc1234..def5678`) that doesn't translate cleanly to a jj revset:

```text
/code-review-local:review --git HEAD
/code-review-local:review --git main..HEAD
/code-review-local:review --git abc1234
```

There is no `--jj` flag, because jj is already preferred wherever it exists.

### Why the slash command is `/code-review-local:review`

Claude Code namespaces every plugin command as `/<plugin-name>:<command-name>`. For this plugin:

- plugin name: `code-review-local`
- command file: `commands/review.md`
- → invocation: `/code-review-local:review`

The official `/code-review` plugin is similarly namespaced as `/code-review:code-review`. Autocomplete prefixes diverge after the third character, so typing `/code-review` completes toward the official plugin while `/code-review-l…` completes toward this one — no collision.

## Verify your install with a scratch repo

To check that the plugin is wired up correctly, create a disposable repo with two intentional issues — one CLAUDE.md violation, one defect — and run the review against it. Expected outcome: both findings are reported, each above the score threshold, citing `buggy.py`.

### With jj

```sh
mkdir /tmp/cr-test-jj && cd /tmp/cr-test-jj
jj git init --colocate

cat > CLAUDE.md <<'EOF'
# Project rules

- All public Python functions must have a docstring.
- Never use a bare `except:` clause; always name the exception type.
EOF

jj commit -m "Add CLAUDE.md"

cat > buggy.py <<'EOF'
def process(data):
    try:
        return data["key"] * 2
    except:
        return None
EOF

jj commit -m "Add processing function"
```

Then in Claude Code, from `/tmp/cr-test-jj`:

```text
/code-review-local:review @-
```

Expected: the review flags both the missing docstring on `process` (kind `rule`) and the bare `except:` clause (kind `rule`, and possibly also `defect`).

### With plain git

```sh
mkdir /tmp/cr-test-git && cd /tmp/cr-test-git
git init -q

cat > CLAUDE.md <<'EOF'
# Project rules

- All public Python functions must have a docstring.
- Never use a bare `except:` clause; always name the exception type.
EOF

git add . && git commit -q -m "Add CLAUDE.md"

cat > buggy.py <<'EOF'
def process(data):
    try:
        return data["key"] * 2
    except:
        return None
EOF

git add . && git commit -q -m "Add processing function"
```

Then in Claude Code, from `/tmp/cr-test-git`:

```text
/code-review-local:review HEAD
```

Expected: same findings as the jj case.

If the review comes back with nothing flagged when you expect findings, check that the plugin is enabled in `/plugin`, and that you are pointing at the right target — `@-` for jj (the change that added `buggy.py`, since `@` is the empty working-copy change `jj commit` left behind), `HEAD` for git. Clean up with `rm -rf /tmp/cr-test-*` when done.

## Output format

The header names the VCS and the target so you can always tell which code path ran. Examples:

- `### Local code review — jj: \`@\``
- `### Local code review — jj: \`trunk()..@\``
- `### Local code review — git: \`HEAD\``
- `### Local code review — git: \`main..HEAD\``
- `### Local code review — jj: \`all\` (47 tracked files)`

A review with findings (jj target):

```text
### Local code review — jj: `@`

Found 2 finding(s):

**`src/auth.ts:L67-L72`** — defect
OAuth callback path returns before clearing the pending state, leaking it across sessions.
> ```ts
> if (!resp.ok) return;       // pending_state still set
> setSession(resp.body);
> clearPendingState();
> ```

**`src/auth.ts:L88-L95`** — rule
Catches `Exception` broadly; CLAUDE.md requires named exception types.
> CLAUDE.md: "Never catch bare Exception; name the specific exception class."
```

A review with findings (git target):

```text
### Local code review — git: `HEAD`

Found 1 finding(s):

**`src/retry.py:L23-L27`** — defect
Loop terminates one iteration too early because the bound uses `<` where the inclusive form is intended.
> ```py
> for attempt in range(max_attempts - 1):
>     ...
> ```
```

A review with nothing to flag:

```text
### Local code review — jj: `@`

Nothing flagged. The pipeline ran rules, defects, and context passes; no finding scored ≥ 7.
```

`path:Lstart-Lend` references render as clickable links in Claude Code's terminal and don't depend on commit SHAs, so they keep working after `jj squash` / `jj rebase` or `git commit --amend`.

## Acknowledgements and licensing

This plugin's design is inspired by Anthropic's official `/code-review` plugin and the broader "have an LLM review the diff before a human does" pattern. The specific structure here — three review passes with a separate validator, an explicit `evidence` field, a 1–10 score with a 7 threshold, the `kind`-tagged output format — is this plugin's own.

The plugin is licensed under the MIT License. See [`LICENSE`](../../LICENSE) at the repository root.

## Roadmap

- **Per-repo `REVIEW.md` support.** Match the managed Code Review product's pattern of letting a repo file override review priorities (severity calibration, what counts as Important vs Nit).
- **Optional write-to-file.** For long reviews, also emit to `.claude/reviews/<slug>.md` in addition to chat output.
