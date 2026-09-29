# Local agents

A local MCP server that delegates cheap operational work to a model running in Ollama. Raw output (logs,
`kubectl`, `gh`) stays between the server and the local model; only a short structured result comes back
into Claude Code's context. Goal: save tokens without losing control. Claude remains the main agent for
reasoning, code, architecture and decisions.

!!! note "Source of truth"
    Code and README in `~/Projecs/local-agents` (outside `cmoreira-dev/`, not a git repository).
    Changed a tool, the config or the routing? Update this page (PT and EN) and the root `CLAUDE.md`.

## Architecture

```text
Claude Code
    | MCP over stdio (registered in ~/Projecs/.mcp.json)
    v
local-agents (Node, src/server.mjs)
    +-- deterministic workers: collect evidence with allowlists, reduce and sanitize
    +-- JSON validation and response size cap
    v
Ollama (http://localhost:11434, OLLAMA_HOST) -> qwen2.5:14b
```

Workers do as much as possible in code (sync status, health, revision and counts come from JSON, not from
the model); the model only explains, summarizes or drafts. The local model **never runs commands**: the
worker does, through `spawn` without a shell, after validating structured arguments against an allowlist.

## Tools

All are read-only; the `draft_*` tools only return drafts (nothing is written to disk).

| Tool | Input | Use | Measured (qwen2.5:14b) |
|---|---|---|---|
| `check_actions` | `repos?`, `limit?` (1-20, default 3), `include_failure_logs?`, `verbose?` | GitHub Actions status across the 14 configured repos; for failed runs, probable cause and a hint | 1.2 s, 87 B when all green |
| `check_argocd` | `namespace?` (`argocd`), `only_problems?` (`true`) | Argo CD apps not Synced+Healthy, with a short explanation each | 22.7 s, 1.9 KB (46 apps, 3 flagged) |
| `investigate_infra` | `resource`, `namespace?`, `helm_release?`, `include_logs?`, `include_events?`, `include_metrics?`, `verbose?` | Pod/workload/Helm: state, Warning events, logs (only for `type/name`) | 11.8 s, 273 B |
| `investigate_gitops` | `application`, `namespace?` (`argocd`), `repository_path?`, `verbose?` | Git vs Argo CD vs cluster for one app. Sync, health and revision come from code; the `argocd` CLI is optional | 10.9 s, 892 B |
| `analyze_logs` | `text`, `source?`, `context?` | Structured log diagnosis (errors, patterns, causes, next steps) | depends on size |
| `draft_docs_update` | `repository_path`, `docs_page`, `base_ref?` (`HEAD~1`), `instructions?` | Reads the repo diff and the current page; proposes markdown sections to change | 60.3 s, 1.4 KB |
| `draft_code` | `task`, `files` (1-5), `constraints?` | Small, well-specified task: returns proposed file contents | 17.5 s, 768 B |

Load the tools in a session (they are deferred):
`ToolSearch("select:mcp__local-agents__check_actions,mcp__local-agents__check_argocd,mcp__local-agents__investigate_infra,mcp__local-agents__investigate_gitops,mcp__local-agents__analyze_logs,mcp__local-agents__draft_docs_update,mcp__local-agents__draft_code")`.

## When to use (routing)

Short rule in section 5 of the root `CLAUDE.md`. In short: CI green? → `check_actions`. Argo synced? →
`check_argocd`. Pod or Helm trouble → `investigate_infra`. One app's drift → `investigate_gitops`.
Log or output over ~30 lines → `analyze_logs`. Docs stale after a change → `draft_docs_update`. Small
mechanical task → `draft_code`. Always review before applying.

## Known model limits (qwen2.5:14b)

- Does only part of multi-step edits and reformats code unasked (`draft_code`).
- Blurs component names in docs (`draft_docs_update`) and gives generic Argo CD and Actions explanations.
- So: anything decision-critical is verified with a direct command, and `draft_*` is never applied blindly.
- `qwen2.5-coder:14b` is the natural candidate for the `code` agent (`ollama pull qwen2.5-coder:14b`, ~9 GB,
  then change `models.code.model` in the config). Not pulled yet.

## Configuration (`config/local-agents.yaml`)

| Key | Effect |
|---|---|
| `ollama.host`, `timeout_ms` (180000), `max_input_bytes`, `num_ctx` (16384) | Connection and context window (without `num_ctx` Ollama may silently truncate input) |
| `models.{logs,infra,gitops,actions,argocd,docs,code}.model` | Model per agent (all `qwen2.5:14b` today) |
| `actions.org`, `actions.repos` | Org and repos `check_actions` accepts |
| `security.allowed_paths` | Readable roots (`/Users/oliveirac/Projecs`); paths resolved with realpath |
| `security.max_output_chars` (6000), `max_draft_output_chars` (30000) | Maximum result size; above it the result becomes a `{truncated, partial, note}` envelope |

`LOCAL_AGENTS_CONFIG` points to another file; `OLLAMA_HOST` to another host.

## Security

- Read-only. `kubectl`/`helm`/`argocd`/`git`: subcommand allowlist and a block on state-changing verbs
  (`apply`, `delete`, `patch`, `scale`, `sync`, `commit`, `push` ...).
- `gh`: three exact shapes (`run list`, `run view --log-failed`, `pr list`), fixed `--json` fields, numeric
  ids, `org/repo` restricted to `actions.org`. `merge`, `rerun`, `cancel`, `comment` and `api` are impossible.
- Arguments with shell syntax are rejected; no `eval`, `sh -c` or model-built commands.
- Input is sanitized before it reaches the model (Authorization, Bearer, passwords, API keys, connection
  strings, JWTs). A practical barrier, not a formal guarantee.
- `draft_*` reject relative paths, `..`, symlinks, `.env*`, `*.pem`, `*.key`, `secret*`, `*credentials*`,
  and re-validate the paths the model proposes in its output.

## Operation

- Ollama runs as a user service: `brew services start ollama`; check with
  `curl -s localhost:11434/api/version` and `ollama list`.
- After changing tools or config: reconnect the server (`/mcp` in Claude Code) to load the new version.
- Tests: `cd ~/Projecs/local-agents && npm test` (31 tests, no network or Ollama needed).
- The server's own logs go to `stderr` as JSON (agent, model, duration, sizes; no raw content).

## Adding a tool

1. Worker in `src/workers/<name>-worker.mjs` (deterministic collection, reduction, one model call only if needed).
2. Output validator in `src/structured.mjs`/`output-validator.mjs` and a model entry in `models.<name>` in the config.
3. Registration in `src/server.mjs` with a description saying WHEN to use it and that it is read-only.
4. Tests with injected executor and model (no network) and a live check through `fixtures/smoke.mjs`.
5. Update this page and the routing table in the root `CLAUDE.md`.

## Verifying it works

Do not rely on Claude's word that the agents were called: use these proofs, from simplest to most independent.

1. **Connection:** `/mcp` in Claude Code should list `local-agents` connected with 7 tools. Without it no
   session can call them (`ToolSearch` from the Tools section finds nothing).
2. **Usage log in the server:** `cd ~/Projecs/local-agents && npm run stats`. Shows server starts (sessions
   that connected), calls per tool, errors, average latency and estimated tokens kept out of context. The log
   is `logs/usage.jsonl`, one JSON line per call, no raw content.
3. **Proof on the Ollama side (independent of the server):**
   `grep '/api/chat' /opt/homebrew/var/log/ollama.log | tail` lists every call with time and duration;
   `ollama ps` shows the loaded model while a call runs.
4. **Active test:** in a new session ask for "CI status" or an "Argo CD health check"; the
   `mcp__local-agents__...` call shows in the transcript and adds a new line to `usage.jsonl`.
5. **If it does not connect:** the current registration lives in `~/Projecs/.mcp.json` (project scope: only
   applies to sessions under `~/Projecs` and may ask for approval again). Robust alternative, user scope:
   `claude mcp add --scope user local-agents -- node /Users/oliveirac/Projecs/local-agents/src/server.mjs`.
   The server's stderr is in `~/Library/Caches/claude-cli-nodejs/<project>/mcp-logs-local-agents/`.

**Reading the numbers:** `est_tokens_kept_out` is a lower bound (input sent to the local model minus the result
returned) and small calls give 0. The gain shows with long logs, `kubectl describe` and `check_actions` over many
repos; the raw `gh`/`kubectl` output the worker reduces before the model is not counted.
