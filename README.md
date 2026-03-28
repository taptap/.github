# .github

Org-wide shared workflows and configurations for **taptap**.

## Reusable Workflows

### Claude Code Review

AI-powered pull request review using [Claude Code Action](https://github.com/anthropics/claude-code-action). It reviews every non-draft PR for code quality, correctness, and security, posting inline comments on specific lines.

#### Usage

Create `.github/workflows/claude-code-review.yml` in your repo:

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    uses: taptap/.github/.github/workflows/claude-code-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      # ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}  # optional
```

#### Secrets

| Secret               | Required | Description                                     |
| -------------------- | -------- | ----------------------------------------------- |
| `ANTHROPIC_API_KEY`  | Yes      | API key for Anthropic                           |
| `ANTHROPIC_BASE_URL` | No       | Custom API base URL (e.g. for proxy or gateway) |

Set these as **org-level secrets** so all repos inherit them, or per-repo if needed.

#### Behavior

- **Triggers** on PR opened, synchronized (new push), or marked ready for review.
- **Skips** draft PRs and PRs from forks (security).
- **Concurrency** — only one review runs per PR at a time; new pushes cancel in-progress reviews.
- **Timeout** — 15 minutes per run.

#### Customization

Add a `REVIEW.md` file to your repo root with project-specific review guidelines. The reviewer will follow them automatically.
