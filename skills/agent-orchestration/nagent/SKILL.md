---
name: nagent
description: 'Launch a new bb worker thread with an explicit project, provider, model, reasoning effort, fast/default tier, worktree, and prompt. Short for "new agent". Use when the user says "nagent", "/nagent", "launch a bb session", "bb subagent", "spawn a bb thread", "new worktree agent", "launch it as its own thread, not as your subagent", "standalone bb thread", or asks to run Grok/Codex/Cursor CLI in bb on a repo. Differentiator vs bb-cli: this is only the spawn recipe. Differentiator vs codex-subagent/launch-subagent: those are Codex CLI and Cursor Task tool, not bb threads.'
---

# nagent

Spawn a bb thread the way the user launches them. Read this before any `bb thread spawn`.

Use `bb-cli` for status, tell, wait, inspect, automations. This skill is spawn-only.

**Default worker:** Codex `gpt-5.6-sol`, `--reasoning-level high`, `--service-tier fast` ("GPT 5.6 Sol High Fast"); bump to `xhigh` for larger refactors. Use this unless the user requests otherwise.

## Hard rules

- Pass `--project` explicitly. The CLI does not infer it from the current thread.
- Never guess provider or model IDs. Lookup, then spawn.
- After spawn: do **not** `bb thread open`, `--split`, or focus the new thread unless the user asks. Just launch. Report the thread id. Stop.
- Do **not** poll, `bb thread wait`, or read logs unless the user asks.
- Prefer bare `bb`. If `BB_CLI` is set, `"$BB_CLI"` is fine.

## Recipe

```bash
bb project list --json
bb provider list --json
bb provider models <provider-id> --json

bb thread spawn --json \
  --project <project-id> \
  --new-environment worktree \
  --provider <provider-id> \
  --model <model-id> \
  --reasoning-level <level> \
  --permission-mode full \
  --title "Short title" \
  --prompt "..."
```

Add `--service-tier fast` only when the user says **fast**.

## Parent / sidebar nest

`--parent-self` and `--parent-thread` are optional. Use them when it makes sense — not on every spawn.

`--parent-self` parents the new thread to the current thread (`BB_THREAD_ID`). The left sidebar then nests the child under this thread (indented, expandable), like a worker under a manager.

`--parent-thread <id>` parents it to a specific other thread. Do not combine it with `--parent-self`.

Omit both for a **root** thread — its own top-level sidebar row, not nested. This is what the user means by "not as your own subagent" or "standalone thread".

When it makes sense: this thread is coordinating the work, the user asked for a worker under this session, or the job is a clear subtask of this thread. Skip it for unrelated one-off work that should sit as its own top-level thread.

```bash
# add when it makes sense — not required
--parent-self
```

`environmentId` is often `null` in the spawn JSON. The worktree is still creating. That is fine.

## Lookup

**Project.** Match `name` (e.g. `DeepAPI`) in `bb project list --json`. Use that `id`. Confirm `sources[].path` is the repo the user named. Do not hardcode project IDs.

**Provider.** From `bb provider list --json`:

| User says | Provider id |
|---|---|
| Cursor CLI / Cursor subscription | `acp-cursor` |
| Codex / Codex subscription / ChatGPT sub | `codex` |
| Claude Code | `claude-code` |

Do not use `acp-grok` for "Grok via Cursor CLI". That is Grok Build, a different product.

**Model + reasoning.** From `bb provider models <id> --json`. Use the catalog `id`. Match the user's words to `displayName` **and** `supportedReasoningEfforts[].description`. Those two are not the same field.

If the user does not name a reasoning level, use that model's `defaultReasoningEffort` from the catalog. Do not pick extra high or max on your own. Example: Claude Code Fable 5 defaults to `high`.

## Known mappings (re-fetch if spawn rejects them)

### Cursor CLI — Grok 4.6 Extra High Fast

User: "Grok 4.6 extra high fast (via Cursor CLI subscription)"

```bash
--provider acp-cursor \
--model cursor-grok-4.6-medium \
--reasoning-level xhigh \
--service-tier fast
```

Traps:

- The model id stays `cursor-grok-4.6-medium` even for Extra High. Extra High is a **reasoning** value, not a different model id.
- On this catalog, `high` displays as **Fast**. `xhigh` displays as **Extra High**.
- "Extra High Fast" = `xhigh` **plus** `--service-tier fast`. Fast-the-reasoning-label and Fast-the-service-tier are different flags.
- Cursor ACP has no `auto` permission mode. Use `full` or `accept-edits`.
- Do not pass Cursor Task-tool slugs like `cursor-grok-4.6-high-fast`. Those are not bb model ids.

### Codex — GPT 5.6 Sol Max

User: "GPT 5.6 Sol Max, via the Codex subscription"

```bash
--provider codex \
--model gpt-5.6-sol \
--reasoning-level max
```

Traps:

- Use native `gpt-5.6-sol`, not `cursor/gpt-5.6-sol` (opencodex routed through Cursor).
- Max is `--reasoning-level max`, not a model id suffix.
- Codex supports `auto` permission mode. Still use `full` when the worker must hit prod, network, or leave the sandbox.
- Pass `--service-tier fast` only if the user asks for fast on Codex too.

## Worktree

`--new-environment worktree` is the bb-managed worktree. Do not create a git worktree by hand unless the user says so.

- New worktrees get **tracked files only**.
- Untracked files (`.env`) copy only if the repo has `.worktreeinclude`.
- Optional: `--base-branch <branch>`. Omit to use the project's default.
- Do not combine `--machine` with an existing `--environment` id.

For repos that must keep `.env` in worktrees, the repo needs `.worktreeinclude` plus optional `.bb-env-setup.sh`. Details: `bb guide environments`.

## Prompt

The worker starts blind. It does not see this chat. Write the full brief into `--prompt`.

Include:

1. **Objective** — one sentence.
2. **Constraints** — read-only vs implement; no git push; no prod writes.
3. **Skills / files to use** — name them. Skills do not carry over. Example: prod reads → `~/.claude/skills/read-prod-database/SKILL.md`.
4. **Deliverable** — what to return.
5. **Validation** — how it knows it is done.
6. **Report back** — concise report only, or diff + files changed.

Read-only jobs must say **no code changes, no git writes, no database writes** in the prompt, even if permission mode is `full`.

Use a heredoc so quotes survive:

```bash
--prompt "$(cat <<'EOF'
Objective: ...
Constraints: READ ONLY. ...
Deliverable: ...
Report back: ...
EOF
)"
```

Always pass `--json` and `--title`.

## Permission mode

| Mode | Meaning |
|---|---|
| `accept-edits` | Sandbox on. User approves escalations. |
| `auto` | Sandbox on. Provider auto-approves. Codex/Claude have this. Cursor ACP does not. |
| `full` | No sandbox. Needed for prod DB, many network tools, and "just go". |

Default for the user's investigation/build workers: `full`, with the prompt forbidding writes when the job is read-only.

## After spawn

Tell the user, then stop:

- thread id
- title
- project name
- provider + model + reasoning (+ fast if set)
- worktree, yes/no
- read-only vs implement

Do not dump the prompt. Do not open the thread. Do not wait for it.

## Unblocking the worker

If bb reports the worker is blocked on a command, file change, or permission, unblock it yourself. Do not wait for the user. Quickly read WHAT it wants to run and WHY (`bb thread interactions list <id> --json`), and if it fits the brief, approve it (`bb thread interactions approve <interactionId> <id>`). Only escalate to the user when the action is destructive, touches prod, or falls outside the brief.

## Archive (only when the user asks)

Do not auto-archive after spawn.

```bash
bb thread archive <id> --json
bb thread stop <id>
```

Archive first, then stop. Archive also archives child threads. Stop frees the agent runtime; the thread history stays.

```bash
bb thread unarchive <id> --json
bb thread list --archived
```

Do not use `bb thread delete` unless the user wants it gone forever. Do not use `--visibility hidden` as a substitute for archive. Hidden is sidebar-only; archive is the lifecycle close-out.

Bulk, one worktree/environment: `bb environment archive-threads <environment-id>`.
