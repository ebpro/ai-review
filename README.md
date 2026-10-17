# ebpro/ai-review

Canonical **reusable AI code-review workflow** for the ebpro org.

Call [`.github/workflows/ai-review-reusable.yml`](.github/workflows/ai-review-reusable.yml)
from any repository — public or private — to get an automated review of every pull
request by 11 specialized [opencode](https://opencode.ai) agents (architecture,
security, performance, testing, docs, and more) running on self-hosted runners.

This repo lives here (public) because GitHub forbids a **public** consumer repo
from calling a **private** repo's reusable workflow. The runner image, agent
definitions, and their build pipeline remain in the private
[`ebpro/quarkus-ci-runner`](https://github.com/ebpro/quarkus-ci-runner) repo —
consumers never need them.

## Consumer quickstart

Add this file to your repo at `.github/workflows/ai-review.yml`:

```yaml
# AI Review — thin caller for the ebpro reusable AI review workflow.
name: AI Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    # Skip fork PRs (they cannot receive org secrets anyway).
    if: github.event.pull_request.head.repo.full_name == github.event.pull_request.base.repo.full_name
    uses: ebpro/ai-review/.github/workflows/ai-review-reusable.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
      repo: ${{ github.repository }}
      agents: all          # "all" | "code" | "lecture" | '["review-security",...]'
    secrets:
      LLM_REVIEW_ENDPOINT: ${{ secrets.LLM_REVIEW_ENDPOINT }}
      LLM_REVIEW_TOKEN: ${{ secrets.LLM_REVIEW_TOKEN }}
```

Why explicit named secrets instead of `secrets: inherit`: **least
privilege** — the review job receives only these two values, not every
repository/org secret an `inherit` would forward to the called workflow.
On ref pinning: `@main` is used here for teaching convenience; the hub is
org-owned. If you want stronger guarantees, pin a commit SHA
(`@<full-sha>`) and/or enable the org-level "require SHA pinning" setting.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `pr_number` | yes | — | PR number to review (`${{ github.event.pull_request.number }}`) |
| `repo` | yes | — | `owner/name` of the consuming repo (`${{ github.repository }}`) |
| `agents` | no | `all` | `all`, `code`, `lecture`, or a JSON array of agent names |

## Prerequisites

For the workflow to execute (not just parse), the consuming repo needs:

1. **Org secrets shared to it**: `LLM_REVIEW_ENDPOINT` and `LLM_REVIEW_TOKEN`
   (GitHub org/user secrets → *Secrets and variables* → *Actions* → repository
   access, or `dependabot`-style selected access). Set at the org level, the
   caller passes them through with an explicit named-secret mapping
   (`LLM_REVIEW_ENDPOINT: ${{ secrets.LLM_REVIEW_ENDPOINT }}`, etc.), which
   resolves org-level secrets in the caller context.
   - **Public repos need the secrets' visibility widened to public repos** —
     by default org secrets are visible only to private repositories.
2. **Runner access**: the workflow targets self-hosted runners labeled
   `ebpro-ai-review` (plus `linux`, `self-hosted`, `best-practices`). For
   organization runner groups / runner scale sets, grant the consuming repo
   access in the runner group settings.

If either is missing, jobs will queue forever or the LLM connectivity pre-check
will fail — everything else still works and each agent posts a clear fallback
comment rather than silently disappearing.

## What you get

Each selected agent posts one issue comment on the PR, identified by an
`<!-- ai-review:<agent> -->` marker (updated in place on later pushes). If an
agent errors (bad model config, API outage), a fallback comment records the
failure so the review is visibly incomplete instead of silently missing.

## Fork PRs

Fork pull requests are **skipped** by the `if:` guard above. GitHub never
exposes org/repo secrets to workflows from forks, and running third-party code
on your self-hosted runners is a well-known supply-chain risk. Internal-branch
PRs always run.

## Security posture

- This repository contains **no secrets, no internal URLs, no credentials**.
  It is 100% safe to read, and public by design.
- The review model, endpoint, and agent definitions live in the **private
  runner image** built by
  [`ebpro/quarkus-ci-runner`](https://github.com/ebpro/quarkus-ci-runner);
  nothing sensitive is evaluated from this repo's contents.
- Org secrets reach the runner only through the explicit named-secret
  passthrough above, on PRs from **same-repo branches** (fork PRs are
  skipped, see above).
- The reusable workflow only reads your PR diff and posts comments; it never
  writes to your repository.

## Related

- [`ebpro/quarkus-ci-runner`](https://github.com/ebpro/quarkus-ci-runner) — the
  private runner image + 11 opencode agents + the dogfooding caller for this hub.
