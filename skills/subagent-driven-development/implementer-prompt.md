# pi Implementer Delegation Template

Use this template when delegating an implementation task to the `pi` coding agent.
This file is the **single source of truth** for pi invocation mechanics and
exit-status handling — `planner-prompt.md`, `feature-designer-prompt.md`, and
`documenter-prompt.md` cross-reference the two sections below rather than restating
them.

pi runs the user's locally configured model — this is a delegation transport, not a
model choice. Never pass `--provider` or `--model`.

## Persona Preamble (prepend verbatim into the brief)

> You are an implementer. Implement the assigned task TDD-style: write the failing
> test first, run it and confirm it fails because the implementation is missing,
> write the minimal implementation, and make the tests green. Commit when the tests
> pass. Read the plan and the spec yourself for the detail you need — do not wait to
> be handed file contents.

## Brief Preparation (do this before delegating)

1. **Resolve genuinely blocking ambiguity.** pi cannot ask questions in print mode.
   If a decision would block progress, resolve it from context or ask the user (one
   question at a time) and state the resolution in the brief. Leave non-blocking
   judgment calls to pi — the review step catches bad ones.
2. **Name the source-of-truth files.** Give the plan path and the spec path. pi reads
   them itself with its own `read` tool; do not transcribe their contents into the
   brief.

## Delegation Call

Write the brief to `.pi-delegations/implementer-<timestamp>.md`:

```
[PERSONA PREAMBLE — paste verbatim from above]

## Task

Implement Task <N> from `<plan path>` TDD-style. Read that task and its steps in the
plan, and read the spec at `<spec path>` for context.

## Notes

[Any blocking ambiguity you resolved, stated plainly. Omit this section if none.]

## Done when

The task's tests are green and the work is committed.

## On completion

Reply with a concise summary of what you implemented and the test result.
```

Then run it from the project root:

```bash
pi -p @.pi-delegations/implementer-<timestamp>.md
```

One blocking call per delegation, runs to completion, exits. With no
`--continue`/`--resume`, each call starts a fresh session — every delegation gets
clean, intentional context.

## Handling pi exit status (canonical)

pi's stdout is the delegate's summary text. The process exit code signals outcome:

| Exit code | Action |
|---|---|
| 0 | Delegation completed. Proceed to review the artifact. |
| non-zero | Failed. Read stdout/stderr for the error. Retry once if it looks transient. Decompose and re-delegate if the task was too big. Otherwise escalate to the user, pointing at the brief under `.pi-delegations/` and the latest pi session under `~/.pi/agent/sessions/`. |

Verify results by inspecting the repo (`git status`, `git diff`) and running the
tests yourself — not by trusting the summary text.

## Fix-Loop Context Discipline

When re-delegating after a reviewer finds issues, write a fresh brief that names only
the specific change required and the file(s) it affects. Do not paste the original
task plus the full review into the brief. If a review surfaces multiple unrelated
fixes, split them into separate delegations.
