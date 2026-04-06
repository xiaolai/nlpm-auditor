# nlpm-auditor

Automated pipeline for discovering, auditing, and contributing to Claude Code plugin/skill repos across GitHub.

## Architecture

```
discover.yml (weekly cron)
  → searches GitHub API for Claude Code plugin/skill repos
  → updates registry/repos.json with new finds
  → creates "audit-candidate" issues for repos meeting threshold

audit.yml (triggered by issue label "audit-ready")
  → claude-code-action clones the target repo
  → runs NLPM scoring (50 rules, 100-point scale)
  → writes audit report to audits/{owner}/{repo}.md
  → comments findings on the issue
  → labels issue "audit-complete" or "audit-clean"

contribute.yml (triggered by issue label "contribute-approved")
  → claude-code-action reads the audit report
  → forks the target repo
  → creates focused PRs for verified bugs only
  → creates a tracking issue in the target repo
  → updates registry with PR links
  → labels issue "prs-submitted"

track.yml (weekly cron)
  → checks status of all submitted PRs
  → updates registry with merge/close status
  → promotes accepted engagements to case-studies/
```

## Registry Schema

`registry/repos.json`:
```json
{
  "repos": {
    "owner/name": {
      "discovered": "2026-04-05T00:00:00Z",
      "stars": 48000,
      "artifacts": 80,
      "score": 74,
      "status": "discovered|audited|contributed|tracked",
      "audit_issue": 12,
      "prs": [{"repo": "owner/name", "number": 1573, "status": "merged"}],
      "case_study": "case-studies/owner-name.md"
    }
  }
}
```

## Filtering Rules

Not every popular repo deserves a house call. A repo is audit-worthy when ALL of these are true:
- Has `.claude-plugin/plugin.json` OR `agents/*.md` OR `commands/*.md` OR `skills/**/SKILL.md`
- 500+ stars
- Updated in last 60 days
- Not archived
- Has CONTRIBUTING.md or doesn't explicitly block contributions
- 10+ NL artifacts
- Not already in registry with status "audited" or later

## PR Submission Rules

Only submit PRs for verified bugs — NOT convention preferences:
- Missing required fields that prevent registration (missing `name:`)
- Broken cross-references (skill/partial paths that don't resolve)
- `allowed-tools` missing tools the body calls (guaranteed runtime failure)
- Missing `allowed-tools` entirely (least-privilege violation)
- Scripts referenced by hooks that don't exist

Do NOT submit PRs for:
- YAML format preferences (scalar string vs list for tools:)
- Missing `<example>` blocks
- Vague quantifiers
- Output format suggestions
- Model tier recommendations

## Etiquette

Think of it like visiting someone's home with a toolbox — you were invited by the open-source license, but you're still a guest.

1. One tracking issue first — always. Knock before entering.
2. Max 5 PRs per repo — fix the leaking tap, don't remodel the kitchen.
3. Wait for maintainer response before submitting PRs. Patience is politeness.
4. Accept "no" gracefully — their house, their rules.
5. Credit the project — the audit validates NLPM, not the other way around.
6. Never submit to more than 2 repos in the same week. Even a welcome guest can overstay.

## Build & Run

Requires:
- `CLAUDE_CODE_OAUTH_TOKEN` secret for claude-code-action
- `GH_TOKEN` or `GITHUB_TOKEN` with repo + public_repo scope
- nlpm plugin installed in the action environment

## Commands

No slash commands. Everything is driven by GitHub Actions + issue labels. The human steers; the pipeline rows.

## Testing

Manual dispatch of discover.yml with `dry_run: true` to verify search results without creating issues.
