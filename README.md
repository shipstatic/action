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
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: shipstatic/action@v1
        with:
          path: ./dist
```

Your site is live instantly on `*.shipstatic.com`. No API key, no sign-up, no configuration.

Deployments without an API key are public and expire in 3 days. The `claim` output contains a URL to keep the site permanently.

Add `github-token: ${{ secrets.GITHUB_TOKEN }}` to post the URL as a PR comment and track deployments in the repo sidebar.

## All Features — Free API Key

For permanent deployments and full control over your sites and domains, get a free API key from [my.shipstatic.com/api-key](https://my.shipstatic.com/api-key):

1. Get a free key at [my.shipstatic.com/api-key](https://my.shipstatic.com/api-key)
2. Add it as a repository secret named `SHIP_API_KEY` (Settings > Secrets and variables > Actions)

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
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: shipstatic/action@v1
        with:
          api-key: ${{ secrets.SHIP_API_KEY }}
          path: ./dist
          domain: www.example.com
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | No | — | ShipStatic API key for permanent deployments |
| `path` | No | `.` | Directory to deploy |
| `domain` | No | — | Domain to link (requires `api-key`) |
| `github-token` | No | — | GitHub token for PR comments and deployment tracking |

The `github-token` input enables two features:

- **PR comments** — posts the deployment URL as a comment on pull requests
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
| `id` | Deployment ID (e.g. `happy-cat-abc1234.shipstatic.com`) |
| `url` | Deployment URL (e.g. `https://happy-cat-abc1234.shipstatic.com`) |
| `claim` | Claim URL for public deployments (visit to keep permanently) |

## Examples

Four copy-pasteable workflows in the [action-example](https://github.com/shipstatic/action-example) repo:

- [`deploy-no-account.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-no-account.yml) — deploy on push, no account needed
- [`deploy-api-key.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-api-key.yml) — permanent deploy with API key
- [`deploy-domain.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/deploy-domain.yml) — permanent deploy with custom domain
- [`preview-pr.yml`](https://github.com/shipstatic/action-example/blob/main/.github/workflows/preview-pr.yml) — preview deploy on pull request

## License

MIT
