---
name: total-review
description: 'Run both fable-review and gpt-review on the current work, then merge their findings, dedupe them, and triage which issues are real vs overthinking. Outputs one clear, concise list of actual issues for David to approve before fixing. Use when David says "/total-review", "total review", "review with both", or wants both Fable and GPT reviewers at once. Differentiator: runs BOTH reviewers and triages the merged findings — for a single reviewer use fable-review or gpt-review.'
---

# Total Review

Run both code reviews, merge the findings, and give David one shortlist of real issues to approve.

## Workflow

1. **Launch both reviewers in parallel**, in one message with two Task tool calls:
   - One `fable-review` subagent (Fable 5 Max 1M).
   - One `gpt-review` subagent (GPT 5.6 Sol Max).
   - Follow each skill's own instructions exactly: neutral, unbiased prompt; tell it what to review broadly; ask for a detailed report on critical/serious issues; concise plain-English final report.
   - Run both in the background so they work at the same time.

2. **Wait for both to finish.** Do not start triage until both reports are back.

3. **Merge and triage.** Read both reports in full. Then:
   - Combine all findings into one list.
   - Dedupe: the same issue reported by both counts once — and is almost certainly real.
   - For each finding, think deeply: is this a real bug / real risk, or is the reviewer overthinking (style preference, theoretical edge case, non-issue)?
   - Be ruthless. Most review findings are overthinking. Only keep issues that genuinely matter.

4. **Output to David** — clear and very concise:
   - A numbered list of the **actual, real issues** only, each in one line: what it is + where.
   - Mark which reviewer(s) found each: `[both]`, `[fable]`, or `[gpt]`. Issues found by both go first.
   - One short line at the end: how many findings were dropped as overthinking.
   - Then ask David: approve fixing these, or adjust the list.

5. **On approval, fix and ship.** Fix only the approved issues. Then stage, commit with a clear message, and push to GitHub (per the standard ship workflow). Do not fix anything David did not approve.

## Rules

- Keep every step's output short and in plain English.
- Do not show David the raw reviewer reports by default — only the merged shortlist. (He can ask for the full reports if he wants them.)
- Never fix an issue before David approves the shortlist.
