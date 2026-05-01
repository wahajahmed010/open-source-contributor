# open-source-contributor

Autonomous GitHub contribution agent using subagent orchestration for real code fixes.

## Description

Scouts open-source projects for intermediate-difficulty contribution opportunities, analyzes issues, implements fixes using AI subagents, and submits PRs under your identity.

## Quality Standards
- Every contribution must be **meaningful** — no typo fixes, no placeholder PRs
- Must pass CI/tests where available
- Must include proper description and tests if applicable
- Must reference the issue it fixes
- Must disclose AI assistance

## ⚠️ GitHub Token

The GitHub token is at `~/.openclaw/.github_token`. Read it with the token resolver at `~/.openclaw/lib/token_resolver.py`. **Never ask the user for the token — it's already on disk. Never log or print token values.**

## Excluded Repos

Do not contribute to repos we've already contributed to:
- MemPalace/mempalace
- simpler-grants-gov
- code-charity/youtube
- osslab-pku/gfi-bot
- puppeteer/puppeteer (already have PR #14916)
- TwiN/gatus (already have PR #1644)
- dundee/gdu (already have PR #558)
- FilipePS/Traduzir-paginas-web (already have PR #1007)

## Pipeline (Subagent-Based)

This skill uses the **subagent-orchestration pattern** with two phases:

### Phase 1: Research (spawn Researcher agent)

Spawn a Researcher agent with `toolsAllow: ["ollama_web_fetch", "ollama_web_search"]` to:

1. Search GitHub API for issues with 1000+ stars that are:
   - Labeled: `bug`, `feature`, `performance`, or `enhancement`
   - **NOT** labeled: `good first issue` or `help wanted` (we target intermediate)
   - Open for 7-60 days (not brand new, not stale)
   - Python, JavaScript, TypeScript, Go, or Rust repos
   - Fewer than 5 comments (not already crowded)
   - Exclude repos from the excluded list above
2. For each candidate, fetch the issue body, repo structure, and recent PRs
3. Return: top 3 issues ranked by impact-to-effort ratio, with full context

### Phase 2: Implement (spawn Worker agent)

For the best issue from Phase 1, spawn a Worker agent to:

1. Fork the repo to `wahajahmed010` using GitHub API (token at `~/.openclaw/.github_token`)
2. Clone the fork, create branch: `fix/<issue-number>-<short-description>`
3. Read `CONTRIBUTING.md` or similar docs first — follow their style
4. **Implement the fix** — write real code, not placeholder comments
5. Run existing tests. If they pass, proceed. If no tests exist, manually verify.
6. If the fix touches >50 lines or changes core logic, add or update a test
7. Push and create a PR via GitHub API with:
   - Clear description referencing the issue
   - Explanation of the approach
   - Note that this was AI-assisted

### Phase 3: Report

Return a final summary with:
- Issue title + link
- PR link
- Lines changed
- Brief approach description
- Difficulty rating (easy/medium/hard)

If no suitable issues found, report that. Do NOT force low-quality contributions.

## Critical Rules

- **Do NOT use `sessions_yield` for intermediate status** — only return the final report
- **Do NOT send partial progress messages** — the first message you return is what gets delivered
- **Use GitHub API** (urllib + token) for fork/PR operations, **git CLI** for clone/branch/push
- **Write `.py` files** for any scripts — never use `python3 -c` inline
- **Pre-fetch web content yourself** for Worker agents — they can't browse
- **Keep task descriptions under 2000 words** — longer = context overflow
- **Use `lightContext: true`** on all subagent spawns
- **Use `runTimeoutSeconds: 900`** for both Researcher and Worker

## Cron Configuration

Runs daily at midnight (Europe/Berlin). Uses the subagent-orchestration pattern as described above.

## Storage

```
~/.openclaw/workspace/contrib-scout/
├── repos/              # Cloned repositories (cleaned up after each run)
├── logs/               # Activity + audit trail (JSONL)
└── nightly-report.json # Daily summary
```

## Companion Skills

- **subagent-orchestration** — Required. Provides spawn patterns, timeout config, sandbox constraints.
- **council-of-llms** — Optional. For complex decisions about which issues to tackle.