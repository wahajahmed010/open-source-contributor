# open-source-contributor

Autonomous GitHub contribution agent using subagent orchestration for real code fixes.

## Description

Scouts open-source projects for contribution opportunities, analyzes issues, implements fixes using AI subagents, and submits PRs under your identity. Supports three difficulty levels from beginner-friendly to advanced.

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
- tektoncd/pipeline (already have PR #9950)
- fleetdm/fleet
- refined-github
- napalm-automation/napalm (already have PR #2320)
- huggingface/smolagents (already have PR #2255)
- collective/icalendar (already have PR #1369)
- go-git/go-git
- sphinx-doc/sphinxcontrib-htmlhelp
- prisma/prisma (already have PR #1 on fork)
- BeanieODM/beanie (issue already fixed upstream)

---

## Difficulty Levels

### Level 1: Easy — Warm-up Contributions

**Target:** Small, well-scoped fixes in familiar territory. Good for building reputation.

| Criteria | Requirement |
|----------|------------|
| Labels | `good first issue`, `help wanted`, `bug` |
| Stars | 500+ |
| Issue age | 3-30 days |
| Comments | < 10 |
| Scope | Single file, < 30 lines changed |

**Typical fixes:**
- UI text / label corrections
- Missing error handling (try/except, null checks)
- Simple configuration fixes
- Small accessibility improvements
- Missing validation on user inputs

**Research query:**
```
label:"good first issue" OR label:"help wanted" OR label:"bug"
language:python|javascript|typescript|go|rust
state:open sort:updated
```

**Timeout:** `runTimeoutSeconds: 600` for Worker (simpler fixes, faster)

---

### Level 2: Intermediate — Real Feature Work

**Target:** Multi-file changes requiring understanding of codebase architecture. This is the default level.

| Criteria | Requirement |
|----------|------------|
| Labels | `bug`, `feature`, `performance`, `enhancement` |
| Stars | 1,000+ |
| Issue age | 7-60 days |
| Comments | < 5 |
| Scope | Multi-file, 30-150 lines changed |

**Typical fixes:**
- Dependency version bumps with regression tests
- Adding missing API parameters or options
- Fixing edge cases in data processing
- Implementing missing methods or handlers
- Performance improvements (caching, lazy loading)

**Research query:**
```
label:"bug" OR label:"feature" OR label:"performance" OR label:"enhancement"
-language:"good first issue" -language:"help wanted"
language:python|javascript|typescript|go|rust stars:>1000
state:open sort:updated
```

**Timeout:** `runTimeoutSeconds: 900` for Worker (needs more time for multi-file changes)

---

### Level 3: Advanced — Architecture & Deep Fixes

**Target:** Complex changes requiring deep codebase understanding, cross-module impact, or algorithmic thinking.

| Criteria | Requirement |
|----------|------------|
| Labels | `bug`, `feature`, `performance`, `enhancement`, `design` |
| Stars | 2,000+ |
| Issue age | 7-90 days |
| Comments | < 8 |
| Scope | Multi-module, 100-500+ lines changed |

**Typical fixes:**
- Race condition / concurrency bug fixes
- Memory leak detection and remediation
- API redesign or new endpoint implementation
- Database query optimization
- Plugin/extension system development
- Cross-browser/cross-platform compatibility fixes

**Research query:**
```
label:"bug" OR label:"feature" OR label:"performance" OR label:"enhancement" OR label:"design"
language:python|javascript|typescript|go|rust stars:>2000
state:open sort:updated
```

**Timeout:** `runTimeoutSeconds: 1200` for Worker (complex changes need more time)

**Optional:** Spawn a **Council of L3ms** to evaluate approach before implementing. Use when the fix involves:
- Security-sensitive code paths
- Breaking API changes
- Database schema changes
- Performance-critical hot paths

---

## Pipeline (Subagent-Based)

This skill uses the **subagent-orchestration pattern** with research → implement → report phases:

### Phase 1: Research (spawn Researcher agent)

Spawn a Researcher agent with `toolsAllow: ["ollama_web_fetch", "ollama_web_search"]` to:

1. Search GitHub API using the query for the **configured difficulty level**
2. Filter results by:
   - Open for the right age range (varies by level)
   - Comment count within threshold
   - No existing PRs already addressing the issue
   - Not in the excluded repos list
3. For each candidate, fetch the issue body, repo structure, and recent PRs
4. Return: top 3 issues ranked by impact-to-effort ratio, with full context

### Phase 2: Evaluate & Implement (spawn Worker agent)

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
   - Difficulty level tag
   - Note that this was AI-assisted

### Phase 2.5: Council Review (Level 3 only — optional)

For Level 3 fixes that are security-sensitive or architecturally complex, spawn a **Council of LLMs** before implementing:

1. Spawn 3 parallel subagents: Strategos, Analyticos, Creativos (use your preferred models)
2. Pass the issue context and proposed approach to each
3. Synthesize their verdicts
4. If consensus is "don't implement" or high risk → skip this issue, move to next candidate

### Phase 3: Report

Return a final summary with:
- Issue title + link
- PR link
- Lines changed
- Difficulty level (1/2/3)
- Brief approach description

If no suitable issues found, report that. Do NOT force low-quality contributions.

---

## Configuration

Set the difficulty level in `~/.openclaw/workspace/contrib-scout/config.json`:

```json
{
  "difficulty_level": 2,
  "github_token_path": "~/.openclaw/.github_token",
  "max_contributions_per_night": 1,
  "languages": ["python", "javascript", "typescript", "go", "rust"],
  "excluded_repos": ["MemPalace/mempalace", "puppeteer/puppeteer", "TwiN/gatus", "dundee/gdu", "FilipePS/Traduzir-paginas-web"]
}
```

**Default:** Level 2 (Intermediate). Change to 1 for warm-up, 3 for deep work.

---

## Critical Rules

- **Do NOT use `sessions_yield` for intermediate status** — only return the final report
- **Do NOT send partial progress messages** — the first message you return is what gets delivered
- **Use GitHub API** (urllib + token) for fork/PR operations, **git CLI** for clone/branch/push
- **Write `.py` files** for any scripts — never use `python3 -c` inline
- **Pre-fetch web content yourself** for Worker agents — they can't browse
- **Keep task descriptions under 2000 words** — longer = context overflow
- **Use `lightContext: true`** on all subagent spawns
- **Researcher:** Use `glm-5.1` for web search and synthesis tasks
**Worker:** Use `minimax-m2.7` for coding and implementation (fallback: `qwen3-coder:480b:cloud`)
**Tester:** Use `qwen3-coder:480b:cloud` if Worker was `minimax-m2.7`, or `minimax-m2.7` if Worker was `qwen3-coder:480b:cloud`
**Reviewer/Submitter:** Use `gemma4:31b` for pre-flight review and PR submission
**Scale timeouts with difficulty:** L1=600s, L2=900s, L3=1200s
- **Never force a contribution** — if no good fit, report "no suitable issues found"

## Cron Configuration

Runs daily at midnight (Europe/Berlin). Uses the multi-agent orchestrator pattern.

### Cron Agent Pattern

The cron agent (Buck/main agent) acts as **orchestrator** and delegates actual work to specialized subagents:

1. **Researcher agent** with `toolsAllow: ["ollama_web_fetch", "ollama_web_search"]` — use `glm-5.1` for search and synthesis
   - Finds 3 intermediate-difficulty issues from open source repos
   - Focus: documentation fixes, typo fixes, small bug fixes, test additions, config improvements
   - Requirements: repo 1000+ stars, issue labeled "good first issue" or "help wanted" or "bug", last activity <30 days
   - Returns: repo URL, issue URL, issue title, difficulty estimate, why it's a good fit

2. **Worker agent** — use `minimax-m2.7` for coding and implementation (if it fails/times out, respawn with `qwen3-coder:480b:cloud`)
   - Picks the best issue from Researcher results
   - Forks repo, implements fix, creates PR
   - Uses GitHub API (urllib + token at `~/.openclaw/.github_token`) for fork/PR
   - Uses git CLI for clone/branch/push
   - Writes `.py` files for scripts — never `python3 -c` inline
   - Tests fix locally before pushing
   - Returns: PR URL, repo name, issue number, summary of changes

3. **Tester agent** — use `qwen3-coder:480b:cloud` if Worker was `minimax-m2.7`, or `minimax-m2.7` if Worker was `qwen3-coder:480b:cloud`
   - Validates the fix by running the repo's existing test suite
   - If tests fail, reports back specific failures for Worker to address
   - Returns: pass/fail, test output summary, any issues found

4. **Reviewer/Submitter agent** — use `gemma4:31b` for final review and PR submission
   - Pre-flight checklist: code quality, security patterns, style consistency
   - Creates the PR via GitHub API with proper description and AI disclosure
   - Returns: final PR URL and submission confirmation

**CRITICAL RULES for cron execution:**
- The orchestrator MUST use `sessions_spawn` for both phases — never do research or coding itself
- Each subagent gets `runTimeoutSeconds: 900` and `lightContext: true`
- Each subagent gets the appropriate model for its role:
  - Researcher: `glm-5.1`
  - Worker: `minimax-m2.7` (fallback: `qwen3-coder:480b:cloud`)
  - Tester: opposite model from Worker
  - Reviewer/Submitter: `gemma4:31b`
- The orchestrator waits for both phases to complete before returning the final report
- Never use `sessions_yield` for intermediate status — only return the final consolidated report
- If a subagent times out or fails, report what was accomplished and move on

## Storage

```
~/.openclaw/workspace/contrib-scout/
├── repos/              # Cloned repositories (cleaned up after each run)
├── logs/               # Activity + audit trail (JSONL)
├── config.json         # Difficulty level + settings
└── nightly-report.json # Daily summary
```

## Companion Skills

- **subagent-orchestration** — Required. Provides spawn patterns, timeout config, sandbox constraints.
- **council-of-llms** — Optional. For Level 3 complex decisions before implementing.