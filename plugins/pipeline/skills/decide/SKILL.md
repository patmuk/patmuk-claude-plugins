---
name: decide
description: Works a large open decision as a map of decision questions in its todo doc, one question per session, until nothing is left to decide before a spec. Use for a "!decide" item too big for one round of questions, a loose idea whose way to a spec is not visible yet, or when the user asks to chart or continue such a map. Not Matt Pocock's wayfinder.
---

# Decide

For an effort too large or too foggy for one session. The output is **decisions**, never code: when the pull to build arrives, the map is done and `pipeline:gap-report` takes over. The layout rules are the project's general rules, `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §3.

## The map

The effort's todo doc, `plan/todos/<category>/<effort>.md`, with these sections:

```markdown
## Destination
<what the end of this map looks like: a spec to write, a decision to lock. One or two lines.>

## Notes
<standing preferences, skills every session should use>

## Decisions so far
- <question title>: <one-line gist> (DECIDED <date>)

## Open questions
### 01 — <question>
Type: research | prototype | grilling | task
Status: open | claimed | resolved
Blocked by: <numbers, or none>

## Not yet specified
<what you can tell is coming but cannot phrase as a sharp question yet>

## Out of scope
<ruled out of this effort, one line each with the reason>
```

A question is a **ticket** when it can be stated precisely now, even if it is blocked; otherwise it stays in "Not yet specified". Refer to a question by its title, not its number.

## Chart (the user brings a loose idea)

1. Call `mattpocock-skills:grilling` and `mattpocock-skills:domain-modeling` to name the destination.
2. Grill again, breadth-first: surface the open decisions and what is takeable now. No fog found — the way is clear and fits one session → no map; say so and offer `pipeline:gap-report`.
3. Write the map: destination, notes, the questions you can state now with their blockers, the rest under "Not yet specified". Add the line to `./todo.md`.
4. Start a background subagent with `mattpocock-skills:research` for each `research` question; its findings go into a file beside the doc, linked from the question.
5. `jj st`. Stop: charting resolves nothing by hand.

## Work (the user names a map)

1. Read the map — not every resolved answer.
2. Take the question the user names, else the first that is open, unblocked and unclaimed. Set `Status: claimed`, `jj st`.
3. Resolve it by its type: `research` → a subagent with `mattpocock-skills:research`; `prototype` → `mattpocock-skills:prototype`, with the user; `grilling` → `mattpocock-skills:grilling` and `mattpocock-skills:domain-modeling`, with the user — you never answer for them; `task` → do it, or hand the user a checklist (`!human`).
4. Record: the answer under the question, `Status: resolved`, a line in "Decisions so far". A decision that outlives the effort also goes to `plan/design_decisions/<topic>.md`.
5. Graduate what the answer made sharp from "Not yet specified" into new questions; move what turned out beyond the destination to "Out of scope"; correct questions the answer invalidated.
6. `jj st`. **One question per session**, research excepted. No open question and nothing unspecified left → say the map is done and name the next step.
