# Open Source Contributor

Autonomous GitHub contribution agent using the **Architect-Builder** pattern. The main agent handles all git/network I/O; subagents handle focused cognitive work only.

**ClawHub:** https://clawhub.ai/skills/open-source-contributor

## Why the Architect-Builder Pattern?

Previous versions spawned subagents to do everything — clone repos, modify code, run tests, create PRs. This failed because:
- Each tool call through cloud model proxies takes 5-58 seconds
- 50+ tool calls = 12+ minutes of pure latency
- Large repo clones eat the time budget
- Subagents timed out before completing work

The fix: **The main agent does all I/O. Subagents only do focused cognitive work on pre-staged content.**

```
┌─────────────────────────────────────────────────────┐
│  MAIN AGENT — The Architect & Builder                │
│                                                       │
│  Research → Stage → Delegate → Apply → Push         │
│  (All git operations, GitHub API, file I/O)          │
└─────────────────────────────────────────────────────┘
              │
              │ File contents pasted inline
              ▼
┌─────────────────────────────────────────────────────┐
│  SUBAGENT — The Specialist (600s max)                 │
│                                                       │
│  Receives files + issue context → Returns code changes│
│  (No git, no API calls, no web browsing)             │
└─────────────────────────────────────────────────────┘
```

## Difficulty Levels

### Level 1: Easy — Warm-up Contributions
- Single file, < 30 lines changed
- Labels: `good first issue`, `help wanted`, `bug`
- Stars: 500+, Age: 3-30 days

### Level 2: Intermediate — Real Feature Work (Default)
- Multi-file, 30-150 lines changed
- Labels: `bug`, `feature`, `performance`, `enhancement`
- Stars: 1,000+, Age: 7-60 days

### Level 3: Advanced — Architecture & Deep Fixes
- Multi-module, 100-500+ lines changed
- Optional Council of LLMs review before implementing
- Stars: 2,000+, Age: 7-90 days

## API-First Approach

For simple fixes (1-3 files), **skip `git clone` entirely** and use the GitHub API directly:

1. Read files via `GET /repos/{owner}/{repo}/contents/{path}`
2. Write changes via `PUT /repos/{owner}/{repo}/contents/{path}`
3. Create PR via `POST /repos/{owner}/{repo}/pulls`

This completes in 2-5 minutes instead of 15+.

**Use API-first when:** Fix touches 1-3 files, no test suite needed, no complex branching.

**Use clone when:** Fix touches 4+ files, needs local testing, or requires understanding project structure.

## Pipeline

### Phase 1: Research (Main Agent)
Find suitable issues using GitHub Search API. Filter by difficulty level criteria, exclude already-contributed repos, verify no open PRs.

### Phase 2: Stage (Main Agent)
Fork repo via GitHub API. Shallow clone (`--depth 1`). Create branch. Read `CONTRIBUTING.md` and relevant source files. Identify files that need changes. Prepare a focused task for the subagent with all file contents pasted inline.

### Phase 3: Implement (Subagent — 600s max)
Spawn a specialist subagent that receives:
- Full file contents (pasted inline, not file paths)
- The issue description
- Project style/lint requirements
- A request for EXACT code changes

The subagent returns the modified code. It does NOT clone repos, push code, or create PRs.

### Phase 4: Apply & Push (Main Agent)
Apply the subagent's changes. Run linters/tests. Fix any remaining issues. Commit, push, create PR via GitHub API.

### Phase 4.5: Council Review (Level 3 only)
For complex fixes, spawn a Council of LLMs before Phase 3. Three different models evaluate the approach from strategic, analytical, and creative perspectives.

## Quality Standards

- **Meaningful contributions only** — no typo fixes, no placeholder PRs
- **Must pass CI/tests** where available
- **Must reference the issue** it fixes
- **Must disclose AI assistance**
- **No emojis** in PR titles, descriptions, or comments — professional tone throughout

## Configuration

Store settings in `contrib-scout/config.json`:

```json
{
  "difficulty_level": 2,
  "github_token_path": "~/.openclaw/.github_token",
  "max_contributions_per_night": 1,
  "languages": ["python", "javascript", "typescript", "go", "rust"],
  "excluded_repos": []
}
```

### Council Models (Optional — Level 3 only)

For Council of LLMs review, configure your preferred models in `~/.openclaw/council-config.json`:

```json
{
  "council_models": [
    "your-strategic-model",
    "your-analytical-model",
    "your-creative-model"
  ],
  "default_timeout": 900,
  "max_tokens": 8192
}
```

Choose models with different strengths — strategic, analytical, creative. The more diverse, the better.

## Installation

### Via ClawHub (Recommended)

```bash
clawhub install open-source-contributor
clawhub install subagent-orchestration
clawhub install council-of-llms  # Optional, for Level 3
```

### Via GitHub

```bash
cd ~/.openclaw/skills
git clone https://github.com/wahajahmed010/open-source-contributor.git
git clone https://github.com/wahajahmed010/subagent-orchestration.git
git clone https://github.com/wahajahmed010/council-of-llms.git  # Optional
```

### GitHub Token

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Generate new token (classic) with `public_repo` scope
3. **Never** grant `repo` scope (includes private repos)
4. Store at `~/.openclaw/.github_token`

## Requirements

- OpenClaw with `sessions_spawn` capability
- Git
- Python 3.10+
- GitHub Personal Access Token with `public_repo` scope

## What Changed in v3

| Feature | v2 (Old) | v3 (Current) |
|---------|----------|-------------|
| Architecture | Subagent does everything | Architect-Builder: main agent does I/O, subagent does cognition |
| Git operations | Subagent clones/pushes | Main agent handles all git |
| API-first | No | Yes — skip clone for simple fixes |
| Subagent timeout | 900-1200s | 600s max (focused work only) |
| Council models | Hardcoded | User-configurable via `council-config.json` |
| Difficulty levels | 4 levels with approval gates | 3 levels with clear criteria |
| Emoji in PRs | Allowed | Forbidden — professional tone only |

## Companion Skills

- **[Subagent Orchestration](https://github.com/wahajahmed010/subagent-orchestration)** — Required. Provides spawn patterns, timeout config, and sandbox constraints.
- **[Council of LLMs](https://github.com/wahajahmed010/council-of-llms)** — Optional. For Level 3 complex decisions before implementing.

## License

MIT-0

## Disclaimer

This tool makes commits under your GitHub identity. Review the quality standards before use. You are responsible for all contributions made under your account.