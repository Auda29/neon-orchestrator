# Harness Dispatch Reference

Pick the first available backend. Never silently substitute models if `gpt-6.2-sol` or `gpt-6.2-astra` is unavailable.

## 1. T3 Code (`delegate_task`) — Antigravity, Claude Code, Codex

Call `orchestrator_capabilities` once per thread first. Always use `mode: "async"` (no polling) and a **new** `delegate_task` per review round (never `t3_thread_send`).

- **Implementer (`gpt-6.2-sol`)** — `role: "implementation"`, `isolation: "shared"` (or `"worktree"` for independent parallel slices):
  ```json
  { "providerInstanceId": "codex", "model": "gpt-6.2-sol", "options": { "reasoningEffort": "xhigh" } }
  ```
  *(Use `"high"` only for small/routine tasks.)*
- **Reviewer (`gpt-6.2-astra`)** — `role: "review"`, `isolation: "shared"`, instruct strictly read-only:
  ```json
  { "providerInstanceId": "codex", "model": "gpt-6.2-astra", "options": { "reasoningEffort": "high" } }
  ```

*(If `T3_ACP_MCP_NODE` is set while MCP tools are absent, use T3 Code's `acp-mcp-call` fallback.)*

## 2. Standalone Codex (`spawn_agent` or `codex exec`)

Use native `spawn_agent` if it supports specifying `gpt-6.2-sol` (`xhigh`, or `high` for small tasks) and fresh read-only `gpt-6.2-astra` (`high`) agents. Otherwise use `codex exec` below.

## 3. Standalone CLI (`codex exec`) — Claude Code, Antigravity, Gemini CLI, Codex Fallback

Verify `command -v codex`, then dispatch fresh ephemeral runs:

```bash
# Implementer (use model_reasoning_effort="high" only for small/routine tasks)
codex exec --ephemeral -m gpt-6.2-sol -c 'model_reasoning_effort="xhigh"' -s workspace-write -C "$PWD" - <<'EOF'
<Self-contained brief>
EOF

# Independent Reviewer (read-only enforced by sandbox)
codex exec --ephemeral -m gpt-6.2-astra -c 'model_reasoning_effort="high"' -s read-only -C "$PWD" - <<'EOF'
<Self-contained review brief>
EOF
```

*Tip: Review `git diff` yourself in parallel (background shell) or right before synchronous `codex exec` so you don't anchor on Astra's output.*
