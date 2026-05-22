# pi Planner Delegation Template

Use this template in the `writing-plans` skill, after Claude has done the Scope Check.
The planner produces the whole plan: the File Structure map, the task decomposition
(which tasks exist and in what order), and the full plan body. Only the Scope Check
stays with Claude; the Self-Review gate after delegation also stays with Claude.
Delegate via `pi -p @<brief-file>`; see
`subagent-driven-development/implementer-prompt.md` for the canonical invocation
mechanics and exit-status rule.

## Persona Preamble (prepend verbatim into the brief)

> You are an implementation planner. Read the spec yourself and decide the
> decomposition: map out which files to create or modify and what each is responsible
> for, then break the work into tasks — each a self-contained, independently testable
> change — in a sensible order. Expand every task into bite-sized steps: write the
> failing test, run it to confirm it fails, write the minimal implementation with real
> code, run the test to confirm it passes, commit. Every code step must contain actual,
> complete code — never "TBD", never "add error handling", never "similar to Task N".
> Use exact file paths and exact commands with their expected output.

## Brief Preparation (do this before delegating)

1. **Paste the Scope Check result** — confirm the spec is a single plan's worth of
   work (or state how it was scoped), so the planner does not re-litigate scope.
2. **Name the spec file path** — pi reads the spec itself; do not transcribe it.
3. **State the plan file path and the required plan header** —
   `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` plus the plan header block from
   the writing-plans skill.
4. **Name the conventions** — test framework, run commands, and commit-message style,
   so generated steps match the codebase.

pi can read an existing plan under `docs/superpowers/plans/` itself for format
reference — point it at one rather than handing it format boilerplate.

## Delegation Call

Write the brief to `.pi-delegations/planner-<timestamp>.md`:

```
[PERSONA PREAMBLE — paste verbatim from above]

## Write this implementation plan

Read the spec at `<spec path>`, then write the plan to `<exact plan path>`, starting
with the required header block below. Produce the File Structure map and the task
decomposition yourself, then expand every task into bite-sized steps. An existing plan
under `docs/superpowers/plans/` is a useful format reference — read one if it exists.

## Required plan header

[The exact header block from the writing-plans skill, filled in]

## Scope

[The Scope Check result — this spec is a single plan's worth of work]

## Conventions

[Test framework, run commands, commit-message style]

## Done when

The plan file exists at the path above, maps the file structure, decomposes the spec
into ordered tasks, expands every task into bite-sized steps with complete code blocks,
and contains no placeholders.

## On completion

Reply with a concise summary: the file you wrote and the list of tasks it contains.
```

Then run it from the project root:

```bash
pi -p @.pi-delegations/planner-<timestamp>.md
```

## After Delegation

Handle pi's exit status per `subagent-driven-development/implementer-prompt.md` →
"Handling pi exit status" (the canonical rule). For this prose persona, a budget or
oversize failure means re-delegating the plan section-by-section rather than
escalating immediately.

Then run the writing-plans Self-Review yourself on pi's plan — spec coverage,
placeholder scan, type consistency, implementer fit, and the decomposition check
(task boundaries, ordering, right-sizing). If you find issues, re-delegate a focused
fix or fix them inline.
