# Prompt Templates for `neon-orchestrator`

Adapt these templates when dispatching `gpt-6.2-sol` or `gpt-6.2-astra` via your active harness adapter (`delegate_task`, `spawn_agent`, or `codex exec`). Keep every brief self-contained — delegated workers start with no parent conversation history.

---

## 1. Implementation Brief (`gpt-6.2-sol`, `xhigh` default)

```markdown
You are the Implementer for this task. Do not spawn subagents.

## Goal
<Clear statement of what needs to be built or changed and why.>

## Acceptance Criteria
- <Observable criterion 1>
- <Observable criterion 2>

## Scope & Constraints
- Focus on: <paths / modules>
- Out of scope (do not modify): <paths / unrelated refactors>
- Conventions / constraints: <architecture notes, compatibility rules, libraries to use or avoid>

## Verification
Run and report exact outcomes for:
- `<test command>`
- `<typecheck / lint / build command>`

## Required Output
Return a concise report containing:
1. Summary of changes and key implementation decisions.
2. List of created/modified/deleted files.
3. Verification commands run and their results.
4. Any assumptions, edge cases, or follow-ups worth reviewer attention.
```

---

## 2. Independent Review Brief (`gpt-6.2-astra`, `reasoningEffort: "high"`)

```markdown
You are the Independent Code Reviewer for this change.
STRICT RULE: Review only. Do NOT modify, create, or delete any files, and do not spawn subagents.

## Context & Goal
<Original goal and acceptance criteria.>

## What to Inspect
- Changed/added files: <list of paths from git status>
- Inspect the full working tree diff (`git diff` plus any untracked files) and enough surrounding code to verify correctness, contracts, and edge cases.
- Implementer verification summary: <paste concise verification status from Sol>

## Review Focus
Check for:
1. Correctness, logic bugs, race conditions, and unhandled edge cases.
2. Fulfillment of all acceptance criteria and unintended regressions.
3. Security, data integrity, error handling, and resource cleanup.
4. Architecture fit, maintainability, and test quality/coverage.

## Required Output
Group findings strictly by severity:
- **Blocker**: Must fix before shipping (bugs, broken contracts, security issues, missing core requirements).
- **Major**: Should fix (significant edge cases, fragile design, meaningful missing tests).
- **Minor**: Nice to improve if straightforward.
- **Nit**: Optional polish (non-blocking).

For each finding, include:
- `file:line`
- Concrete issue / failure scenario
- Recommended fix

End with a single explicit verdict line:
- `VERDICT: APPROVE` (no blockers or majors)
- `VERDICT: CHANGES_REQUESTED`
```

---

## 3. Revision Brief (`gpt-6.2-sol`, `xhigh` default; `high` for small fixes)

```markdown
You are implementing the agreed fixes from review round <N>. Do not spawn subagents.

## Coordinator Decisions & Required Fixes
Address the following accepted findings in priority order:

1. [<Severity>] `<file:line>` — <Exact problem>
   - Required fix / coordinator decision: <Unambiguous direction>
2. [<Severity>] `<file:line>` — <Exact problem>
   - Required fix / coordinator decision: <Unambiguous direction>

## Explicitly Out of Scope (Do Not Change)
- <Any rejected/deferred reviewer nits so Sol doesn't over-edit>

## Verification
Re-run after applying fixes and report results:
- `<test / typecheck / lint commands>`

## Required Output
1. Item-by-item confirmation of how each numbered fix was addressed.
2. Updated list of touched files.
3. Fresh verification results.
```

---

## 4. Re-Review Brief (`gpt-6.2-astra`, `reasoningEffort: "high"`)

```markdown
You are conducting Re-Review Round <N> on the updated working tree.
STRICT RULE: Review only. Do NOT modify any files, and do not spawn subagents.

## Context & Goal
<Brief restatement of original goal and acceptance criteria.>

## Items Expected to Be Fixed Since Last Round
1. `<file:line>` — <Prior finding and expected fix>
2. `<file:line>` — <Prior finding and expected fix>

## Coordinator Decisions (Already Settled — Do Not Re-Flag Unless Broken)
- <Brief note on intentional tradeoffs or explicitly deferred items>

## What to Inspect
- Inspect the current working tree diff (`git diff` and untracked files) around the modified areas.
- Latest verification results reported by Implementer: <concise status>

## Required Output
1. Status of each expected fix (`FIXED` / `PARTIAL` / `UNRESOLVED`).
2. Any newly introduced regressions or remaining blockers/majors (same severity format with `file:line`).
3. Final verdict line: `VERDICT: APPROVE` or `VERDICT: CHANGES_REQUESTED`.
```
