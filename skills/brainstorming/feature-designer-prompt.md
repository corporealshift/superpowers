# pi Feature Designer Delegation Template

Use this template at step 6 of the `brainstorming` skill ("Write design doc"), after
the design has been approved with the user (step 5). Every design decision is already
made; the feature designer's job is to turn the approved design into a well-structured
spec document. Delegate via `pi -p @<brief-file>`; see
`subagent-driven-development/implementer-prompt.md` for the canonical invocation
mechanics and exit-status rule.

## Persona Preamble (prepend verbatim into the brief)

> You are a feature designer. You are writing a design specification document from a
> complete set of decisions that have already been made and approved. Every design
> question is settled — do not invent requirements, change scope, or add features.
> You have latitude on how to structure the document and how to word it: organize the
> sections clearly, write in plain technical prose, and make the spec easy to read.
> If something is genuinely missing or contradictory, note it explicitly at the end
> under "Open questions for Claude" rather than guessing.

## Brief Preparation (do this before delegating)

The approved design lives in this conversation, which pi cannot read — so the brief
must carry the full design substance.

1. **Collect every approved design section** — architecture, components, data flow,
   error handling, testing, non-goals. Paste the full substance of each into the
   brief. The feature designer must not have to reconstruct decisions.
2. **List the resolved decisions and tradeoffs** — for each significant choice, state
   what was chosen and what was rejected, so the spec records the reasoning.
3. **State the exact spec file path** — `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`.
4. **Provide the metadata** — date, author, status line.

pi can read an existing spec under `docs/superpowers/specs/` itself for format
reference — point it at one rather than handing it format boilerplate.

## Delegation Call

Write the brief to `.pi-delegations/feature-designer-<timestamp>.md`:

```
[PERSONA PREAMBLE — paste verbatim from above]

## Write this spec document

Write the design specification to `<exact spec path>`. An existing spec under
`docs/superpowers/specs/` is a useful format reference — read one if it exists.

## Approved design (full substance)

[Every approved section, with all decisions inline]

## Resolved decisions and tradeoffs

[Each choice: what was chosen, what was rejected, why]

## Document metadata

Date / Author / Status: [values]

## Done when

The spec file exists at the path above, covers every section listed, contains no
"TBD"/"TODO"/placeholder text, and records the resolved decisions. No code is written.

## On completion

Reply with a concise summary: the file you wrote, the sections it contains, and
anything you flagged under "Open questions for Claude".
```

Then run it from the project root:

```bash
pi -p @.pi-delegations/feature-designer-<timestamp>.md
```

## After Delegation

Handle pi's exit status per `subagent-driven-development/implementer-prompt.md` →
"Handling pi exit status" (the canonical rule). For this prose persona, a budget or
oversize failure means re-delegating the spec section-by-section rather than
escalating immediately.

Then run brainstorming step 7 (spec self-review) yourself on pi's draft — the
placeholder, consistency, scope, and ambiguity checks. If you find issues, re-delegate
a focused fix or fix them inline.
