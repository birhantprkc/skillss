---
name: who-is-this
description: 'Deep-scrape a person across X, LinkedIn, GitHub, and the open web to judge if they are legit. Manual-only; invoke with /who-is-this. Use when the user says who-is-this, "who is this", "who is this guy", "research this person", "are they legit", "vet this founder", or drops a profile screenshot. Differentiator vs twitter-alpha: one-person background check, not a 7-person idea list. Differentiator vs deep-research: platform scrapes plus a short verdict, not a long cited memo.'
disable-model-invocation: true
---

# who-is-this

Figure out who a person is and whether their story holds.

Read the `deepapi` skill first. All search and scraping goes through DeepAPI. Never use built-in search, fetch, or a browser.

## Seed

Need a person. A name, handle, URL, or profile screenshot is enough.

If missing, ask once. Do not guess a different person.

Verify identity before going deep. Bio, company, location, and photo must match. If two people share the name, stop and ask.

## Research

Find their X, LinkedIn, and GitHub first (dedicated endpoints, not `site:` search). Then run these in parallel:

1. **GitHub activity.** `POST /v1/scrape/github/profile` with `includeRepos: true`, plus recent PRs (`POST /v1/scrape/github/search`, `type: "pulls"`, `query: "author:<username>"`, `sort: "updated"`).
2. **Last 10 LinkedIn posts.** `POST /v1/scrape/linkedin/posts`, `maxItems: 10`. Also scrape the LinkedIn profile if you have the slug.
3. **Deep research.** `POST /v1/research/deep` on who they are, what they have actually done, and whether the public story holds.
4. **50 newest X posts.** `POST /v1/scrape/twitter/search` with the handle, `sort: "latest"`, `maxItems: 50`. Also pull `POST /v1/scrape/twitter/user`.

Follow DeepAPI polling. If a platform is missing or private, say so. Do not invent a profile.

## What to extract

From the scrape, keep only:

- Real track record (jobs, products, exits, code, talks)
- Claims vs evidence (especially self-reported revenue)
- What they actually post about
- The honest archetype: who they are in practice, not their bio. Builder and marketer are examples, not the only options. Name the real type (operator, researcher, grifter, investor, recruiter, hobbyist, etc.) when that is more true.

Ignore congrats, logo spam, paid "king of X" press, and unverified follower-count flexing.

Self-reported numbers stay labeled as their claims.

## Output

Short. Plain English. Nice readable markdown. No tables. No tweet dumps. No research narration.

markdown
**Full name** — [@handle](https://x.com/handle)

- LinkedIn: [url]
- GitHub: [url]
- Site: [url]

**Who they are**
Two or three short sentences.

**What they have done**
The real resume. Dates and companies. Label unverified numbers.

**What they are known for**
3–4 bullets.

**Archetype**
One brutally honest line. Who they actually are. Builder and marketer are examples, not a forced choice. Say why.


Omit a missing profile instead of faking it.

If the user asks a follow-up, answer even shorter. Do not re-run the full scrape unless the first pass missed that platform.
