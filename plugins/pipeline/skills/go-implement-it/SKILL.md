---
name: go-implement-it
description: Phase 3 of the gated task protocol - builds an approved spec or a supplied step list in vertical slices in a jj workspace, verifying each slice, then wraps up. Use when the user says go, go ahead, implement it, build it, or approves a brief. Not Matt Pocock's implement.
---

# Go implement it

The rules are the project's general rules, `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1.2 (Phase 3), §2 and §4; this is the procedure. One **slice** at a time: narrow, through every layer it needs, verified before the next starts.

## Steps

1. **The spec.** The plan doc's `## Plan` with its slices, or the step list the user supplied. Neither exists and a gate trigger holds (§1.1) → `pipeline:gap-report` instead.
2. **The workspace.** `jj workspace list`; create the task workspace and describe its change (§2.3), or work in the user's `@` when they asked for that (§2.4). Run what `LLM/LLM_PROJECT.md` §Fresh workspace says before the first build or format.
   Done when a build of the untouched tree passes there.
3. **Read before writing.** `LLM_GENERAL/LLM_CODING_STYLE.md`, the `stacks/` files of the languages the slices touch, and the area patterns of the touched areas.
4. **Per slice:**
   1. call the Skill tool with `mattpocock-skills:tdd`: one test at an agreed seam, red; the code that makes it green; again until the slice's scenarios are covered. The scenario stands as a comment above its test;
   2. run the verify commands of every touched area (`LLM/LLM_PROJECT.md` §Verify) and format the touched files;
   3. a red test the spec predicted → note it. Any other red test or compile error → fix it now and state the error and the fix. A failure you cannot explain after one look → call `mattpocock-skills:diagnosing-bugs`. A fix that needs a design change → stop, report, back to `pipeline:spec` for that part;
   4. commit: `jj commit -m "type(feature/scope): what the slice delivers"`; a topic the slice does not need is its own commit (§2.5). `jj log -r "conflicts()"` is empty;
   5. tick the slice's `- [ ]` in the plan doc.
   Done when the slice's verify commands exit clean, or the only red tests are the ones the spec named.
5. **Wrap up.** Call the Skill tool with `pipeline:wrap-up`. The work is not reported before it ran.

No approval is asked between slices. Stop only for a design change, a `!decide` the spec missed, or an action the rules reserve for the user (a push, a commit that is not yours).
