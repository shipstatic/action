# shipstatic/action

GitHub Action for [ShipStatic](https://shipstatic.com) — deploy static websites, landing pages, and prototypes instantly from CI.

## Deploy — Free, No Account Needed

```yaml
name: Deploy
on: push

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: npm ci && npm run build
      - uses: shipstatic/action@v2
        with:
          path: ./dist
```

Your site is live instantly on `*.shipstatic.com`. No token, no sign-up, no configuration.

Deployments made without a token are public and expire — the `claim` output holds a URL that keeps the site permanently, and `expires` holds the deadline.

Add `github-token: ${{ secrets.GITHUB_TOKEN }}` to post the URL as a PR comment and track deployments in the repo sidebar.

## All Features — Free API Key

For permanent deployments and full control over your sites and domains, get a free API key from [my.shipstatic.com/api-key](https://my.shipstatic.com/api-key):

1. Get a free key at [my.shipstatic.com/api-key](https://my.shipstatic.com/api-key)
2. Add it as a repository secret named `SHIP_TOKEN` (Settings > Secrets and variables > Actions)

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  deployments: write
  pull-requests: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: npm ci && npm run build
      - uses: shipstatic/action@v2
        with:
          token: ${{ secrets.SHIP_TOKEN }}
          path: ./dist
          domain: www.example.com
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### One slot, two kinds of token

`token` takes either credential ShipStatic issues:

| Value | What it is |
|-------|------------|
| `ship-…` | An **API key** — durable, full account access |
| `deploy-…` | A **deploy token** — scoped to deploys, optionally time-limited, revocable |

A deploy-only workflow can run on a deploy token. This action never reads your account, so nothing here needs the wider credential — create one at [my.shipstatic.com](https://my.shipstatic.com) and keep the API key out of CI.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `token` | No | — | API key (`ship-…`) or deploy token (`deploy-…`). Omit to deploy anonymously |
| `api-url` | No | production API | ShipStatic API endpoint |
| `path` | No | `.` | Directory to deploy |
| `domain` | No | — | Domain to link (requires `token`) |
| `password` | No | — | Password-protect the deployment (6–128 characters). Visitors are prompted to unlock before viewing |
| `labels` | No | — | Comma-separated labels, added to the automatic commit label |
| `idempotency-key` | No | *derived* | Override the replay key (see below) |
| `github-token` | No | — | GitHub token for PR comments and deployment tracking |

The `github-token` input enables two features:

- **PR comments** — posts the deployment URL as a single comment, updated in place on every push
- **GitHub Deployments** — creates deployment objects visible in the repo sidebar

Use the automatic `${{ secrets.GITHUB_TOKEN }}` — no extra secrets needed. Your workflow needs these permissions:

```yaml
permissions:
  contents: read
  deployments: write
  pull-requests: write
```

## Outputs

| Output | Description |
|--------|-------------|
| `deployment` | Deployment hostname (e.g. `happy-cat-abc1234.shipstatic.com`) |
| `url` | Deployment URL (e.g. `https://happy-cat-abc1234.shipstatic.com`) |
| `claim` | Claim URL — anonymous deployments only (visit to keep permanently) |
| `expires` | Expiry as a unix timestamp in seconds — anonymous deployments only |

```yaml
      - uses: shipstatic/action@v2
        id: deploy
        with:
          path: ./dist
      - run: echo "Live at ${{ steps.deploy.outputs.url }}"
```

Every deploy also writes a summary table to the workflow run page — the deployment, its URL, the domain when one is linked, and for anonymous deploys the claim link and expiry.

## Labels

Every deployment is labelled with the commit's short SHA automatically. `labels` adds your own on top:

```yaml
      - uses: shipstatic/action@v2
        with:
          token: ${{ secrets.SHIP_TOKEN }}
          path: ./dist
          labels: preview,pr-${{ github.event.number }}
```

Labels are 3–25 characters, lowercase alphanumeric with `.`, `_` or `-` between segments, up to 10 per deployment. Find them again with `ship deployments list`.

## Re-running a job does not deploy twice

Every deploy carries an idempotency key derived from the workflow run:

```
<owner>/<repo>-<run id>-<job id>
```

A GitHub run id is stable across re-runs and fresh on every new push, so pressing **Re-run jobs** replays the deployment the first attempt created — same URL, no second upload, no second deployment in your account — while the next push deploys normally. Nothing to configure.

Two cases need an explicit key, because no ambient value distinguishes them:

- **Matrix legs deploying different content.** All legs of a matrix share one job id.
- **Two deploys in one job.** Both invocations share the run and the job.

Give each its own key:

```yaml
    strategy:
      matrix:
        site: [docs, marketing]
    steps:
      - uses: shipstatic/action@v2
        with:
          path: ./dist/${{ matrix.site }}
          idempotency-key: ${{ github.run_id }}-${{ matrix.site }}
```

## Deploying to a different endpoint

`api-url` points the action at another ShipStatic API. Omit it and the production API is used.

```yaml
      - uses: shipstatic/action@v2
        with:
          token: ${{ secrets.SHIP_TOKEN }}
          api-url: ${{ vars.SHIP_API_URL }}
          path: ./dist
```

## Requirements

GitHub-hosted runners need nothing. Self-hosted runners need **Node.js 20 or newer** and `jq` on `PATH`; the action installs the ShipStatic CLI itself.

## Versioning

`@v2` is a moving tag that always points at the latest 2.x release, and it speaks the ShipStatic **2.x** platform — the action's major and the platform major it targets are the same number, by design. Pin `@v2.0.0` for an immutable reference.

`@v1` is frozen and speaks the 1.x platform. It receives no changes.

## Things this action deliberately does not have

- **No `cli-version` input.** The action's major names the CLI major it speaks; letting a workflow float across that boundary is the failure this design exists to prevent.
- **No SPA / path-detection toggles.** The CLI's defaults are right, and a repo that needs different behaviour ships a [`ship.json`](https://docs.shipstatic.com), which the detection defers to.
- **No `setup-node` step.** Hosted runners already carry a supported Node; installing another would slow every workflow for the exception's sake.
- **No timeout inputs.** The CLI owns its own budgets, sized to the platform's upload limits.

## Examples

Five copy-pasteable workflows in the [action-example](https://github.com/shipstatic/action-example) repo:

- [`deploy-no-account.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-no-account.yml) — deploy on push, no account needed
- [`deploy-api-key.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-api-key.yml) — permanent deploy with a token
- [`deploy-domain.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-domain.yml) — permanent deploy with a custom domain
- [`preview-pr.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/preview-pr.yml) — preview deploy on pull request
- [`deploy-password.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-password.yml) — password-protected deploy (any tier)

## License

MIT
