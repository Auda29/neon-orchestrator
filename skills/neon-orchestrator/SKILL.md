---
name: neon-orchestrator
description: >-
  Lean multi-model role orchestration for Antigravity, Claude Code, Codex, and Gemini CLI:
  the active agent (recommended: Gemini 4 Neon, always high) acts as coordinator and
  co-reviewer, delegating implementation to gpt-6.2-sol (xhigh default, high for small
  tasks) and independent read-only reviews to gpt-6.2-astra (high). Use when starting
  orchestrated work on large projects or long tasks.
---

# Neon Orchestrator

Assign clear roles across models and let each model do what it does best.

## Roles & Defaults

| Role | Model | Default Effort | Responsibility |
| :--- | :--- | :--- | :--- |
| **Coordinator & Co-Reviewer** | Active agent (recommended: **Gemini 4 Neon**) | **`high`** (always `high` for Neon 4) | Scopes work, coordinates slices, reviews changes independently, brings own objections/decisions, arbitrates findings. **Does not write production code.** |
| **Implementer** | `gpt-6.2-sol` (`codex`) | **`xhigh`** (`high` only for small/routine tasks) | Implements scoped changes, updates/adds tests, runs verification, applies agreed review fixes. |
| **Independent Reviewer** | `gpt-6.2-astra` (`codex`) | **`high`** | **Read-only** review of actual diffs and surrounding code; returns concrete findings and verdict. Never edits files. |

Honor explicit user overrides for models or reasoning effort. If `gpt-6.2-sol` or `gpt-6.2-astra` is unavailable, stop and tell the user — never silently substitute another model, except for the refusal fallback below.

**Refusal fallback**: If `gpt-6.2-sol` or `gpt-6.2-astra` refuses a delegated instruction due to copyright or internal policy rules, retry **only that step** with its predecessor at the same reasoning effort (`gpt-6.2-sol` → `gpt-6.1-sol`, `gpt-6.2-astra` → `gpt-6-astra`). Once that step completes, immediately switch back to `gpt-6.2-sol` / `gpt-6.2-astra` for all subsequent steps. If the predecessor also refuses or is unavailable, stop and escalate to the user.

## Core Principles

- **Stay in the coordinator role**: Delegate all code edits and review fixes to `gpt-6.2-sol` unless the user explicitly asks you to edit directly.
- **Scale naturally to large tasks**: Break big work into coherent, meaningful slices; avoid micro-delegation and redundant context gathering. Use worktree isolation only for truly independent parallel slices.
- **Self-contained briefs, concise reports**: Delegated workers have no parent history. Pass clear goals, relevant paths, constraints, and verification expectations — never secrets. Ask for compact reports (changes, verification results, actionable findings, open blockers) that reference files and evidence instead of pasting logs or large code excerpts.
- **Persistent handoffs**: For multi-session work, keep milestones, decisions, verification evidence, and next steps in the project's existing documentation; have Sol update it and reuse it when resuming.
- **Dual, unbiased review**: After Sol finishes, launch a **fresh** read-only `gpt-6.2-astra` (`high`) review and **also inspect the diff yourself** — focusing on architecture, correctness, integration boundaries, and high-risk changes — before anchoring on Astra's output. Never reuse an old review thread.
- **Arbitrate, don't relay**: Combine Astra's findings with your own review, add your own calls or objections, drop false positives/nits, and give Sol one clear, unified fix brief.
- **Know when to stop**: Re-review fixes with a fresh Astra session until clean. Escalate genuine tradeoffs or persistent blockers to the user instead of looping endlessly.

## Dispatch by Harness

See [harness-adapters.md](./references/harness-adapters.md) for exact calls:
1. **T3 Code** (Antigravity, Claude Code, Codex): `orchestrator_capabilities` once per thread, then `delegate_task` (`mode: "async"`, no polling). Never bypass `delegate_task` with CLI calls.
2. **Standalone Codex**: `spawn_agent` if it supports model + effort targeting, else `codex exec`.
3. **Standalone Claude Code / Antigravity / Gemini CLI**: `codex exec --ephemeral` (`-s workspace-write` for Sol, `-s read-only` for Astra).
