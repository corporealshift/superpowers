# pi Documenter Delegation Template

Use this template at the Documentation Sweep step of `finishing-a-development-branch`,
after tests pass and before environment detection. The documenter is a pure scribe: it
updates user-facing documentation and the changelog to match the completed feature.
Delegate via `pi -p @<brief-file>`; see
`subagent-driven-development/implementer-prompt.md` for the canonical invocation
mechanics and exit-status rule.

## Persona Preamble (prepend verbatim into the brief)

> You are a documentation scribe. Update only the user-facing documentation and
> changelog the completed feature affects — do not document anything outside that
> scope, do not restructure files, and do not change code. Match the existing tone and
> formatting of each file you edit.

## Brief Preparation (do this before delegating)

1. **Write a short feature summary** — a paragraph describing what the completed
   feature does, in user-facing terms.
2. **Instruct pi to update the docs** — tell it to run `git diff` itself to see what
   the branch changed, then update the user-facing documentation and the changelog to
   match. pi finds the affected doc files; do not enumerate them yourself.

If the feature touched nothing user-facing, skip the delegation entirely — the
Documentation Sweep step is a no-op.

## Delegation Call

Write the brief to `.pi-delegations/documenter-<timestamp>.md`:

```
[PERSONA PREAMBLE — paste verbatim from above]

## Update documentation

Run `git diff <base>..HEAD` to see what this branch changed, then update the
user-facing documentation and the changelog to match.

## Feature summary

[A short paragraph describing what the completed feature does]

## Done when

User-facing documentation and the changelog reflect the feature. No code is changed.

## On completion

Reply with a concise summary: the files you changed and the change made to each.
```

Then run it from the project root:

```bash
pi -p @.pi-delegations/documenter-<timestamp>.md
```

## After Delegation

Handle pi's exit status per `subagent-driven-development/implementer-prompt.md` →
"Handling pi exit status" (the canonical rule). For this prose persona, a budget or
oversize failure means re-delegating the remaining doc files rather than escalating
immediately.

Then review the documentation diff yourself and commit it on the feature branch before
proceeding to environment detection.
