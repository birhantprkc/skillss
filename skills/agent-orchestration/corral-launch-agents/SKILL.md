---
name: corral-launch-agents
description: Launch one or many CLI coding agents through Corral Design 1 in fresh, isolated Git worktrees. Use when the user asks to launch, spawn, batch-start, or programmatically create Corral agents, especially a Pi Agent in a specific repository or exact worktree path. Unlike the Herdr skill, this creates new Corral-owned tasks and reconstructs the plugin environment for calls made outside Herdr.
---

# Corral Launch Agents

Launch agents through Corral's real `launch` pipeline. Let Corral create the branch, worktree, Herdr workspace, pane, agent process, metadata, and persisted task record. Do not recreate that sequence with raw Git or Herdr commands.

## Resolve the request

Distinguish these targets before acting:

- **Specific repository:** pass its checkout path with `--repo`. Corral resolves the primary checkout and creates a fresh worktree.
- **Specific new worktree location:** also pass `--worktree-path`. The destination is for a new Corral-created checkout; it is not an existing Herdr workspace ID.
- **Existing Corral/Herdr agent or resumable CLI session:** do not launch another Corral task. Use the `herdr` and relevant harness skill instead.
- **Existing external worktree:** use Corral adoption only when the user asks to adopt it. Launching always creates a new task/worktree.

Treat launching as a state-changing action. Execute it only when the user asks to launch; inspection, explanation, or dry-run requests do not authorize a launch.

## Resume existing Cursor CLI sessions

Corral's launch helper is the wrong tool for this workflow. Load the `herdr` and `cursor-cli` skills, then:

1. Identify the named Herdr session, repository workspace, original worktree path, and exact Cursor chat ID.
2. Select only non-empty chats. Use `meta.json`'s `updatedAtMs` or transcript update time—not the chat directory's modification time. Distinguish the human-facing parent chat from review subagents. If the user named a specific chat, use that exact ID.
3. Inspect live panes first. Never resume the same chat ID concurrently in two panes; reuse the existing pane instead.
4. Create a tab under the existing repository workspace while keeping the original worktree as that tab's cwd:

```bash
herdr --session "$SESSION" tab create \
  --workspace "$REPO_WORKSPACE_ID" \
  --cwd "$WORKTREE_PATH" \
  --label "$LABEL" \
  --no-focus
```

5. Capture the returned root pane ID and resume the exact chat without sending a new prompt:

```bash
herdr --session "$SESSION" pane run "$PANE_ID" \
  "cursor-agent --yolo --trust --resume $CHAT_ID"
```

6. Verify the agent became idle and the expected history loaded:

```bash
herdr --session "$SESSION" agent wait "$PANE_ID" --until idle --timeout 30000
herdr --session "$SESSION" pane read "$PANE_ID" \
  --source recent-unwrapped \
  --lines 200
```

Confirm the terminal title, conversation tail, and `foreground_cwd`. Resuming restores conversation history, not the old process environment; the original worktree preserves its filesystem state.

## Use the helper

Resolve the installed skill directory without hardcoding a machine path:

```bash
CORRAL_LAUNCH_SKILL_DIR="${AGENT_SKILLS_DIR:-$HOME/.agents/skills}/corral-launch-agents"
CORRAL_LAUNCH_HELPER="$CORRAL_LAUNCH_SKILL_DIR/scripts/corral_agents.py"
```

Run preflight checks first. Always pass the intended named Herdr session when known:

```bash
python3 "$CORRAL_LAUNCH_HELPER" doctor --session corral
python3 "$CORRAL_LAUNCH_HELPER" list-presets --session corral --agent pi
```

If the session is stopped, report that clearly. Do not silently start, stop, or delete Herdr sessions.

Dry-run the exact launch before creating anything:

```bash
python3 "$CORRAL_LAUNCH_HELPER" launch \
  --session corral \
  --repo <repository-checkout> \
  --task "<task title>" \
  --preset pi \
  --priority 2 \
  --prompt "<task prompt>" \
  --no-focus \
  --dry-run
```

Review the resolved session, preset, repository, worktree destination, base, priority, dirty-checkout policy, and prompt length. Then repeat without `--dry-run` only when the launch is authorized.

To choose an exact new worktree directory, add:

```bash
--worktree-path <new-worktree-path>
```

To launch several agents, use `batch` with a JSON task file. Corral itself permits many launcher processes but caps the preparation pipeline at three concurrent jobs. Read [references/batch-launches.md](references/batch-launches.md) before a batch launch.

## Verify every launch

The helper returns only after Corral has created the worktree and Herdr has detected the interactive agent. Verify persisted and live state:

```bash
python3 "$CORRAL_LAUNCH_HELPER" status \
  --session corral \
  --task "<task title>" \
  --live
```

Report the Corral task ID, title, priority, preset, worktree path, pane ID, persisted launch status, and live Herdr agent status. If launch fails after worktree creation, preserve the failed task/worktree for diagnosis; do not clean it automatically.

## Safety rules

- Never pass `--allow-dirty` without explicit user acceptance. It does **not** copy dirty primary-checkout changes; it merely proceeds without them.
- Never use `--mock-state` outside Corral's smoke test.
- Never insert API keys into command arguments, prompts, output, or preset files. Use the provider's normal credential store or environment.
- Never edit `presets.toml` merely to inspect it. Ask before adding or changing a preset because it affects future launches.
- Do not use Pi's `--print`, RPC mode, export, or model-list commands in a Corral preset. Corral and Herdr expect a long-lived interactive TUI.
- Prefer `--no-focus` for automation so a batch does not steal the user's active pane.
- Use the same Herdr binary/session that runs Corral; a different binary or session can target the wrong runtime.

## Read detailed references

- Read [references/corral-design1.md](references/corral-design1.md) before troubleshooting, changing presets, using the raw Corral CLI, or handling unusual runtime paths. It documents the architecture, every native launch option, preset fields, environment variables, state model, and failures.
- Read [references/pi-agent.md](references/pi-agent.md) before selecting Pi provider/model/thinking/session/tool/skill options or creating a specialized Pi preset.
- Read [references/batch-launches.md](references/batch-launches.md) before launching multiple agents from a file.

For helper options, run:

```bash
python3 "$CORRAL_LAUNCH_HELPER" --help
python3 "$CORRAL_LAUNCH_HELPER" launch --help
```
