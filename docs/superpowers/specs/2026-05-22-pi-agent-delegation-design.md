# Pi Agent Delegation Across the Development Workflow

**Date:** 2026-05-22
**Author:** Kyle Diedrick (with Claude)
**Status:** Approved, ready for implementation

## Problem

The superpowers development workflow delegates production work — writing the design
spec, writing the implementation plan, authoring tests, implementing code, and
updating documentation — to a local model via the `mcp__llama-mcp__delegate_to_llama`
MCP tool. Five personas use it: feature designer (in the `brainstorming` skill),
planner (in `writing-plans`), test author and implementer (in
`subagent-driven-development`), and documenter (in
`finishing-a-development-branch`).

That MCP server is a weak, small-context, non-agentic executor. The skills were
designed around its limitations: the controller (Claude) had to meticulously curate
every scrap of context, transcribe full task text into the brief, never let the
executor read a file itself, resolve every ambiguity up front, and split tasks small
to fit a tiny context budget.

The user now has the `pi` coding agent installed (v0.75.4). pi is a full coding agent:
it reads files, runs git and cargo and build commands with its own tools, and reasons
well. Critically, pi is configured to run the same local model the MCP server used
(provider `llama`, model `Qwen3.6-35B-A3B`) — so this is a transport change, not a
model change. The MCP-era constraints in the skills are now unnecessary overhead.

## Goals

- Replace the `llama-mcp` delegation transport with the `pi` CLI agent.
- Remove the context-curation constraints that only existed because the MCP executor
  was weak. The controller's role shifts from context-curator to orchestrator and
  reviewer.
- Collapse the test-author and implementer personas into a single **implementer**
  delegation, since pi is capable enough to do task TDD-style within one delegation.
- Preserve the existing two-stage Claude review of code artifacts (spec compliance,
  then code quality) as the quality backbone.

## Non-goals

- **No model change.** pi runs the user's already-configured local model; no
  `--provider` or `--model` flag is passed.
- **No new skill.** Personas stay in their existing home skills.
- **No change to the two-stage Claude review** of code artifacts. That review is the
  quality backbone, not an MCP workaround.
- **No use of pi's RPC mode or SDK mode.** Print mode only.

## Personas

| Persona | Home skill | Trigger point | Judgment latitude |
|---|---|---|---|
| Feature designer | `brainstorming` | Step 6 — "Write design doc" | Highest: latitude on doc structure and prose |
| Planner | `writing-plans` | After scope check, before self-review | Moderate: expands scope into a decomposed plan |
| Implementer | `subagent-driven-development` | Per task, the only delegation | Moderate: writes failing test, implements, makes green |
| Documenter | `finishing-a-development-branch` | Documentation Sweep step | Low: identifies which docs the branch diff touches |

**Division of labor.** Claude always makes the judgment calls and composes the brief.
pi produces the artifact. The personas sit at different points on a judgment spectrum:

- **Documenter** — low latitude. Claude gives a short feature summary; pi reads the
  branch diff, identifies which user-facing docs and changelog entries to update, and
  writes the changes.
- **Feature designer** — most latitude. Claude supplies all substance and every
  resolved decision and tradeoff; pi has latitude on how to structure the document
  and word it.
- **Planner** — moderate. Claude locks the scope; pi decomposes it into bite-sized
  steps with real code blocks.
- **Implementer** — moderate. Claude names the task and the acceptance criteria; pi
  writes the failing test, implements, and makes the tests green within one
  delegation.

## Architecture

The controller stops being a context-curator and becomes an orchestrator and reviewer.
Every delegation follows the same shape:

1. The controller composes a thin brief — a pointer to the source-of-truth file
   (spec, plan, or branch diff) plus the unit of work. pi reads the spec, plan, or
   diff itself; the controller does not transcribe file contents into the brief.
2. The controller writes the brief to a gitignored `.pi-delegations/` directory as
   `<persona>-<timestamp>.md` and runs `pi -p @<brief-file>` from the project root.
3. pi runs the local model to completion, using its own `read`/`bash`/`edit`/`write`/`git`
   tools.
4. On exit 0 the controller reviews the artifact; on exit non-zero it applies the
   exit-status handling described below.

Each persona gets one prompt file alongside its home skill's `SKILL.md`, mirroring the
existing `implementer-prompt.md` pattern:

- `skills/brainstorming/feature-designer-prompt.md`
- `skills/writing-plans/planner-prompt.md`
- `skills/subagent-driven-development/implementer-prompt.md`
- `skills/finishing-a-development-branch/documenter-prompt.md`

Each prompt file contains three parts:

1. **Persona preamble** — the role identity and judgment latitude, written to be
   prepended verbatim into the brief.
2. **Brief preparation** — what Claude must nail down *before* delegating.
3. **Delegation call** — the `pi -p @<file>` invocation and exit-status handling.

### Shared delegation mechanics (single source of truth)

`subagent-driven-development/implementer-prompt.md` holds the canonical invocation
mechanics and the exit-status rule. The other three prompt files cross-reference it
rather than restating it — preserving the existing single-source-of-truth structure.

**Invocation.** Every delegation uses `pi -p @<brief-file>` (print mode): one blocking
call per delegation, runs to completion, exits. Each call with no `--continue` or
`--resume` starts a fresh session, so every delegation gets clean, intentional context.

**Result capture.** pi's stdout is the delegate's summary text. The process exit code
signals outcome:

- **exit 0** → delegation completed; the controller proceeds to review the artifact.
- **exit non-zero** → failed. The controller reads stdout/stderr for the error, retries
  once if the failure looks transient, decomposes and re-delegates if the task was
  too big, and otherwise escalates to the user — pointing at the brief file under
  `.pi-delegations/` and the latest pi session under `~/.pi/agent/sessions/`.

This collapses the old `complete`/`error`/`max_steps`/`timeout`/`token_limit`
stop-reason taxonomy into exit-0-vs-non-zero. pi print mode runs to completion; there
is no step budget, so an oversized task surfaces as a plain failure rather than a
distinct budget stop-reason. The old `files_changed` and `commands_run` report fields
are dropped: the controller already verifies results by inspecting the repo (`git
status`, `git diff`) and running tests, which is more reliable than a self-reported
field.

**Brief delivery.** Briefs are delivered via `pi -p @<file>` uniformly — verified to
work — so that large multi-line briefs never hit shell-quoting problems and every
brief is inspectable on failure. The `.pi-delegations/` directory replaces the
obsolete `.llama-delegations/` directory in `.gitignore`.

**Obsolete parameters.** The MCP-era `working_dir` parameter is gone: pi uses the
current working directory, which the controller already controls. The MCP-era
`context_hints` parameter is gone: the brief names relevant file paths in prose and pi
reads them with its own `read` tool — one mechanism.

## Per-Skill Integration

### brainstorming → feature designer

The approved design lives in the brainstorming conversation, which pi cannot read, so
the brief must still carry the full design substance. Only the invocation and the
format handling change — pi can read an existing spec for format rather than being
handed format boilerplate.

After the design is approved with the user, Claude composes the brief — every design
section, every resolved decision and tradeoff, the exact spec file path — and delegates
the *writing* of the spec file to pi via `feature-designer-prompt.md`.

- The spec self-review step is unchanged in intent but now reviews pi's draft. If
  issues are found, Claude re-delegates a focused fix or fixes inline.
- The user reviews the written spec. Unchanged.
- The terminal state is still invoking `writing-plans`.

Edit surface: `brainstorming/SKILL.md` step 6 wording + a pointer to the new prompt
file. The tuned behavior-shaping prose (Red Flags, anti-patterns, process diagram) is
not touched.

### writing-plans → planner

Claude still does the judgment-heavy scope check itself. pi then produces the whole
implementation plan, including the decomposition. The controller no longer produces
the File Structure map or the task-list outline itself.

- The controller's **Self-Review** of the produced plan against the spec becomes the
  key quality gate — it now also validates pi's decomposition. Plan-structure
  expectations (bite-sized steps, real and complete code blocks, no placeholders) move
  into the planner persona preamble so pi follows them.
- The **Execution Handoff** section is unchanged.

Edit surface: `writing-plans/SKILL.md` — a delegation step between the scope check and
self-review, plus a pointer to the new prompt file.

### subagent-driven-development → implementer

Per task, the controller reads the plan only to extract the task list (for TodoWrite
tracking and to dispatch one task at a time). It then issues one delegation per task —
e.g. "Implement Task N from `<plan path>` TDD-style; commit when green" — and pi reads
the plan and spec itself.

The existing two-stage Claude review (spec compliance, then code quality, each a Claude
reviewer subagent) runs unchanged on the result.

The per-task process diagram loses the test-author node and the "tests fail for the
right reason?" checkpoint node.

The **Right-Sizing Before Delegating** section shrinks to a light "one task per
delegation; split only a genuinely huge task."

The **Handling pi exit status** section replaces the old "Handling Llama stop_reason"
section (see Shared delegation mechanics above).

Red Flags that previously forbade letting the executor read the plan file are inverted
— pi reading the plan is now the intended path.

The Model Selection section, Example Workflow, and the references to the MCP tool are
rewritten.

The Reviewer Output Discipline and Focused Re-Review sections (about Claude reviewer
subagents) are unchanged in substance.

Edit surface: `subagent-driven-development/SKILL.md` and `implementer-prompt.md`. The
`test-author-prompt.md` file is deleted.

### finishing-a-development-branch → documenter

The **Documentation Sweep** step already exists in The Process (it currently delegates
to the Llama documenter). It is updated to delegate via pi.

The brief becomes a short feature summary plus an instruction to update user-facing
documentation and the changelog. pi runs `git diff` itself and finds the files to edit.

- Claude reviews the documentation diff, then commits it on the feature branch before
  merge/PR.
- If no docs need updating, the step is a no-op and the skill proceeds.

Edit surface: `finishing-a-development-branch/SKILL.md` — the existing Documentation
Sweep step is updated to delegate via pi. The merge/PR/cleanup logic is untouched.

## Review of Produced Artifacts

- **Code artifacts** (implementer): reviewed by the existing two-stage Claude reviewer
  subagents in `subagent-driven-development`.
- **Prose artifacts** (feature designer, planner, documenter): reviewed by Claude
  directly, using the self-review checklists that already exist in those skills. No
  reviewer subagent is dispatched for prose — consistent with how brainstorming's spec
  self-review and writing-plans' self-review are already defined.

## Exit Status Handling

All four personas cross-reference the exit-status rule in
`subagent-driven-development/implementer-prompt.md`:

| Exit code | Action |
|---|---|
| 0 | Proceed to Claude review of the artifact |
| non-zero | Assess failure from stdout/stderr; retry once if transient; decompose if oversized; escalate to user if neither applies (pointing at the brief under `.pi-delegations/` and the pi session under `~/.pi/agent/sessions/`) |

For prose personas, a budget-hit failure means delegating the document section-by-section
rather than escalating immediately.

## Testing

No automated tests for skill files. Validation is manual:

- Run `brainstorming` to the design-doc step and confirm a single `pi -p` call produces
  the spec file.
- Run `writing-plans` and confirm pi produces the full plan (including decomposition)
  from the spec.
- Run `subagent-driven-development` and confirm one delegation per task (implementer
  only, no separate test author), with the existing two-stage Claude review on each
  result.
- Run `finishing-a-development-branch` and confirm the documentation sweep delegates to
  pi and commits the result before the options menu.
- Confirm exit-0 proceeds to review and a simulated failure escalates correctly.
- Grep the skills tree to confirm zero remaining `delegate_to_llama` or `llama-mcp`
  references.

## Risks

- **Reworking tuned, behavior-shaping skill content.** Red Flags are inverted, process
  diagrams change, and a persona is deleted. This is acceptable only because the change
  is an explicitly fork-local customization, not an upstream contribution. Edits to
  surrounding tuned prose (anti-pattern sections, unrelated Red Flags) are kept minimal.
- **pi cannot ask questions in print mode.** If the controller under-specifies a brief,
  pi will make a judgment call that may be wrong. Mitigation: the controller resolves
  genuinely blocking ambiguity up front, and the review step catches bad calls.
- **Extra capability means pi may do more than asked** (over-building). Mitigation: the
  spec-compliance review stage already exists to catch over- and under-building.

## Files Affected

**Edited (9):**

| File | Change |
|---|---|
| `skills/subagent-driven-development/SKILL.md` | Collapse test author + implementer; update process diagram; invert Red Flags; rewrite stop_reason → exit status; rewrite model selection and example workflow |
| `skills/subagent-driven-development/implementer-prompt.md` | Canonical `pi -p @<file>` invocation; exit-status rule; cross-reference from other prompt files |
| `skills/brainstorming/SKILL.md` | Update step 6 to delegate to pi |
| `skills/brainstorming/feature-designer-prompt.md` | Rename from "Llama"; update invocation to `pi -p @<file>`; cross-reference exit status |
| `skills/writing-plans/SKILL.md` | Replace controller decomposition with pi delegation; update self-review to validate decomposition |
| `skills/writing-plans/planner-prompt.md` | Rename from "Llama"; update invocation; move plan-structure expectations into preamble |
| `skills/finishing-a-development-branch/SKILL.md` | Update the existing documentation sweep step to delegate to pi |
| `skills/finishing-a-development-branch/documenter-prompt.md` | Rename from "Llama"; update invocation; pi runs `git diff` itself |
| `.gitignore` | `.llama-delegations/` → `.pi-delegations/` |

**Deleted (1):**

| File | Reason |
|---|---|
| `skills/subagent-driven-development/test-author-prompt.md` | Test author persona merged into implementer |

## Resolved Decisions and Tradeoffs

- **Print mode chosen over RPC mode and SDK mode.** RPC mode is a persistent, stateful,
  streaming conversational session whose value (cross-turn persistence, steering,
  follow-ups) is wasted when every delegation deliberately wants fresh context. It would
  also force the skill to manage a long-lived background process over bidirectional
  JSONL stdio and answer permission prompts — fragile, and a poor fit for Claude Code's
  request/response shell tools.
- **Plain text + exit code chosen over `--mode json`.** `--mode json` emits a very
  chatty JSONL event stream (one line per token delta) that the controller would have to
  parse, with no benefit over the exit code for success/failure detection.
- **"Llama" renamed to "pi" in prose.** "Delegate to pi", "pi implementer", prompt
  titles become "pi <persona> Delegation Template", and the shared "Handling Llama
  stop_reason" section becomes "Handling pi exit status". A one-line note records that
  pi runs the user's locally configured model, so the rename does not read as a model
  change. Persona role definitions (the "You are a …" preambles) are not renamed — they
  describe roles, not the harness.
- **Test author and implementer merged into one implementer delegation.** The mid-task
  test-fails-correctly checkpoint is dropped, chosen for simplicity and justified by
  pi's stronger reasoning and the existing two-stage review safety net.
- **pi owns plan decomposition.** The controller's spec-vs-plan Self-Review is the gate,
  chosen over the controller pre-decomposing, justified by pi's reasoning ability.
- **Briefs delivered uniformly via `pi -p @<file>` from `.pi-delegations/`.** Chosen for
  robustness against shell quoting and for inspectability on failure.

## Open Questions for Claude

None. All resolved decisions and tradeoffs are captured above.
