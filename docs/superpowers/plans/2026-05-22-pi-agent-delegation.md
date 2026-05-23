# Pi Agent Delegation Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `llama-mcp` MCP delegation transport with the `pi` CLI agent across all four delegation personas in the superpowers development workflow.

**Architecture:** Each delegation becomes a thin brief written to `.pi-delegations/<persona>-<timestamp>.md` and run with `pi -p @<brief-file>`. The controller shifts from context-curator to orchestrator/reviewer: pi reads spec/plan/diff files itself with its own tools. `subagent-driven-development/implementer-prompt.md` becomes the single source of truth for invocation mechanics and exit-status handling; the other three prompt files cross-reference it. The test-author and implementer personas collapse into one implementer delegation.

**Tech Stack:** Markdown skill files. No code, no automated tests — validation is grep checks and read-back.

**Source spec:** `docs/superpowers/specs/2026-05-22-pi-agent-delegation-design.md`

---

## Notes for the executor

- There is no test suite for skill files. Each task's verification is a `grep`/read-back check, not a test run.
- Task 2 must land before Tasks 4–6: `implementer-prompt.md` is the single source of truth those tasks cross-reference.
- Persona role preambles (`You are a …`) describe roles, not the harness — **do not rename them**. Only prose, titles, and invocation mechanics change from "Llama" to "pi".
- pi runs the user's locally configured model. The migration is a transport change, not a model change — never pass `--provider` or `--model`.
- Keep edits to surrounding tuned behavior-shaping prose minimal. Change what the spec calls for; leave unrelated Red Flags, anti-pattern sections, and reviewer-discipline sections alone except for the literal "Llama" → "pi" rename where it appears.

---

## Task 1: Rename the delegations directory in `.gitignore`

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Replace the ignored directory**

Change the last line of `.gitignore` from `.llama-delegations/` to `.pi-delegations/`.

- [ ] **Step 2: Verify**

Run: `grep -n "delegations" .gitignore`
Expected: one line, `.pi-delegations/`. No `.llama-delegations/`.

- [ ] **Step 3: Commit**

```bash
git add .gitignore
git commit -m "chore: gitignore .pi-delegations instead of .llama-delegations"
```

---

## Task 2: Rewrite `implementer-prompt.md` as the canonical pi template, and delete `test-author-prompt.md`

This task collapses the test-author and implementer personas into one implementer delegation and makes `implementer-prompt.md` the single source of truth for pi invocation mechanics and exit-status handling.

**Files:**
- Modify (full rewrite): `skills/subagent-driven-development/implementer-prompt.md`
- Delete: `skills/subagent-driven-development/test-author-prompt.md`

- [ ] **Step 1: Replace the entire contents of `implementer-prompt.md`**

```markdown
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
```

- [ ] **Step 2: Delete the test-author prompt file**

```bash
git rm skills/subagent-driven-development/test-author-prompt.md
```

- [ ] **Step 3: Verify**

Run: `grep -rn "delegate_to_llama\|llama-mcp\|stop_reason" skills/subagent-driven-development/implementer-prompt.md`
Expected: no output.

Run: `test -f skills/subagent-driven-development/test-author-prompt.md && echo EXISTS || echo GONE`
Expected: `GONE`.

- [ ] **Step 4: Commit**

```bash
git add skills/subagent-driven-development/implementer-prompt.md
git commit -m "feat: make implementer-prompt the canonical pi delegation template"
```

---

## Task 3: Update `subagent-driven-development/SKILL.md`

The heaviest edit. Apply each change below in place. Do not touch the Reviewer Output
Discipline or Focused Re-Review sections except for the literal "Llama" → "pi" rename.

**Files:**
- Modify: `skills/subagent-driven-development/SKILL.md`

- [ ] **Step 1: Continuous-execution paragraph (line ~14)**

Change "a Llama delegation failure you cannot resolve after retry or decomposition" to "a pi delegation failure you cannot resolve after retry or decomposition".

- [ ] **Step 2: Rewrite the per-task process diagram (the `digraph process` block)**

Remove the test-author nodes, the "Run tests — fail for the right reason?" checkpoint and its re-delegate node, and every `stop_reason` diamond. The per-task flow becomes:

```
Prepare implementer brief (resolve blocking ambiguity)
  -> Delegate implementer (pi -p @<brief>, ./implementer-prompt.md)
  -> pi exit status? : 0 -> Dispatch spec reviewer subagent
                       : non-zero -> Assess / retry / decompose / escalate
  -> Spec reviewer confirms code matches spec? : no -> Re-delegate fix to pi (spec) -> re-review
                                                : yes -> Dispatch code quality reviewer subagent
  -> Code quality reviewer approves? : no -> Re-delegate quality fix to pi -> re-review
                                      : yes -> Mark task complete in TodoWrite
```

Keep the outer flow (read plan / extract task list / TodoWrite → per-task loop → final code reviewer → finishing-a-development-branch) intact. Replace `mcp__llama-mcp__delegate_to_llama` node labels with `pi -p @<brief>`.

- [ ] **Step 3: Rewrite the "Model Selection" section**

Replace with: implementation is a single pi delegation per task (no separate test author — pi writes the failing test, implements, and makes it green within one delegation). pi runs the user's locally configured model. Review roles still use Claude subagents for spec compliance and code quality.

- [ ] **Step 4: Shrink "Right-Sizing Before Delegating"**

Replace the section body with a light version: one task per delegation; split only a genuinely huge task (one that bundles unrelated changes or whose "Done when" cannot be stated in a short paragraph). Drop the `context_hints` trimming advice — pi reads files itself.

- [ ] **Step 5: Replace "Handling Llama stop_reason" with "Handling pi exit status"**

Replace the whole section with a brief one that defers to the canonical table: "pi signals outcome via process exit code. Exit 0 → proceed to spec review. Non-zero → assess, retry once if transient, decompose if oversized, else escalate. See `./implementer-prompt.md` → 'Handling pi exit status' for the canonical rule." Keep the existing "Fix-loop context discipline" guidance, reworded Llama → pi (or note it now lives in `implementer-prompt.md`).

- [ ] **Step 6: Update the "Prompt Templates" list**

Remove the `./test-author-prompt.md` line. Reword the implementer line: `./implementer-prompt.md` — Delegate an implementation task to pi (canonical invocation + exit-status rule).

- [ ] **Step 7: Rewrite the "Example Workflow"**

Rewrite so each task is one pi delegation. Replace `delegate_to_llama(...)` calls with `pi -p @.pi-delegations/implementer-<timestamp>.md`, replace `stop_reason: complete` / `files_changed` output with an exit-0 result and a short summary line, and keep the two-stage spec-then-quality review flow.

- [ ] **Step 8: Update "Advantages"**

Remove claims that are no longer true: "No file reading overhead (controller provides full text)", "Controller curates exactly what context is needed", "Subagent gets complete information upfront". Reword the cost section: one pi delegation per task (not "Llama delegation + 2 reviewer subagents" with a separate test author). Keep the quality-gates bullets.

- [ ] **Step 9: Invert the Red Flags**

Remove these (they only existed because the MCP executor was weak):
- "Make Llama read the plan file (provide full text in the `task` string instead)" — **inverted**: pi reading the plan is the intended path.
- "Attach large files via `context_hints` when only a region is needed"
- "Delegate to Llama without running the context preparation step first"
- "Skip scene-setting context (Llama needs to understand where the task fits)"

Reword the rest Llama → pi: "Leave genuine ambiguity unresolved before delegating (pi cannot ask questions)", "If pi delegation fails (non-zero exit)", etc. Keep all review-discipline Red Flags. Delete the "If context prep reveals ambiguity" subsection's `task` string / `context_hints` references; keep the "resolve, ask one question at a time" guidance.

- [ ] **Step 10: Verify**

Run: `grep -rn "llama\|Llama\|stop_reason\|delegate_to_llama\|context_hints\|test-author" skills/subagent-driven-development/SKILL.md`
Expected: no output.

- [ ] **Step 11: Commit**

```bash
git add skills/subagent-driven-development/SKILL.md
git commit -m "feat: migrate subagent-driven-development to pi delegation"
```

---

## Task 4: Update the brainstorming feature-designer persona

**Files:**
- Modify: `skills/brainstorming/feature-designer-prompt.md`
- Modify: `skills/brainstorming/SKILL.md`

- [ ] **Step 1: Update `feature-designer-prompt.md`**

- Title: `# Llama Feature Designer Delegation Template` → `# pi Feature Designer Delegation Template`.
- Intro: replace "Delegate via the `mcp__llama-mcp__delegate_to_llama` MCP tool." with a sentence pointing at the canonical invocation: "Delegate via `pi -p @<brief-file>`; see `subagent-driven-development/implementer-prompt.md` for the canonical invocation mechanics and exit-status rule."
- **Keep the persona preamble verbatim** ("You are a feature designer …").
- Brief Preparation: unchanged in substance — the approved design lives in the conversation pi cannot read, so the brief still carries the full design substance. Note pi can read an existing spec under `docs/superpowers/specs/` for format rather than being handed format boilerplate.
- Replace the `mcp__llama-mcp__delegate_to_llama:` YAML call block with: write the brief to `.pi-delegations/feature-designer-<timestamp>.md`, then run `pi -p @.pi-delegations/feature-designer-<timestamp>.md` from the project root. Drop `working_dir` and `context_hints`.
- "After Delegation": replace the response-field inspection with the exit-status rule — cross-reference `subagent-driven-development/implementer-prompt.md` → "Handling pi exit status". A budget/oversize failure means re-delegating the spec section-by-section rather than escalating immediately. Then run brainstorming step 7 (spec self-review) on pi's draft.

- [ ] **Step 2: Update `brainstorming/SKILL.md`**

- Checklist step 6 (line ~29): "delegate the writing of the spec to the Llama feature designer" → "delegate the writing of the spec to the pi feature designer".
- "After the Design" → "Documentation" bullets (line ~111–114): reword "Llama feature designer" → "pi feature designer" and "Review Llama's draft" → "Review pi's draft".

- [ ] **Step 3: Verify**

Run: `grep -rn "llama\|Llama" skills/brainstorming/SKILL.md skills/brainstorming/feature-designer-prompt.md`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add skills/brainstorming/SKILL.md skills/brainstorming/feature-designer-prompt.md
git commit -m "feat: migrate brainstorming feature designer to pi delegation"
```

---

## Task 5: Update the writing-plans planner persona

The planner now owns the full plan including decomposition: pi produces the File
Structure map and the task-list outline as well as the plan body. The controller
keeps the judgment-heavy Scope Check and the Self-Review gate.

**Files:**
- Modify: `skills/writing-plans/planner-prompt.md`
- Modify: `skills/writing-plans/SKILL.md`

- [ ] **Step 1: Update `planner-prompt.md`**

- Title: `# Llama Planner Delegation Template` → `# pi Planner Delegation Template`.
- Intro: the planner now produces the whole plan including the File Structure map and the decomposition; only the Scope Check stays with Claude. Replace the MCP-tool sentence with the canonical-invocation cross-reference (as in Task 4 Step 1).
- Persona preamble: **keep "You are an implementation planner"** but update its body — pi now decides the decomposition. Move the plan-structure expectations into the preamble: bite-sized steps (failing test → run → minimal implementation with real code → verify → commit), real and complete code blocks, no placeholders ("TBD", "add error handling", "similar to Task N"), exact file paths, exact commands with expected output.
- Brief Preparation: Claude pastes the Scope Check result and points pi at the spec path; pi reads the spec itself and produces the File Structure map and task-list outline. Drop the requirement that Claude pre-produces the File Structure map and outline.
- Replace the `mcp__llama-mcp__delegate_to_llama:` block with: write the brief to `.pi-delegations/planner-<timestamp>.md`, run `pi -p @.pi-delegations/planner-<timestamp>.md`. Drop `working_dir`/`context_hints`.
- "After Delegation": cross-reference the canonical exit-status rule; budget-hit → re-delegate section-by-section. Then run the writing-plans Self-Review on pi's plan.

- [ ] **Step 2: Update `writing-plans/SKILL.md`**

- Replace the "Delegate Plan-Body Expansion" section with a "Delegate Plan Creation" section: after the Scope Check, delegate the whole plan — File Structure map, decomposition, and bite-sized task bodies — to the pi planner; see `planner-prompt.md`. The controller no longer produces the File Structure map or the task-list outline itself.
- The "File Structure", "Bite-Sized Task Granularity", and "Task-Level Right-Sizing" sections describe what the planner must produce — keep them as reference for the planner persona, but frame the workflow so pi performs the decomposition and the controller validates it.
- "Self-Review": add a check that validates pi's decomposition (task boundaries, ordering, right-sizing) against the spec, in addition to the existing spec-coverage / placeholder / type-consistency / implementer-fit checks.
- Leave the Plan Document Header, Task Structure, No Placeholders, and Execution Handoff sections intact.

- [ ] **Step 3: Verify**

Run: `grep -rn "llama\|Llama" skills/writing-plans/SKILL.md skills/writing-plans/planner-prompt.md`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add skills/writing-plans/SKILL.md skills/writing-plans/planner-prompt.md
git commit -m "feat: migrate writing-plans planner to pi delegation"
```

---

## Task 6: Update the finishing-a-development-branch documenter persona

**Files:**
- Modify: `skills/finishing-a-development-branch/documenter-prompt.md`
- Modify: `skills/finishing-a-development-branch/SKILL.md`

- [ ] **Step 1: Update `documenter-prompt.md`**

- Title: `# Llama Documenter Delegation Template` → `# pi Documenter Delegation Template`.
- Intro: replace the MCP-tool sentence with the canonical-invocation cross-reference.
- **Keep the persona preamble verbatim** ("You are a documentation scribe …").
- Brief Preparation: the brief becomes a short feature summary plus an instruction to update user-facing documentation and the changelog. pi runs `git diff` itself to find the files. Drop the requirement that Claude reads the diff and enumerates every doc file — pi does that.
- Replace the `mcp__llama-mcp__delegate_to_llama:` block with: write the brief to `.pi-delegations/documenter-<timestamp>.md`, run `pi -p @.pi-delegations/documenter-<timestamp>.md`. Drop `working_dir`/`context_hints`.
- "After Delegation": cross-reference the canonical exit-status rule; budget-hit → re-delegate remaining doc files. Then review the documentation diff and commit it on the feature branch.

- [ ] **Step 2: Update `finishing-a-development-branch/SKILL.md` Step 2 (Documentation Sweep)**

- Item 1 currently has Claude run `git diff` and read it — change so the brief carries only a short feature summary and pi runs `git diff` itself.
- Item 3: "delegate the writing to the Llama documenter" → "delegate the writing to the pi documenter".
- Keep item 4 (review diff, commit on feature branch) and the no-op-if-nothing-to-update behavior.
- Leave Steps 1 and 3–7 (test verification, environment detection, options, execution, cleanup) untouched.

- [ ] **Step 3: Verify**

Run: `grep -rn "llama\|Llama" skills/finishing-a-development-branch/SKILL.md skills/finishing-a-development-branch/documenter-prompt.md`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add skills/finishing-a-development-branch/SKILL.md skills/finishing-a-development-branch/documenter-prompt.md
git commit -m "feat: migrate finishing-a-development-branch documenter to pi delegation"
```

---

## Task 7: Whole-tree verification

**Files:** none modified — verification only.

- [ ] **Step 1: Confirm zero MCP delegation references remain**

Run: `grep -rn "delegate_to_llama\|llama-mcp\|llama-delegations\|stop_reason" skills/`
Expected: no output.

- [ ] **Step 2: Confirm the deleted persona file is gone and not referenced**

Run: `grep -rn "test-author" skills/`
Expected: no output.

Run: `test -f skills/subagent-driven-development/test-author-prompt.md && echo EXISTS || echo GONE`
Expected: `GONE`.

- [ ] **Step 3: Confirm no stray "Llama" prose remains in the four touched skills**

Run: `grep -rni "llama" skills/brainstorming skills/writing-plans skills/subagent-driven-development skills/finishing-a-development-branch`
Expected: no output.

- [ ] **Step 4: Confirm `pi -p` invocation is present in all four prompt files**

Run: `grep -rln "pi -p @" skills/`
Expected: the four prompt files (`implementer-prompt.md`, `feature-designer-prompt.md`, `planner-prompt.md`, `documenter-prompt.md`).

- [ ] **Step 5: Read-back review**

Read each of the 8 edited skill files end-to-end. Confirm: persona preambles unchanged, cross-references to `implementer-prompt.md` resolve, no dangling reference to `working_dir`/`context_hints`/`files_changed`/`commands_run`/`transcript_path`, no broken process-diagram edges.

- [ ] **Step 6: Commit any read-back fixes**

If Step 5 surfaces fixes, apply and commit them:

```bash
git add <fixed files>
git commit -m "fix: resolve pi-delegation read-back issues"
```

If no fixes are needed, this step is a no-op.

---

## Manual validation (after all tasks)

These are from the spec's Testing section. They require running the skills and are
the human partner's call to perform — not blocking for plan completion:

- Run `brainstorming` to the design-doc step; confirm one `pi -p` call produces the spec file.
- Run `writing-plans`; confirm pi produces the full plan including decomposition.
- Run `subagent-driven-development`; confirm one delegation per task with the two-stage Claude review.
- Run `finishing-a-development-branch`; confirm the documentation sweep delegates to pi and commits before the options menu.
- Confirm exit-0 proceeds to review and a simulated failure escalates correctly.
