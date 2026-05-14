<div align="center">

# Open Source Contributor

Autonomous GitHub contribution agent using the **Architect-Builder** pattern.

[![ClawHub](https://img.shields.io/badge/ClawHub-v3.0.0-blue?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIyNCIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDI0IDI0IiBmaWxsPSJ3aGl0ZSI+PHBhdGggZD0iTTIgMmg4djZIMnptMTIgMGg4djZoLTh6TTIgMTRoOHY2SDJ6bTEyIDBoOHY2aC04eiIvPjwvc3ZnPg==)](https://clawhub.ai/skills/open-source-contributor)
[![License: MIT-0](https://img.shields.io/badge/License-MIT--0-green.svg)](https://opensource.org/licenses/MIT-0)
[![GitHub](https://img.shields.io/badge/GitHub-Repo-181717?logo=github)](https://github.com/wahajahmed010/open-source-contributor)

**Main agent handles all I/O. Subagents handle focused cognitive work only.**

[Install](#installation) · [Quick Start](#quick-start) · [Configuration](#configuration) · [How It Works](#how-it-works) · [Difficulty Levels](#difficulty-levels)

</div>

---

## Why the Architect-Builder Pattern?

Previous versions spawned subagents to do everything — clone repos, modify code, run tests, create PRs. This failed because:

- Each tool call through cloud model proxies takes **5-58 seconds**
- 50+ tool calls = **12+ minutes of pure latency**
- Large repo clones **eat the time budget** before real work starts
- Subagents **timed out** before completing work

The fix: **The main agent does all I/O. Subagents only do focused cognitive work on pre-staged content.**

```
┌──────────────────────────────────────────────────────────┐
│  MAIN AGENT — Architect & Builder                        │
│                                                          │
│  Research ──► Stage ──► Delegate ──► Apply ──► Push    │
│  (All git ops, GitHub API calls, file I/O)              │
└──────────────────────────────────────────────────────────┘
              │
              │ File contents pasted inline
              ▼
┌──────────────────────────────────────────────────────────┐
│  SUBAGENT — Specialist (600s max timeout)                │
│                                                          │
│  Receives files + issue ──► Returns exact code changes  │
│  (No git, no API calls, no web browsing, no cloning)    │
└──────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Install

```bash
# Via ClawHub (recommended)
clawhub install open-source-contributor
clawhub install subagent-orchestration   # Required dependency

# Optional — for Level 3 Council review
clawhub install council-of-llms
```

Or from GitHub:
```bash
cd ~/.openclaw/skills
git clone https://github.com/wahajahmed010/open-source-contributor.git
git clone https://github.com/wahajahmed010/subagent-orchestration.git
git clone https://github.com/wahajahmed010/council-of-llms.git   # Optional
```

### 2. Set up your GitHub token

You need a GitHub Personal Access Token with `public_repo` scope.

<details>
<summary>How to create a GitHub token</summary>

1. Go to **GitHub Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Give it a descriptive name (e.g., `open-source-contributor`)
4. Select the **`public_repo`** scope only
5. ⚠️ **Never** grant the full `repo` scope — it includes private repo access
6. Click **Generate token** and copy it
7. Store it securely:

```bash
# Option A: File on disk (recommended)
echo "ghp_your_token_here" > ~/.openclaw/.github_token
chmod 600 ~/.openclaw/.github_token

# Option B: Environment variable
export GITHUB_TOKEN="ghp_your_token_here"
```

The agent reads the token from `~/.openclaw/.github_token` by default. Never log or print the token value.

</details>

### 3. Configure

Create `~/.openclaw/workspace/contrib-scout/config.json`:

```json
{
  "difficulty_level": 2,
  "github_token_path": "~/.openclaw/.github_token",
  "max_contributions_per_night": 1,
  "languages": ["python", "javascript", "typescript", "go", "rust"],
  "excluded_repos": []
}
```

### 4. Run

```bash
# Manual run via OpenClaw
python3 ~/.openclaw/workspace/scripts/github-issue-search.py

# Or set up a cron job (Mon/Wed/Fri at midnight)
# See the Cron Configuration section below
```

## How It Works

### The 4-Phase Pipeline

#### Phase 1: Research (Main Agent)

The main agent searches GitHub for suitable open-source issues:

1. Run the authenticated search script (`github-issue-search.py`)
2. Filter candidates by difficulty level (stars, age, labels, comment count)
3. Exclude repos already contributed to
4. Verify no open PRs already address the issue
5. Select the best candidate

#### Phase 2: Stage (Main Agent)

The main agent prepares the workspace — **all I/O happens here, not in the subagent**:

1. Fork the repo via GitHub API
2. Shallow clone: `git clone --depth 1`
3. Create branch: `fix/<issue-number>-<short-description>`
4. Read `CONTRIBUTING.md` and relevant source files
5. Identify files that need to change
6. Read the full content of those files
7. Prepare a focused task with all file contents pasted inline

#### Phase 3: Implement (Subagent — 600s max)

A specialist subagent receives:

| What | How |
|------|-----|
| File contents | Pasted inline in the task description |
| Issue description | Full text from GitHub |
| Project style/lint rules | From `CONTRIBUTING.md` or lint config |
| Expected output | Exact modified file content or unified diff |

The subagent **does NOT**:
- Clone repos or run git commands
- Make GitHub API calls
- Browse the web or install packages
- Have access to your token or credentials

It simply receives the code and returns the fix.

#### Phase 4: Apply & Push (Main Agent)

The main agent takes the subagent's output and:

1. Applies the code changes to local files
2. Runs linters/tests if available
3. Fixes any remaining issues (lint errors, test failures)
4. Commits, pushes, and creates the PR via GitHub API
5. Returns the final summary

#### Phase 4.5: Council Review (Level 3 only — optional)

For complex fixes, spawn a **Council of LLMs** before Phase 3. Three different models evaluate the approach from strategic, analytical, and creative perspectives. If consensus is "don't implement," skip the issue and move to the next candidate.

<details>
<summary>Council model configuration</summary>

Create `~/.openclaw/council-config.json`:

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

Choose models with different strengths — one for strategy, one for analysis, one for creative thinking. The more diverse, the better the output.

**Example (Ollama cloud):**
```json
{
  "council_models": [
    "ollama/kimi-k2.6:cloud",
    "ollama/deepseek-v4-pro:cloud",
    "ollama/gemma4:31b-cloud"
  ]
}
```

**Example (local models):**
```json
{
  "council_models": [
    "ollama/qwen3:32b",
    "ollama/llama3.1:70b",
    "ollama/mistral:7b"
  ]
}
```

</details>

## Difficulty Levels

### Level 1: Easy — Warm-up Contributions

| Criteria | Requirement |
|----------|------------|
| Labels | `good first issue`, `help wanted`, `bug` |
| Stars | 500+ |
| Issue age | 3-30 days |
| Comments | < 10 |
| Scope | Single file, < 30 lines changed |

**Typical fixes:** UI text corrections, missing error handling, simple config fixes, accessibility improvements, missing validation.

### Level 2: Intermediate — Real Feature Work (Default)

| Criteria | Requirement |
|----------|------------|
| Labels | `bug`, `feature`, `performance`, `enhancement` |
| Stars | 1,000+ |
| Issue age | 7-60 days |
| Comments | < 5 |
| Scope | Multi-file, 30-150 lines changed |

**Typical fixes:** Dependency bumps with tests, missing API parameters, edge case fixes, missing method implementations, performance improvements.

### Level 3: Advanced — Architecture & Deep Fixes

| Criteria | Requirement |
|----------|------------|
| Labels | `bug`, `feature`, `performance`, `enhancement`, `design` |
| Stars | 2,000+ |
| Issue age | 7-90 days |
| Comments | < 8 |
| Scope | Multi-module, 100-500+ lines changed |

**Typical fixes:** Race conditions, memory leaks, API redesigns, query optimization, cross-platform compatibility.

**Optional:** Spawn a Council of LLMs to evaluate the approach before implementing.

## API-First Approach

For simple fixes (Level 1, some Level 2), **skip cloning entirely** and use the GitHub API directly:

```
1. Read files:   GET  /repos/{owner}/{repo}/contents/{path}
2. Write changes: PUT  /repos/{owner}/{repo}/contents/{path}
3. Create PR:     POST /repos/{owner}/{repo}/pulls
```

This completes in **2-5 minutes** instead of 15+.

| Approach | When to Use | Time |
|----------|-------------|------|
| **API-first** | 1-3 files, no test suite needed | 2-5 min |
| **Clone** | 4+ files, need local testing, complex structure | 10-20 min |

## Configuration

### Main Config (`contrib-scout/config.json`)

```json
{
  "difficulty_level": 2,
  "github_token_path": "~/.openclaw/.github_token",
  "max_contributions_per_night": 1,
  "languages": ["python", "javascript", "typescript", "go", "rust"],
  "excluded_repos": []
}
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `difficulty_level` | int | `2` | 1 (easy), 2 (intermediate), 3 (advanced) |
| `github_token_path` | string | `~/.openclaw/.github_token` | Path to your GitHub PAT |
| `max_contributions_per_night` | int | `1` | Max PRs per scheduled run |
| `languages` | array | 5 languages | Target languages for issue search |
| `excluded_repos` | array | `[]` | Repos to skip (already contributed) |

### GitHub Token

<details>
<summary>Token permissions and setup</summary>

**Required scope:** `public_repo` (read/write access to public repos)

**Do NOT grant:** `repo` (includes private repos), `delete_repo`, `admin:org`

**Setup options:**

```bash
# Option A: File on disk (default path)
echo "ghp_your_token_here" > ~/.openclaw/.github_token
chmod 600 ~/.openclaw/.github_token

# Option B: Environment variable
export GITHUB_TOKEN="ghp_your_token_here"

# Option C: Custom path in config.json
{
  "github_token_path": "/path/to/your/token"
}
```

**Security rules:**
- Never log or print the token value
- Never commit the token to git
- Use `public_repo` scope only — never `repo`
- The token is read silently by the agent at runtime

</details>

### Cron Configuration

The pipeline runs on a schedule (default: Mon/Wed/Fri at midnight). Configure via OpenClaw cron:

```json
{
  "schedule": {
    "kind": "cron",
    "expr": "0 0 * * 1,3,5",
    "tz": "Europe/Berlin"
  }
}
```

Adjust the cron expression for your preferred schedule and timezone.

## Quality Standards

- **Meaningful contributions only** — no typo fixes, no placeholder PRs
- **Must pass CI/tests** where available
- **Must reference the issue** it fixes
- **Must disclose AI assistance** in the PR description
- **No emojis** in PR titles, descriptions, or comments — professional tone throughout
- **Never force a contribution** — if no suitable issue is found, report that honestly

## Subagent Constraints

| Subagent CAN | Subagent CANNOT |
|-------------|-----------------|
| Read pasted file contents | Clone repos or run git commands |
| Return modified code or diffs | Push code or create PRs |
| Analyze issues and suggest fixes | Make GitHub API calls |
| Work within 600s timeout | Browse the web or install packages |
| | Access your GitHub token or credentials |

## What Changed in v3

| Feature | v2 (Old) | v3 (Current) |
|---------|----------|-------------|
| Architecture | Subagent does everything | Architect-Builder: main agent does I/O, subagent does cognition |
| Git operations | Subagent clones/pushes | Main agent handles all git |
| API-first | No | Yes — skip clone for simple fixes |
| Subagent timeout | 900-1200s | 600s max (focused work only) |
| Council models | Hardcoded per model | User-configurable via `council-config.json` |
| Difficulty levels | 4 levels with approval gates | 3 levels with clear criteria |
| Emojis in PRs | Allowed | Forbidden — professional tone only |
| Excluded repos | Hardcoded list | User-configurable in config |

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Subagent times out | Task too broad (clone + edit + push) | Use Architect-Builder pattern |
| Shallow code changes | Vague subagent prompt | Paste full file contents + specific instructions |
| Lint failures after edit | Subagent didn't know lint rules | Include lint config or run lint before subagent |
| PR rejected by maintainer | Low-quality contribution | Follow `CONTRIBUTING.md`, write meaningful fixes |
| Context overflow | Too much data pasted inline | Summarize to <1500 words, only relevant files |
| No suitable issues found | Filters too narrow | Broaden language or star requirements |

## Companion Skills

| Skill | Required | Purpose |
|-------|----------|---------|
| [Subagent Orchestration](https://github.com/wahajahmed010/subagent-orchestration) | Yes | Spawn patterns, timeout config, sandbox constraints |
| [Council of LLMs](https://github.com/wahajahmed010/council-of-llms) | No | Level 3 complex decisions before implementing |

## Requirements

- **OpenClaw** with `sessions_spawn` capability
- **Git** (2.30+)
- **Python** 3.10+
- **GitHub Personal Access Token** with `public_repo` scope

## License

[MIT-0](https://opensource.org/licenses/MIT-0) — Free to use, modify, and redistribute. No attribution required.

## Disclaimer

This tool makes commits under your GitHub identity. Review the quality standards before use. You are responsible for all contributions made under your account. Always review AI-generated code before it reaches maintainers.