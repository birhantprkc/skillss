---
name: push-skill-to-github
description: Commit and push agent-skill changes to the user's private skills GitHub repo (`<your-github-username>/private-skills`, rooted at `<repo-root>`). Use after creating or updating any skill, when the user says "push the skill", "push skills to github", "save the skill to my repo", or "update the skills repo". Handles staging, committing, and pushing directly in the current shell — works in any agent, no cmux needed.
---

# Push Skills to GitHub

For committing any skill change to the user's private skills repo: **`<your-github-username>/private-skills`**, git root **`<repo-root>`** (this is also the canonical skill folder; `.claude` and `.pi/agent/skills` symlink to `<repo-root>/skills`). Pushes here publish a sanitized public mirror to `davidondrej/skills` — never push directly to that public repo.

Use this after creating or editing a skill. If the skill is distributed to all agents, do that first (`distribute-skill-to-all-agents`), then run this to push the canonical copy.

## Steps

Run git directly in the current shell. A push is a quick synchronous command — no persistent terminal, pane, or cmux needed.

1. **Check what's pending** in `<repo-root>`:
   ```bash
   cd <repo-root> && git status --short
   ```
   If there are unrelated uncommitted changes, stage only the skill folder(s) you changed — don't bundle unrelated work under one commit message.
2. **Stage, commit, push**:
   ```bash
   cd <repo-root> && git add skills/<skill-name> && git commit -m "<concise message>" && git push
   ```
3. **Verify** the push landed: the push output must show `main -> main`. If it doesn't, report the error — don't claim success.

## Notes
- Always run git from `<repo-root>` (the repo root), not `<repo-root>/skills`.
- Write a concise, specific commit message describing the skill change.
- Only push to GitHub when the user asks. Don't push speculatively.
