# CLAUDE.md

Claude Code instructions for **`shipstatic/action`** — the GitHub Action.

## Identity

A **composite** action: `action.yml` plus docs, plus a fixture and a workflow
that prove them. No build, no manifest, no runtime code. It installs the
`@shipstatic/ship` CLI on the runner and invokes it twice — deploy, then
optionally link a domain — and wraps that in two GitHub-side conveniences
(a deployment record, a sticky PR comment).

**v2 speaks the 2.x platform.** The action's major and the platform major it
targets are the same number from here on, which is the versioning law below.

## The law: majors pin majors

An action whose interface is a CONTRACT WITH A CLI must name the CLI major it
speaks. `@v1` speaks ship 1.x; `@v2` speaks ship 2.x; the install line says so
in bytes.

This is not tidiness. The action is the ONE consumer on the platform that
resolves the CLI at **runtime**, so whatever the registry serves that minute
runs inside every existing consumer's workflow. `action.yml` installed
`@shipstatic/ship@latest` until v2, and across the 1.x → 2.x boundary that
line has the worst failure shape the platform can produce: the 2.x CLI does
not read `SHIP_API_KEY`, ship's anonymity is **fail-closed** (proven absence
of a credential IS anonymity), so an authenticated deploy would have succeeded
with a 201 into the PUBLIC account — claim URL, three-day expiry, and no error
anywhere. A consumer that also links a domain fails one step later; a consumer
that does not gets no signal at all.

`.github/workflows/ci.yml` holds the law mechanically: the install line must
name a ship major, and that major must be `2`.

### The pin

The install line reads `@shipstatic/ship@2` and freezes there for the life of
this major — **and that line is this repo's constellation pin**, since it has
no manifest. A range is the right shape precisely because it is a major: every
2.x fix reaches every `@v2` consumer with no release here, which is the whole
argument for staying composite.

The fence checks the MAJOR rather than the literal, which is what let the line
carry an exact `2.0.0-beta.N` while no 2.x stable existed on the registry —
npm ranges never match prereleases, so `@2` would have resolved nothing.

A version is not an environment value, so either form is lawful in a public
tree.

## Releases

The release IS the annotated tag; the channel IS the major tag. That is the
whole translation of the platform's npm publish law into a repo with no
registry, no manifest and no version field — see root `CLAUDE.md`, "The npm
publish law", for the mechanisms this one is the smallest instance of.

- `vX.Y.Z` tags are immutable.
- `v1`, `v2` are the moving tags consumers bind. Fast-forward them
  deliberately; **never force-push one backwards** — that rewrites what is
  already running in other people's CI.
- Betas are `vX.Y.Z-beta.N` tags that no major tag points at.
- The gate before any tag is `ci.yml` green **on the tagged commit**.

There is deliberately **no release workflow**. A two-file repo does not earn
one, and the ritual is two commands:

```bash
git tag -a v2.0.1 -m "v2.0.1" && git push origin v2.0.1
git tag -f -a v2  -m "v2 → v2.0.1" && git push -f origin v2
```

### v1 is maintained by never touching it again

`v1.2.1` pinned v1's install line to `@shipstatic/ship@1`, and `v1` points at
it (`fa11a3f`). That release was a **precondition** of the ship `latest` flip
rather than a consequence of it: `v1.2.0` still installed
`@shipstatic/ship@latest`, so the moment `2.0.0` reached `latest` every `@v1`
consumer would have started deploying into the public account, silently. It
published no behaviour — `@1` resolved the same artifact `latest` did that day.

The clock behind it was a demonstrated hazard, not a hypothesis: ship
`2.0.0-beta.16` sat on the registry with a broken artifact for a window of
minutes (2026-08). A floating install line's exposure is a proven class.

`@v1` is now frozen by its own pin and needs no further attention — and must
receive none. `v1` is a moving tag resolved by other people's CI, so it is
never force-pushed backwards. The cost that pin freezes is accepted rather
than overlooked: keyless `@v1` consumers stay broken against the 2.x API until
they move to v2 — the platform's recorded pre-launch posture, chosen over
teaching v1 the 2.x wire.

## The v2 surface

Every absence below is a decision; see "Recorded absences".

| Input | Default | Behaviour |
|---|---|---|
| `token` | — | One slot, any platform token (`ship-` key or `deploy-` token) → `SHIP_TOKEN`. Absent = anonymous deploy |
| `api-url` | *the CLI's* | → `SHIP_API_URL`. Empty is absence, so the CLI's own default (production) applies |
| `path` | `.` | Directory to deploy |
| `domain` | — | Linked after deploy (requires `token`) |
| `password` | — | → `SHIP_PASSWORD` (the empty-unset guard stays) |
| `labels` | — | Comma-separated → repeated `--label`, appended after the automatic short SHA |
| `idempotency-key` | *derived* | Override; else `<repo>-<run id>-<job id>` → `SHIP_IDEMPOTENCY_KEY` |
| `github-token` | — | Enables the deployment record + the sticky PR comment |
| `environment` | *derived* | The record's environment name; read ONLY by the record step, so it does nothing without `github-token` |

| Output | Source |
|---|---|
| `deployment` | `jq -r '.deployment'` — the canonical key |
| `url` | `jq -r '.url'` — the wire's own, never reconstructed |
| `claim` | `jq -r '.claim // empty'` |
| `expires` | `jq -r '.expires // empty'` — unix seconds; feeds the comment's date |

Automatic, no knob: `SHIP_VIA=git` · the commit short-SHA label · the derived
idempotency key · the run summary · SPA detection and junk filtering (the CLI's
defaults; `ship.json` is the override).

### The run summary belongs to the action

`$GITHUB_STEP_SUMMARY` is the first-party furniture of the era
(`actions/deploy-pages`, `attest-build-provenance`), and a deploy action that
says nothing on the run page makes every serious consumer write its own table —
which is precisely what `web/my` had done. So the action writes it: deployment,
URL, the domain when one was asked for, and for an anonymous deploy the claim
link and the rendered expiry. **No opt-out knob** — a summary is inert, and
nothing downstream can depend on it.

Two mechanical facts shape where it lives:

- **Inside the Deploy step, not a sixth step.** Each step gets its own summary
  FILE which the job concatenates for display, so a table split across two steps
  does not render as one table. And the five-step shape is fenced, which makes a
  step a design change rather than an edit.
- **The `Domain` row is the INPUT**, the one row not read off the wire — the
  link happens one step later. A failed link fails the job loudly directly
  beneath the summary, which is the honest place for it.

The expiry is rendered **once**, in the Deploy step where the timestamp is read,
and exposed as an internal `expires-at` step output the PR comment consumes. Two
copies of a `date -u -d @…` incantation in one file is the same drift class this
repo removed from `url` and the hardcoded "3 days"; the step output is not part
of the published surface (`outputs:` is unchanged).

### Everything is read off the wire

v1 reconstructed `url` as `"https://" + .deployment` and hardcoded
`Expires in 3 days` in the PR comment. Both are gone, and the reason is the
same in both cases: **this file has no compiler and no test runner, so the
only facts it can state truthfully forever are the ones it reads off the
response.** The wire carries `url`; it carries `expires`, so the comment
renders a real date, which is better product besides — a relative duration is
stale the moment it is read.

Two restatements remain, both prose, both recorded rather than missed: the
`my.shipstatic.com/api-key` link (owned by `MY_API_KEY_URL` in
`@shipstatic/types`) and the password range `6–128` in the input description
and README (owned by `PASSWORD_CONSTRAINTS` — the CLI validates with the real
rule and its message relays, so the prose can only ever be documentation of a
boundary someone else enforces). Bash and Markdown cannot import, and the
Constellation Law's stopping rule tolerates a forced restatement in prose
that teaches a value.

A third restatement is decided the same way but fenced differently: the yaml
necessarily copies the env-var names (`SHIP_API_URL`, `SHIP_TOKEN`) whose
vocabulary `SHIP_ENV` in `@shipstatic/types` owns. A yaml surface cannot
import, and the drift is LOUD — a wrong name makes the authed e2e leg deploy
anonymously and fail its inverse assertion — so per the stopping rule the
copies are held by the CI fence, not a copy-comparison.

### `api-url` has no default in `action.yml`, deliberately

The production endpoint is `DEFAULT_API` in `@shipstatic/types`. Writing it
here would be a second copy of an owned fact in a file that cannot import it,
so the input has no `default:` — an empty `SHIP_API_URL` is normalized to
absence by the SDK, and the CLI's own default applies. **The action does not
decide the endpoint**, which means it can never disagree about it.

This is also the same reasoning that keeps the action composite (below):
tracking the platform is this product's whole job.

### The idempotency key

`<owner>/<repo>-<run id>-<job id>`, derived in step bash because an
`action.yml` `default:` is a static string and cannot hold an expression.

`GITHUB_RUN_ID` is stable across re-runs and fresh per push, so **Re-run
jobs** replays the original 201 rather than deploying twice, while the next
push deploys normally. Each segment is load-bearing:

- **job** — two jobs in one run may deploy two different sites; one key would
  replay the first site's 201 into the second job.
- **repository** — GitHub guarantees a run id unique *within a repository*
  only, and the API's replay namespace for an ANONYMOUS deploy is keyed by the
  caller's egress IP, which every repository on a hosted runner shares. Without
  this segment two repositories could collide, and the collision would hand
  one of them the other's deployment URL.

Two axes have no ambient discriminator and take the override input: matrix
legs deploying different content, and two invocations inside one job. Both are
documented in the README with an example.

`SHIP_IDEMPOTENCY_KEY` is the mechanism — a CLI-only env var on the
`SHIP_PASSWORD` tier, added to `@shipstatic/ship` for exactly this consumer
(`npm/ship/CLAUDE.md`, "CLI-only env vars"). Not the `--idempotency-key` flag,
which remains ship's own open product call for shell users.

### The sticky PR comment

One comment per PR, updated in place, found by a hidden marker
(`<!-- shipstatic/action -->`) rather than by `gh pr comment --edit-last`.
`--edit-last` edits the authenticated identity's last comment *whatever wrote
it*, so on a PR where any other workflow also comments as
`github-actions[bot]` the two clobber each other.

## The fence

`.github/workflows/ci.yml`, tests-only on both branches:

| Job | What it holds |
|---|---|
| `lint` | actionlint over `.github/workflows/**`; the install line names ship major 2; the action is five shell steps |
| `e2e-anonymous` | Deploys `fixture/` keyless to dev; asserts all four outputs, that `url` addresses `deployment`, that claim + expires are both present, that the URL serves 200 — and that a SECOND same-job invocation REPLAYS the first deployment rather than creating one (the derived key's documented same-job collapse, used as the fence for the silent class: an unread `SHIP_IDEMPOTENCY_KEY` would keep succeeding and simply duplicate) |
| `e2e-authenticated` | Prunes earlier fence rows, then deploys with a token; asserts the deployment is **not** claimable or expiring, and reads `via` and both labels back from the API |

Four things about it are deliberate:

- **The authed leg's assertion is the INVERSE of the anonymous one.** "The
  deploy succeeded" proves nothing here: fail-closed anonymity means a
  credential the CLI does not read still produces a clean 201. Only *no claim,
  no expiry* proves the token was read.
- **The authed leg prunes at the START, never at the end.** It adds one owned
  row per push, owned deployments never expire, and the plan cap counts every
  row whatever its status — so the leg cleans up after itself or goes red on a
  schedule rather than on a defect. Deleting its *own* row at the end would
  break the property the derived idempotency key exists to provide: a re-run
  carries the same key, so the API replays the original 201, which would then
  name a row the previous attempt had deleted, and the read-back would 404.
  Pruning at the start only ever touches earlier runs' rows, whose keys are
  already stale. The selector is the `ci-fence` label the leg already deploys
  with — load-bearing rather than convenient, since the CI account is the
  estate's shared dev e2e identity and only rows this fence made may be
  reaped — floored at one hour so a concurrent run cannot reap a sibling's row
  mid-verification. It is the repo's one destructive CI step, which is why it
  sits *after* the environment guard: with no `SHIP_API_URL` it would prune
  production. Running before the action also means the CLI is not on the runner
  yet, so the step installs it — **deriving the pin from `action.yml` with the
  same grep the `lint` job uses**, never restating `@2`, because a second copy
  of the pin in this file is the one drift the ship-major fence cannot see.
  Every part of it is best-effort, because a prune that cannot run must not
  redden a run that otherwise proves the action works — a real outage fails
  loudly one step later, on the deploy.
- **The jobs refuse an unconfigured environment.** With no `SHIP_API_URL` the
  CLI would fall back to production, and `development` CI would deploy junk
  into the live public account on every push.
- **actionlint does not read `action.yml`** — only workflows. It fences this
  file, not the product. A bespoke extractor to shellcheck the composite's
  `run:` blocks was considered and declined: one that silently found zero
  blocks would be a green fence proving nothing, and guarding against that
  turns a 6-line step into a tool. The five-step count stands in for it — the
  moment someone must change that number is the moment they reread the shell.

Both e2e jobs are skipped on FORK pull requests, by a job-level `if:` — the
only place that can do it. GitHub withholds environment secrets *and*
variables from a fork, so an empty `SHIP_API_URL` there is the platform
working as designed, and the guard step cannot tell that apart from a
misconfigured repo. A red e2e on a community PR would be a lie; a skipped job
says what happened, and a maintainer re-runs the change from a branch.

**USER setup, once:** a `development` GitHub Environment on
`shipstatic/action` carrying a `SHIP_API_URL` variable (the dev API) and,
optionally, a `SHIP_TOKEN` secret. Without the secret the authed leg skips
with a `::notice`; without the variable both e2e jobs fail loudly by design.
**Add no protection rules** — required reviewers or a wait timer would block
every push and every PR, since this environment gates CI rather than a
release.

Two things about that secret are forced rather than chosen. It is a **`ship-`
API key, not a `deploy-` token**: deploy tokens carry
`permissions: { deploy: ['create'] }` and authenticate only on `deployScope`
routes, which the leg's `ship deployments get` read-back is not. And it
belongs to a **dedicated dev CI account**, never the operator's — a public
repo's CI must not hold a human's admin credential, and `PUT /account/key` is
an upsert that sweeps the account's existing key, so minting one for CI on a
shared account would revoke the operator's. That account's deployment cap
carries an override for the fence's burst, the same mechanism the dev PUBLIC
account uses for the anonymous leg.

## Tooling standard

Applied as far as a repo with no manifest can take it, with the gaps recorded
rather than faked:

| Present | Absent, and why |
|---|---|
| This file, `AGENTS.md` | Biome — nothing here it parses |
| Pre-commit credential scan (`scripts/githooks/pre-commit`) | `packageManager` / Node pin — no manifest |
| actionlint in CI | Coverage — no runtime to measure |
| A tests-only `ci.yml` on both branches, Slack failure notify (guarded on the webhook secret, the family shape) | |
| `renovate.json` (org preset — `ci.yml`'s pinned actions, actionlint included, are exactly what it updates elsewhere) | |

The hook has no `prepare` script to install it. Run once per clone:

```bash
git config core.hooksPath scripts/githooks
```

The repo is **PUBLIC**: the public-value law applies to every tracked byte.
Production is the only public value; the dev API URL arrives from the
`development` GitHub Environment, never from a file.

## What must not change

- **Composite, never a bundled JS action** — and this is derived, not
  asserted. Bundling the SDK (the vscode extension's pattern) would freeze the
  platform client per action release, so every CLI fix would need a release
  here to reach consumers. Install-then-invoke under majors-pin-majors delivers
  every 2.x fix to every `@v2` consumer with zero releases, and **tracking the
  platform is this product's whole job** — the extension bundles because an
  editor owns its release train; this action's train IS the platform's. A
  bundled action would also be a third full-platform-bar surface (suite,
  fences, coverage) maintained to express two CLI invocations. Costs named and
  accepted: ~5–10s of install per run, and npm availability coupling.
- **`SHIP_VIA=git`** — the action IS the `git` surface, whatever invoked the
  workflow. It is the recorded reason that member exists in `DeploymentVia`.
- **The automatic short-SHA label** — ambient attempt metadata, the same
  gesture as `SHIP_VIA` and the derived key.
- **The `SHIP_PASSWORD` empty-unset guard** — defense in depth for whatever
  CLI the pin resolves, even though the 2.x CLI normalizes empty to absence
  itself.
- **The five-step shape** and `continue-on-error` on the two GitHub-side
  conveniences: a failed comment must never fail a deploy that succeeded.
- **Anonymous deploy as a first-class path** — the keyless quickstart is a
  product demo, not an accident.
- **Output names within a major.** v1's `id`/`url`/`claim` are its published
  API and freeze with it; v2's `deployment`/`url`/`claim`/`expires` freeze from
  `v2.0.0` on.

## Recorded absences

- **No `cli-version` input.** Majors-pin-majors IS the contract; an override
  would let a consumer float, which is the failure above wearing a knob.
- **No `spa-detect` / `path-detect` toggles.** The SDK's defaults are right,
  and a repo needing different behaviour ships a `ship.json`, which the
  detection already defers to. A toggle would be a second way to say what the
  config file owns.
- **No `setup-node` step.** Hosted runners carry Node ≥20; the README states
  the requirement for self-hosted ones. Installing our own would slow every
  consumer for the exception's benefit.
- **No domain-link output.** The deployment is the product; `domain` is
  routing for it, and the linked URL is `https://<domain>` by construction. An
  output would restate an input.
- **No `via` knob** (law) and **no timeout inputs** (the CLI owns its budgets).

## Consumers

`integrations/action-example` — five workflows, all on `@v2` since
2026-08-11 and all green against production, including
`deploy-no-account.yml`, whose keyless path only works against a 2.x API. Its
domain is owned by a dedicated `ci@shipstatic.com` account: it previously
belonged to `public@shipstatic.com`, and since the public identity can hold no
API key, **no credential could have linked it** — a domain is linkable only by
the account that owns it.

`web/my`, `web/www` and `web/docs` all ship themselves through `@v2` since
2026-08-11 — a push to `development` deploys the dev site, and **the merge to
`main` IS the production deploy**, which is why every such merge is
operator-timed. Their domains belong to the dedicated `sites@shipstatic.{dev,com}`
accounts, since a domain is linkable only by its owner.

## Post-launch: OIDC trusted deploying

**The 2026 endgame for this action class is the runner's own OIDC token
exchanged for a short-lived platform credential — no `SHIP_TOKEN` secret in any
consumer at all.** It is the exact religion the estate already practices from
the other side: the npm publish law abandoned long-lived tokens for trusted
publishing, and this is that law read from the consuming end.

It needs platform work before it can be an action change — an exchange endpoint
that verifies GitHub's OIDC JWT, and a repository↔account binding a user
creates once in the dashboard — so it is a **product decision, not an action
wave**. Recorded here because of what it dissolves: the long-lived secret class
entirely, and with it the key-rotation hygiene that follows every credential
that has ever appeared in a log, a transcript or a screen share.

The `environment:` input that used to sit in this section shipped in **v2.1.0**:
the record step's `pull_request ? preview : production` guess called every dev
push `production`, true of the deploy and false of the record. It is now the
fallback, and a consumer that names its environment gets it recorded.

**A consumer whose job declares `environment:` should not pass `github-token`
on a push at all** — GitHub writes that job's deployment record itself, so the
token buys a SECOND record for the same deploy. The input serves the consumer
this action was designed for: the quickstart workflow with no environments,
where the record step is the only thing writing one.

Both GitHub-side conveniences (the record, the sticky comment) remain
**unfenced by decision** — a bespoke harness for three rarely-changing shell
lines is machinery the estate's idiom refuses — so the new input rides that same
recorded call.

## Pointers

| What | Where |
|---|---|
| The whole product | `action.yml` |
| The fence | `.github/workflows/ci.yml`, `fixture/` |
| Fail-closed anonymity, `SHIP_VIA` / `SHIP_PASSWORD` / `SHIP_IDEMPOTENCY_KEY` | `npm/ship/CLAUDE.md` |
| `--json` shapes the `jq` paths bind | `npm/ship/CLAUDE.md`, "Output Conventions" |
| Branch & CI model, the publish law, the public-value law | root `CLAUDE.md` |
| Cutover choreography | root `backlog.md` |

---

*This file provides agent guidance. User-facing documentation lives in `README.md`.*
