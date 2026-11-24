---
name: gap-report
description: Phase 1 of the gated task protocol - a gap report verified against the code, written into the work item's plan doc. Use when a feature, refactor or non-trivial bug fix starts, when the ways of working's gate triggers hold, or when the user asks what the code does today against what it must do.
---

# Gap report

The rules are the project's general rules, `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1 and §3; this is the procedure. A claim counts only when **verified**: you read the code that proves it in this session.

## Steps

1. **Locate the work item.** `./todo.md` → the linked doc, or the `plan/…` doc the user names. No doc yet: create one in the fitting `plan/todos/<category>/` and add its line to `./todo.md` (major items only).
   Done when you can name the doc's path.
2. **Read the area.** The code the item touches, the matching section of `LLM/LLM_AREA_PATTERNS.md`, the decisions in `plan/design_decisions/` on the topic, and `GLOSSARY.md` if it exists.
   Done when every statement you are about to write is verified, and each doc claim you relied on was checked against the code.
3. **Write `## Current state`** in the doc, short:
   - what the code does today against what it must do, and which scenarios the tests cover;
   - hidden edge cases, one line each — walk every kind listed in `LLM/LLM_PROJECT.md` §Edge cases and write "none" where a kind does not apply;
   - conflicts between docs and code, both sides quoted with their paths;
   - open product decisions as `!decide <question>` lines.
   Done when every kind of edge case has a line and every `!decide` is a question a user can answer.
4. **Snapshot**: `jj st` in the workspace you wrote in.
5. **Gate.**
   - A `!decide` or a conflict exists → call the Skill tool with `mattpocock-skills:grilling` and ask all of them as one round, each with your recommended answer. Print the report above the questions. End the turn.
   - Neither → print the report and call the Skill tool with `pipeline:spec` in the same turn.

A decision too large for one round (it opens further decisions) is `pipeline:decide`'s.
