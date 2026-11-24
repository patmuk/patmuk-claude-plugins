---
name: device-round
description: Runs a device test round with the user operating the devices - a monitor subagent keeps captures and a timeline, questions wait, findings become the next gap report. Use when the user starts testing a build on phones or simulators with you, or asks for a test round.
---

# Device round

The reference is the project's general rules, `LLM_GENERAL/LLM_USER_TESTING.md` — read it in full now, then the project's `LLM/LLM_PROJECT_USER_TESTING.md` for the devices, the capture commands and the platform steps. This skill is the order of events.

## Steps

1. **State the build.** Which commit is on each device, and which plan docs wait under "to be tested" in `./todo.md`: their check lists are the round's agenda.
   Done when the user has confirmed the agenda or changed it.
2. **Start the monitor** as the reference describes: one background subagent that owns the captures and the timeline. It runs no jj command.
   Done when it has reported that every capture file grows.
3. **The round.** Relay each step the user reports to the monitor with its time. Collect the user's questions and observations in a list; answer none that needs analysis — say it waits. Leave the code alone.
4. **Close the round** when the user says so: stop the monitor, take its timeline and the capture paths.
5. **Findings.** Each observation becomes a finding checked first against the tests and the plan docs: covered and passing there → the gap is in the test or the build; not covered → a `!decide` or a new todo. Tick what the round proved in the plan docs; move a doc whose list is done.
6. **Hand over.** Call `pipeline:gap-report` with the findings as its input; the timeline and capture paths go into its `## Current state`.
