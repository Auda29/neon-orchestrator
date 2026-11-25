# Harness & Dispatch Adapters

Select the **first** available dispatch backend in priority order. Never silently substitute different models if the required backend or models (`gpt-6.2-sol`, `gpt-6.2-astra`) are unavailable — stop and inform the user.

---

## 1. T3 Code (`delegate_task`) — Primary (Antigravity, Claude Code, Codex inside T3 Code)

Use this whenever `orchestrator_capabilities` and `delegate_task` are available. **Do not** shell out to `codex exec` or fall back to native subagents when running in T3 Code.

1. Call `orchestrator_capabilities` once per thread to verify `codex` is enabled and inspect exact option IDs.
2. Dispatch **Implementer (`gpt-6.2-sol`, `xhigh` default)**:
   - `role`: `"implementation"`
   - `mode`: `"async"`
   - `isolation`: `"shared"` (or `"worktree"` if explicitly requested / parallel independent streams)
   - `target`:
     ```json
     {
       "providerInstanceId": "codex",
       "model": "gpt-6.2-sol",
       "options": { "reasoningEffort": "xhigh" }
     }
     ```
     *(Use `"high"` only for small/routine changes or when explicitly requested.)*
3. Dispatch **Independent Reviewer (`gpt-6.2-astra`, `high` default)** — always a **new** `delegate_task` per review round (never `t3_thread_send`):
   - `role`: `"review"`
   - `mode`: `"async"`
   - `isolation`: `"shared"`
   - `target`:
     ```json
     {
       "providerInstanceId": "codex",
       "model": "gpt-6.2-astra",
       "options": { "reasoningEffort": "high" }
     }
     ```
4. Wait reactively for completion — never poll `task_status` or sleep. Use `task_cancel` to stop unneeded delegates.

*ACP fallback inside T3 Code*: If `T3_ACP_MCP_NODE` is present in the environment while MCP tools are not directly exposed, invoke `orchestrator_capabilities` and `delegate_task` via `acp-mcp-call` per T3 Code's ACP instructions.

---

## 2. Standalone Codex (`spawn_agent` or `codex exec`)

When running **Codex** outside T3 Code (`delegate_task` absent):

- **If native `spawn_agent` supports targeting `gpt-6.2-sol` and `gpt-6.2-astra` with `reasoning_effort`**:
  - Spawn a fresh agent for implementation (`gpt-6.2-sol`, `reasoning_effort: "xhigh"` default; `"high"` only for small/routine tasks) and wait for completion.
  - Spawn a **brand-new** agent per review round (`gpt-6.2-astra`, `reasoning_effort: "high"` default) with strict read-only instructions and wait for completion.
- **Otherwise**, use **Adapter 3 (`codex exec`)** below so model IDs, `model_reasoning_effort`, ephemeral fresh sessions, and `-s read-only` sandboxing are strictly enforced.

---

## 3. Standalone CLI (`codex exec`) — Claude Code, Antigravity IDE/CLI, Gemini CLI, or Codex Fallback

When running outside T3 Code without `delegate_task`, dispatch Sol and Astra via the installed `codex` CLI.

### Preflight Check

```bash
command -v codex
```

If `codex` is not installed or not authenticated, stop and report the missing dependency to the user.

### Dispatch Implementer (`gpt-6.2-sol`, `xhigh` default)

```bash
codex exec \
  --ephemeral \
  -m gpt-6.2-sol \
  -c 'model_reasoning_effort="xhigh"' \
  -s workspace-write \
  -C "$PWD" \
  -o /tmp/neon-orch-sol-last.md \
  - <<'EOF'
<Self-contained Implementation or Revision Brief>
EOF
```

- Use `model_reasoning_effort="high"` only for small/routine tasks or when explicitly requested.
- If the task requires network access or commands blocked by `workspace-write` (e.g., installing packages), ask the user or adjust `-s` only within the host environment's permitted policy.
- Read `/tmp/neon-orch-sol-last.md` (or stdout) for Sol's final summary.

### Dispatch Independent Reviewer (`gpt-6.2-astra`, `high`)

```bash
codex exec \
  --ephemeral \
  -m gpt-6.2-astra \
  -c 'model_reasoning_effort="high"' \
  -s read-only \
  -C "$PWD" \
  -o /tmp/neon-orch-astra-last.md \
  - <<'EOF'
<Self-contained Review or Re-Review Brief>
EOF
```

- `--ephemeral` guarantees a clean, unbiased review session every round.
- `-s read-only` enforces at the sandbox layer that Astra cannot edit files.
- **Keep your own review independent**:
  - If your harness supports background shell commands (e.g., Claude Code `Bash` with `run_in_background: true`, Antigravity `run_command` async), launch Astra in the background and perform your own `git diff` review while Astra runs.
  - If your harness runs shell commands synchronously (e.g., Gemini CLI without background shell), inspect `git diff` and write down your own review findings **before** executing the `codex exec` review command so you do not anchor on Astra's output.

---

## Harness-Specific Notes

| Harness | Inside T3 Code | Standalone (Outside T3 Code) | Skill Discovery Paths |
| :--- | :--- | :--- | :--- |
| **Antigravity** | Adapter 1 (`delegate_task`) | Adapter 3 (`codex exec`) | `~/.gemini/config/skills/`, `.agents/skills/` |
| **Claude Code** | Adapter 1 (`delegate_task`) | Adapter 3 (`codex exec`) | `~/.claude/skills/`, `.claude/skills/`, `.agents/skills/` |
| **Codex** | Adapter 1 (`delegate_task`) | Adapter 2 (`spawn_agent`) or Adapter 3 (`codex exec`) | `~/.codex/skills/`, `.agents/skills/` |
| **Gemini CLI** | Adapter 1 (`delegate_task`, if mounted) | Adapter 3 (`codex exec`) | `~/.gemini/skills/`, `.gemini/skills/`, `.agents/skills/` |

> **Why not native Claude/Antigravity/Gemini subagents for Sol & Astra?**
> Native subagents in Claude Code (`Agent`), Antigravity (`invoke_subagent`), and Gemini CLI only run models from their own provider and cannot invoke Codex models (`gpt-6.2-sol` / `gpt-6.2-astra`). Use T3 Code's `delegate_task` when available, or `codex exec` standalone.
