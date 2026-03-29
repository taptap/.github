# .github

Org-wide shared workflows and configurations for **taptap**.

## Reusable Workflows

### Code Review

AI-powered pull request review using [Claude Code Action](https://github.com/anthropics/claude-code-action) and [GitHub Copilot](https://docs.github.com/en/copilot). It reviews every non-draft PR for code quality, correctness, and security, posting inline comments on specific lines.

#### Usage

Create `.github/workflows/code-review.yml` in your repo:

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  review:
    uses: taptap/.github/.github/workflows/code-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

#### Secrets

| Secret              | Required | Description             |
| ------------------- | -------- | ----------------------- |
| `ANTHROPIC_API_KEY` | Yes      | API key for Anthropic   |

Set this as an **org-level secret** so all repos inherit it, or per-repo if needed.

#### Variables

| Variable              | Required | Description                                     |
| --------------------- | -------- | ----------------------------------------------- |
| `ANTHROPIC_BASE_URL`  | No       | Custom API base URL (e.g. for proxy or gateway) |

Set this as an **org-level variable** (not a secret) if you need to route requests through a proxy.

#### Behavior

- **Triggers** on PR opened, synchronized (new push), or marked ready for review.
- **Skips** draft PRs and PRs from forks (security).
- **Concurrency** — only one review runs per PR at a time; new pushes cancel in-progress reviews.
- **Timeout** — 15 minutes per run.

#### Customization

Add a `REVIEW.md` file to your repo root with project-specific review guidelines. The reviewer will follow them automatically.
