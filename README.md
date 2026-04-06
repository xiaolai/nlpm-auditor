# nlpm-auditor

Automated pipeline for discovering, auditing, and contributing to Claude Code plugin and skill repos across GitHub.

Uses [NLPM](https://github.com/xiaolai/nlpm-for-claude) scoring (50 rules, 100-point scale) and [claude-code-action](https://github.com/anthropics/claude-code-action) for automated analysis.

## How It Works

```mermaid
graph LR
    subgraph "Weekly Cron"
        D[Discover] -->|creates issues| R[(Registry)]
    end

    subgraph "Human Review"
        R --> H{Worth auditing?}
        H -->|label: audit-ready| A[Audit]
        H -->|close| X[Skip]
    end

    subgraph "Claude Code Action"
        A -->|scores artifacts| Report[Audit Report]
        Report --> H2{Has real bugs?}
        H2 -->|label: contribute-approved| C[Contribute PRs]
        H2 -->|close| X2[No action]
    end

    subgraph "Completion"
        C -->|submits PRs| T[Track]
        T -->|PRs merged| CS[Case Study]
        CS -->|auto-generated| Done[Complete]
    end
```

## Pipeline

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `discover.yml` | Weekly cron / manual | Searches GitHub for Claude Code repos with 500+ stars and 5+ NL artifacts |
| `audit.yml` | Issue labeled `audit-ready` | Clones repo, runs NLPM scoring via claude-code-action, writes audit report |
| `contribute.yml` | Issue labeled `contribute-approved` | Forks repo, creates PRs for verified bugs only (max 5) |
| `track.yml` | Weekly cron | Checks PR status, marks case study candidates when PRs merge |
| `case-study.yml` | Issue labeled `case-study-ready` | Gathers GitHub evidence, writes article via claude-code-action, generates cover image via DALL-E, commits and closes |

## Issue Label Lifecycle

```mermaid
stateDiagram-v2
    [*] --> audit_candidate : discover.yml
    audit_candidate --> audit_ready : human approves
    audit_candidate --> [*] : human closes

    audit_ready --> audit_complete : audit.yml scores
    audit_complete --> contribute_approved : human approves PRs
    audit_complete --> [*] : no real bugs

    contribute_approved --> prs_submitted : contribute.yml
    prs_submitted --> case_study_ready : track.yml detects merges
    prs_submitted --> [*] : all PRs rejected

    case_study_ready --> complete : case-study.yml writes article
    complete --> [*] : issue closed
```

| Label | Meaning |
|-------|---------|
| `audit-candidate` | Discovered by crawler, awaiting human review |
| `audit-ready` | Approved for audit — triggers `audit.yml` |
| `audit-complete` | Audit report generated |
| `contribute-approved` | Human approved PR submission — triggers `contribute.yml` |
| `prs-submitted` | PRs have been submitted to the target repo |
| `case-study-ready` | PRs merged — triggers `case-study.yml` |
| `complete` | Case study published, issue closed |

## Case Study Generation

When the `case-study-ready` label is applied, `case-study.yml`:

1. **Gathers evidence** from GitHub API:
   - Repo metadata (stars, description, owner)
   - All PRs we submitted (states, timestamps, URLs)
   - Tracking issues
   - Commits mentioning NLPM or Claude co-authorship
   - Maintainer review comments on each PR
   - The original audit report

2. **Writes the article** via claude-code-action following a proven template:
   - Disclosure, project context, audit results (with mermaid pie chart)
   - PRs submitted, maintainer response, patterns observed
   - Timeline (mermaid gantt chart), limitations, significance

3. **Generates a cover image** via OpenAI DALL-E API

4. **Commits** article + image to `case-studies/`, updates registry, closes the issue

## Rules of Engagement

1. **Only submit PRs for verified bugs** — missing fields that break registration, tools called but not in allowed-tools, broken references
2. **Never PR convention preferences** — YAML format, missing examples, vague language, model tier
3. **One tracking issue first** — explain methodology before PRs
4. **Max 5 PRs per repo** — focused, minimal diffs
5. **Max 2 repos per week** — don't carpet-bomb
6. **Accept "no" gracefully** — close PRs, thank, learn

## Setup

1. Create the repo on GitHub
2. Add secrets:
   - `CLAUDE_CODE_OAUTH_TOKEN` — for claude-code-action
   - `PAT_TOKEN` — GitHub PAT with `public_repo` scope (for forking and PRing to other repos)
   - `OPENAI_API_KEY` — for DALL-E cover image generation
3. Create the issue labels listed above
4. Run `discover.yml` manually to seed the registry

## Directory Structure

```
registry/repos.json    — Tracking database (all discovered repos + status)
audits/                — Generated audit reports (one per repo)
case-studies/          — Published case studies with cover images
```

## Prerequisites

- GitHub Actions enabled
- `CLAUDE_CODE_OAUTH_TOKEN` secret
- `PAT_TOKEN` secret with `public_repo` scope
- `OPENAI_API_KEY` secret (for cover images)
