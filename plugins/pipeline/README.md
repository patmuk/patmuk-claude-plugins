# pipeline

A gated task pipeline as skills, for jj repositories whose work is tracked in plan docs (`./todo.md`, `plan/todos/`) and whose rules come from the `llm_rules` general rules (`LLM_RULES.md`, `LLM_GENERAL/`, `stacks/`). The rules say when a gate applies and what a skill may not override; these skills are the procedures.

| Skill | When | Ends with |
|---|---|---|
| `/pipeline:gap-report` | a feature, refactor or non-trivial fix starts | a report in the plan doc; halts only on an open decision or a doc/code conflict |
| `/pipeline:spec` | after the gap report | the spec in the plan doc; halts on a one-screen brief |
| `/pipeline:go-implement-it` | the brief is approved, or you supply the steps | all slices built and verified, then `wrap-up` |
| `/pipeline:wrap-up` | all slices built (also for work built by hand) | two-axis review fixed, plan doc and todo index updated, the report |
| `/pipeline:decide` | a decision too large for one round of questions | one decision question resolved per session |
| `/pipeline:device-round` | testing a build on devices with the agent | findings handed to `gap-report` |

Each skill can be typed, and the agent starts it when the ways of working name it.

## Against Matt Pocock's engineering skills

The pipeline replaces his pipeline skills and calls his thinking skills. Install his plugin (`mattpocock-skills`) unmodified beside this one.

| His (installed, not used here) | This plugin | Difference |
|---|---|---|
| — | `gap-report` | analysis verified against the code; halts only on `!decide` / conflict |
| `to-spec` | `spec` | decisions settled in rounds first; his sections plus slices; approval of a brief, not the spec; lives in the plan doc |
| `to-tickets` | inside `spec` | the slice list in the plan doc, no ticket files |
| `implement`, `implement-spec` | `go-implement-it` | vertical slices in one jj workspace, verify per slice, linear stack, no branches or merges |
| `code-review` | `wrap-up` | two axes through jj, findings folded into the owning commits, bookkeeping and report |
| `wayfinder` | `decide` | the map is a section layout in the todo doc, no issue tracker |
| `triage` | — | markers in the todo docs (`!triage`, `!human`, `DROPPED`) |
| `pr` | — | no pull-request flow |
| — | `device-round` | a device test round with a monitor subagent |

Called by name from these skills: `mattpocock-skills:grilling`, `:domain-modeling`, `:tdd`, `:diagnosing-bugs`, `:prototype`, `:research`.

The names differ on purpose, so typing one never starts the other: `/spec` is this plugin's, `/to-spec` is his.

## Live edits

A marketplace added from a local path with relative plugin sources loads its plugins in place: an edit is live at the next session or after `/reload-plugins`. Added from GitHub, the plugin is copied into the cache and an edit needs a push and `/plugin update`.

```text
/plugin marketplace add <path to this repository>
/plugin install pipeline@patmuk-plugins
```
