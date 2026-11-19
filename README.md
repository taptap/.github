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

Pass **one** of these. Both are declared optional so a repo can use whichever
name it already has; the job fails fast if neither is set.

| Secret                 | Required       | Description                                       |
| ---------------------- | -------------- | ------------------------------------------------- |
| `ANTHROPIC_API_KEY`    | one of the two | Sent as `x-api-key`                               |
| `ANTHROPIC_AUTH_TOKEN` | one of the two | Sent as a bearer token, for gateways that want it |

Set this as an **org-level secret** so all repos inherit it, or per-repo if needed.

`ANTHROPIC_API_KEY` wins when both are present. This is a fallback on _which
name exists_, not a retry when a credential is rejected — Actions cannot re-run
a failed step with the other key without re-running the whole review.

#### Inputs

Only needed when calling this as a reusable workflow (`workflow_call`).

| Input                    | Required | Default             | Description                                                       |
| ------------------------ | -------- | ------------------- | ----------------------------------------------------------------- |
| `pr_number`              | Yes      | —                   | PR to review (`github.event.pull_request` is empty here)          |
| `is_draft`               | No       | `false`             | Skip the review while the PR is a draft                           |
| `head_repo_full_name`    | No       | —                   | Head repo, for the same-repo (anti-fork) check                    |
| `request_copilot_review` | No       | `true`              | Set `false` to skip adding `@copilot` as a reviewer               |
| `model`                  | No       | `claude-opus-5[1m]` | Full model ID. Must be a model your gateway allows                |
| `sticky_summary`         | No       | `false`             | One sticky summary comment per PR, rewritten in place (see below) |

The default is a **pinned model ID rather than an alias** (`opus`): aliases are
resolved inside the bundled Claude Code build, so bumping that build would
silently change which model reviews your code.

Whatever you pass must exist on the endpoint behind `ANTHROPIC_BASE_URL`. A
model the gateway rejects surfaces as `403 ... Model is blocked` wrapped in
Claude Code's `Failed to authenticate` text — which reads like a bad key but is
not one.

#### Variables

| Variable             | Required | Description                                     |
| -------------------- | -------- | ----------------------------------------------- |
| `ANTHROPIC_BASE_URL` | No       | Custom API base URL (e.g. for proxy or gateway) |

Set this as an **org-level variable** (not a secret) if you need to route requests through a proxy.

#### Behavior

- **Triggers** on PR opened, synchronized (new push), or marked ready for review.
- **Skips** draft PRs and PRs from forks (security).
- **Concurrency** — only one review runs per PR at a time; new pushes cancel in-progress reviews.
- **Timeout** — 15 minutes per run.

#### Sticky summary (`sticky_summary: true`)

By default every run posts a **new** summary comment, so a PR pushed N times
collects N summaries. With `sticky_summary: true`:

- **One top-level comment per PR.** The action creates it on the first run and
  reuses it afterwards (`track_progress` + `use_sticky_comment`); the model
  only rewrites its body to describe the current head commit.
- **No other top-level posts.** `gh pr comment`, `gh pr review`, issue-comment
  and review-creation endpoints are denied at the permission layer. Findings
  still go to inline threads, which are replied to and resolved once fixed.
- **Quieter threads.** The thread query goes through GraphQL (not a single
  30-item REST page) and includes human replies: a thread a human already
  confirmed is resolved without an extra reply, and "still unresolved" is only
  re-posted when the related code changed or a human replied since the last
  comment.
- **Stale-head guard.** Before writing, the reviewer checks the PR head still
  matches the commit it reviewed, and stops otherwise.
- **Read-only.** This mode runs the action in tag mode, which by itself would
  allow edits and `git commit`/push; those tools are denied explicitly.

Requires a `pull_request` trigger in the caller (`track_progress` only supports
`pull_request` opened / synchronize / ready_for_review / reopened). Upstream
caveats: the sticky lookup matches the _first_ claude[bot] top-level comment on
the PR (an earlier `@claude` answer from another workflow would be reused and
overwritten) and only scans the first ~30 top-level comments.

#### Customization

Add a `REVIEW.md` file to your repo root with project-specific review guidelines. The reviewer will follow them automatically.
