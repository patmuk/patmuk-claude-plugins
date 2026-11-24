---
name: spec
description: Phase 2 of the gated task protocol - settles open decisions with the user in rounds, writes the specification into the plan doc and halts on a one-screen brief. Use after a gap report, or when the user asks for a spec or plan of a feature. Not Matt Pocock's to-spec.
---

# Spec

The rules are the project's general rules, `LLM_GENERAL/LLM_WAYS_OF_WORKING.md` §1.2; this is the procedure. The spec **records** decisions; it makes none. The user approves a **brief**, never the spec.

## Steps

1. **Settle the decisions.** Call the Skill tool with `mattpocock-skills:grilling`: every open decision of the gap report and every design choice with more than one defensible answer, in rounds, each question with your recommendation. Terms that are new or disputed: call `mattpocock-skills:domain-modeling`. Record each answer in the doc as `DECIDED <date>: …`.
   Done when no `!decide` that blocks the design is left. End the turn after each round.
2. **Agree the seams.** Propose where the tests sit: the public interfaces, the highest that reach the behaviour, existing ones first. One line each on what it catches and what it misses. Ask in the same round as the last decisions.
3. **Write `## Plan`** in the doc, in this order:
   - **Problem** — from the user's perspective;
   - **Solution** — from the user's perspective;
   - **User stories** — one per behaviour, "As a …, I want …, so that …"; each names the test scenario that proves it;
   - **Implementation decisions** — changes per layer, types first: ADTs and validating constructors before functions;
   - **Testing decisions** — the agreed seams, the scenarios per seam, prior art in the codebase;
   - **Slices** — vertical slices in build order, each `- [ ]`, each cutting through every layer it needs and verifiable on its own; say which tests a slice leaves red, if any. A mechanical change that touches everything at once is no slice: plan it as expand (add the new form beside the old), migrate in batches, contract;
   - **Out of scope**.
   Done when every user story has a scenario, every scenario a seam, and every implementation decision a slice.
4. **Self-verification.** Check the spec against each item of the ways of working's list (§1.2, Phase 2) and the project's `LLM/LLM_PROJECT.md` §Spec checks. Print one line per item: `checked <rule>: ok` or `checked <rule>: adjusted — <what>`. Adjust the spec first, then print.
5. **Snapshot**: `jj st`.
6. **Gate — the brief.** Print at most one screen: problem, solution, out of scope, the seams, the slice list (titles only), and every point where you decided against your own recommendation or the user's earlier wording. Then the self-verification lines. Ask exactly: "Do you approve this brief?" End the turn.

On approval: `pipeline:go-implement-it`.
