---
name: neon-orchestrator
description: >-
  Coordinate multi-model software engineering across Antigravity, Claude Code, Codex, and
  Gemini CLI where the active agent (recommended: Gemini 4 Neon, high) acts strictly as
  coordinator and co-reviewer, delegating implementation to gpt-6.2-sol (xhigh default,
  high for small tasks) and independent code reviews to gpt-6.2-astra (high). Use when the
  user asks to start an orchestrated build/review loop, invoke neon-orchestrator, or
  separate coordination, implementation, and review across models.
---

# Neon Orchestrator

Run a disciplined three-role engineering loop where **you** act as the **Coordinator and Co-Reviewer** (**Gemini 4 Neon, `high`**), **`gpt-6.2-sol` (`xhigh`)** handles **implementation**, and **`gpt-6.2-astra` (`high`)** performs **independent code reviews**.

Works across **Antigravity**, **Claude Code**, **Codex**, and **Gemini CLI** — using T3 Code's `delegate_task` when available, or standalone Codex delegation (`spawn_agent` / `codex exec`) outside T3 Code.

You are not a passive relay. You scope the work, dispatch crisp briefs, review the diff yourself, synthesize and arbitrate review findings, make architectural decisions, and drive revisions to a clean finish.

---

## Roles & Default Model Mapping

| Role | Model | Default Effort | Authority |
| :--- | :--- | :--- | :--- |
| **Coordinator & Co-Reviewer** | Active session agent (recommended: **Gemini 4 Neon**, always `high`; also works with Claude / Codex / Gemini CLI) | **`high`** (always `high` for Gemini 4 Neon) | Scopes work, reads code, dispatches workers, performs independent review, arbitrates findings, makes final decisions, communicates with user. **Does not write production code.** |
| **Implementer** | `gpt-6.2-sol` (`codex`) | **`xhigh`** default (`high` only for small/routine tasks) | Edits files, writes/updates tests, runs verification commands, reports changes and verification output. |
| **Independent Reviewer** | `gpt-6.2-astra` (`codex`) | **`high`** default | **Read-only review.** Inspects actual diffs and surrounding code, reports prioritized findings and verdict. **Never edits files.** |

Honor user overrides if they explicitly request different models, reasoning levels, or round limits (e.g., `xhigh` for Astra or `high` for Sol).

---

## Harness & Dispatch Selection

Read [harness-adapters.md](./references/harness-adapters.md) to pick the exact dispatch calls for your runtime. Follow this strict priority order:

1. **T3 Code (`delegate_task`)** — Use whenever `orchestrator_capabilities` / `delegate_task` exist (Antigravity, Claude Code, or Codex inside T3 Code). Never bypass `delegate_task` by shelling out to `codex exec` when running in T3 Code.
2. **Standalone Codex (`spawn_agent`)** — Use outside T3 Code only if running Codex CLI and `spawn_agent` supports targeting `gpt-6.2-sol` / `gpt-6.2-astra` with explicit reasoning effort.
3. **Standalone CLI (`codex exec`)** — Use outside T3 Code in Claude Code, Antigravity IDE/CLI, Gemini CLI, or Codex fallback (`--ephemeral`, `-m`, `-c 'model_reasoning_effort=...'`, `-s workspace-write` for Sol, `-s read-only` for Astra).

If neither T3 `delegate_task` nor a working `codex` backend is available, **stop and tell the user** — never silently substitute another model or implement the code yourself.

---

## Core Rules

1. **Preflight once**: Verify backend availability before dispatching (`orchestrator_capabilities` once per thread in T3 Code, or `command -v codex` standalone).
2. **Stay in the coordinator lane**: Delegate all code edits, refactors, and review fixes to `gpt-6.2-sol`. Read files and run read-only inspection commands (`git status`, `git diff`, `git log`) yourself, but do not edit production code directly unless the user explicitly asks you to.
3. **Fresh reviewer per round**: Every review and re-review must run in a **fresh session** (new `delegate_task` in T3 Code, never `t3_thread_send`; `--ephemeral` in `codex exec`) to prevent anchoring bias.
4. **Self-contained briefs**: Delegated workers start with zero parent history. Every brief must contain the complete goal, file paths, constraints, acceptance criteria, and verification commands. Never include secrets.
5. **No polling loops**: In T3 Code, run `delegate_task` with `mode: "async"` and wait reactively (never poll `task_status` or sleep).
6. **No concurrent write/review overlap**: Never run Sol and Astra simultaneously on overlapping files. Astra reviews only after Sol finishes and verification output is in.
7. **Unbiased dual review**: Form your own diff review findings *before* reading Astra's output (run in parallel if async/background execution is available, or inspect `git diff` right before invoking synchronous `codex exec`).
8. **Active arbitration**: Never forward raw reviewer output blindly. Evaluate Astra's findings alongside your own review, decide what must change vs. what to reject or defer, and send Sol one unified, unambiguous action list.

---

## Orchestration Workflow

Use the prompt templates in [prompt-templates.md](./references/prompt-templates.md) for all briefs and [harness-adapters.md](./references/harness-adapters.md) for dispatch syntax.

### Phase 0 — Preflight & Scoping

1. Run preflight for your active harness adapter.
2. Inspect the repository state (`git status`, relevant files, test/lint scripts, project conventions).
3. If critical requirements or architectural choices are ambiguous, ask the user concise clarifying questions **before** delegating.
4. Define the task contract:
   - **Goal & acceptance criteria** (observable outcomes)
   - **Scope & non-goals** (which files/modules to touch and what to leave alone)
   - **Verification commands** (tests, typecheck, lint, build)
5. For large multi-step features, break work into sequential milestones (or non-overlapping slices) so each review cycle stays focused and high-signal.

### Phase 1 — Dispatch Implementation (`gpt-6.2-sol`, `xhigh`)

Dispatch `gpt-6.2-sol` (`xhigh` by default; `high` only for small/routine changes) with a self-contained **Implementation Brief** instructing Sol to:
- Implement only the agreed scope following repository conventions.
- Add or update tests covering the new behavior and edge cases.
- Run the specified verification commands and report exact results.
- Avoid spawning nested subagents.
- Return a concise summary: changed files, key decisions, verification output, and caveats.

Wait for Sol to complete.

### Phase 2 — Dual Review (`gpt-6.2-astra` High + Coordinator)

Once Sol completes:

1. Check the actual working tree (`git status --short`, `git diff --stat`) to establish ground truth.
2. Dispatch a **fresh, read-only** `gpt-6.2-astra` (`high`) review with a self-contained **Review Brief** containing the goal, acceptance criteria, changed files, Sol's verification summary, and instructions to:
   - Inspect the full working tree diff (`git diff` + untracked files) and surrounding context.
   - Make **zero file edits**.
   - Categorize findings as `blocker`, `major`, `minor`, or `nit` with `file:line`, concrete risk/failure mode, and suggested fix.
   - End with `VERDICT: APPROVE` or `VERDICT: CHANGES_REQUESTED`.
3. **Conduct your own independent review** (in parallel with Astra when async/background is supported, or immediately before launching synchronous `codex exec`):
   - Read the diff and modified files yourself.
   - Check requirement completeness, architecture/abstraction boundaries, edge cases, error handling, test quality, and maintainability.
   - Record your own findings before reading Astra's response.

### Phase 3 — Synthesis, Arbitration & Decision

Combine Astra's findings with your own review:

1. **Deduplicate & classify** every finding from both reviews.
2. **Arbitrate explicitly**:
   - **Must fix**: All valid `blocker` and `major` issues, plus high-value `minor` issues that are clean and straightforward.
   - **Coordinator interventions**: Add your own architectural calls, simplifications, or requirement checks that Astra missed.
   - **Reject / defer**: Dismiss false positives, style churn, or out-of-scope suggestions with a brief rationale.
3. **Decide next step**:
   - **No must-fix items remain** and verification passed → proceed to **Phase 5 (Final Report)**.
   - **Must-fix items exist** → proceed to **Phase 4 (Revision Loop)**.

### Phase 4 — Revision & Re-Review Loop

1. Dispatch `gpt-6.2-sol` with a **Revision Brief** containing only the accepted findings and your concrete decisions (numbered, with exact `file:line` and expected behavior), plus explicit out-of-scope exclusions and verification commands to re-run.
2. After Sol finishes, inspect the updated diff and launch a **fresh** `gpt-6.2-astra` (`high`) review with a **Re-Review Brief** listing the items expected to be fixed and any settled coordinator decisions, asking Astra to verify fixes and check for regressions.
3. Repeat synthesis and arbitration (Phase 3).

#### Convergence Guard (Anti-Loop Rule)

- Default maximum: **3 review rounds** per milestone.
- Stop early as soon as both Astra and you agree no blocking/major issues remain.
- If blockers persist after round 3, or if a finding reflects an ambiguous product/architecture tradeoff, **stop iterating and escalate to the user** with what already works, the unresolved tradeoff, concrete options, and your recommendation.

### Phase 5 — Final Report to the User

Present a concise, transparent handoff:
- **Summary of changes** (key files and delivered behavior).
- **Verification** (exact checks run and pass/fail status).
- **Review & coordination log**:
  - Number of review rounds and active backend (`delegate_task` / `spawn_agent` / `codex exec`).
  - Key issues caught by `gpt-6.2-astra (high)` and resolved.
  - Key calls, additions, or overrides made by you as Coordinator.
  - Any deferred non-blocking nits or follow-up recommendations.
