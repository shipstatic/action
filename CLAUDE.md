# CLAUDE.md

Claude Code instructions for **`shipstatic/action`**, the GitHub Action.

## Identity

A **composite** action: `action.yml` plus docs, plus a fixture and a workflow
that prove them. No build, no manifest, no runtime code. It installs the
`@shipstatic/ship` CLI on the runner and invokes it **once** (a deploy that
links a domain when one was asked for), and wraps that in one GitHub-side
convenience, a sticky PR comment.

**v3 speaks the 3.x platform.** The action's major and the platform major it
targets are the same number, which is the versioning law below.

## The law: majors pin majors

An action whose interface is a contract with a CLI names the CLI major it
speaks. `@v1` speaks ship 1.x, `@v2` ship 2.x, `@v3` ship 3.x, and the install
line says so in bytes.

The action is the one consumer that resolves the CLI at **runtime**, so
whatever the registry serves that minute runs inside every consumer's workflow.
A floating `@latest` across a major has the worst failure shape the platform
can produce: ship's anonymity is **fail-closed** (proven absence of a
credential IS anonymity), so a new CLI that no longer reads the old credential
variable deploys with a 201 into the public account, with a claim URL, a
three-day expiry and no error anywhere. A consumer that links a domain fails a
step later; one that does not gets no signal at all.

`.github/workflows/ci.yml` holds the law: the install line names a ship major,
and that major is `3`.

### The pin

The install line reads `@shipstatic/ship@3` and stays there for the life of this
major. **That line is this repo's constellation pin**, since it has no
manifest. A range is right because it is a major: every 3.x fix reaches every
`@v3` consumer with no release here, which is the argument for staying
composite.

The fence checks the MAJOR rather than the literal, so the line may carry an
exact `X.0.0-beta.N` while no stable of a new major exists (npm ranges never
match prereleases). A version is not an environment value, so either form is
lawful in a public tree.

## Releases

The release IS the annotated tag; the channel IS the moving major tag. This is
the platform's publish law translated to a repo with no registry, no manifest
and no version field.

- `vX.Y.Z` tags are immutable.
- `v1`, `v2`, `v3` are the moving tags consumers bind; `v3` is the live one.
  Fast-forward them deliberately and **never force-push one backwards**: that
  rewrites what already runs in other people's CI.
- Betas are `vX.Y.Z-beta.N` tags that no major tag points at.
- The gate before any tag is `ci.yml` green **on the tagged commit**.

There is deliberately **no release workflow**. A two-file repo does not earn
one, and the ritual is three commands:

```bash
git tag -a v3.0.3 -m "v3.0.3" && git push origin v3.0.3
gh release create v3.0.3 --verify-tag --title v3.0.3 --notes "..."
git tag -f -a v3 -m "v3 to v3.0.3" && git push -f origin v3
```

**The Release object is part of the tag, the way the registry entry is part of
a publish.** The Marketplace listing takes its version from Releases, and
Dependabot and Renovate key their bumps and release notes off them, so a
tag-only version is invisible to the consuming ecosystem.

- **Every `vX.Y.Z` tag gets a Release.**
- **Order and `--latest=false` matter on a catch-up**, because GitHub hands
  Latest to the most recently CREATED release whatever its version: create the
  older ones with `--latest=false`, and the current version last, without the
  flag.
- Dead historical tags (nothing points at them) stay Release-less: a storefront
  backfilled with versions nobody may install is noise.

**v1 and v2 are frozen by their own pins** (`@shipstatic/ship@1`, `@2`) and
receive no changes; a consumer moves to v3 for the 3.x CLI. Never force-push
either backwards.

## The v3 surface

Every absence below is a decision; see "Recorded absences".

| Input | Default | Behaviour |
|---|---|---|
| `token` | none | One slot, any platform token (`ship-` key or `deploy-` token), as `SHIP_TOKEN`. Absent = anonymous deploy |
| `api-url` | *the CLI's* | As `SHIP_API_URL`. Empty is absence, so the CLI's own default (production) applies |
| `path` | `.` | Directory to deploy |
| `domain` | none | Passed as `ship --domain`: deployed and linked in ONE invocation. Requires `token`; the CLI refuses before the upload without one |
| `ttl` | none | Passed as `ship --ttl`, in seconds. Requires `token`; the platform refuses it together with `domain` |
| `password` | none | As `SHIP_PASSWORD` (the empty-unset guard stays) |
| `labels` | none | Comma-separated, as repeated `--label`, after the automatic short SHA |
| `idempotency-key` | *derived* | Override; else `<repo>-<run id>-<job id>`, as `SHIP_IDEMPOTENCY_KEY` |
| `github-token` | `${{ github.token }}` | Enables the sticky PR comment, and nothing else. Empty = no comment |

| Output | Source |
|---|---|
| `deployment` | `jq -r '.deployment'`, the canonical key |
| `url` | `jq -r '.url'`, the wire's own, never reconstructed. With `domain` it is the domain's URL, since the answer is the domain's |
| `claim` | `jq -r '.claim // empty'` |
| `expires` | `jq -r '.expires // empty'`, unix seconds; feeds the comment's date. Anonymous and `ttl` deploys carry one |

Automatic, no knob: `SHIP_VIA=git` • the commit short-SHA label • the derived
idempotency key • the run summary • SPA detection and junk filtering (the CLI's
defaults; `ship.json` is the override).

### One invocation, one exit code, one JSON

`domain` is a flag on the deploy, not a second command. `--domain` answers as
the domain, and that shape carries `deployment` and `url`, the two values this
action publishes. It lacks `claim` and `expires`, truthfully: a linked
deployment is owned, so it has nothing to claim and no deadline. A failed link
still leaves the deployment id on screen (the CLI prints it when the upload
lands), and a re-run replays the upload through the derived key. The outputs on
that path are lost, accepted because the job fails and nothing downstream runs.

What it buys:

- **A keyless `domain:` fails loudly, before a byte uploads.** The CLI refuses
  the invocation, because an anonymous account cannot own a domain.
- **`url` is the address that was asked for**: the domain's whenever a domain
  was named.
- **One process, one exit code, one JSON**, one credential wiring, one
  `api-url`, one step to read.

### A deadline is not a claim

A `ttl` deploy is owned AND expiring, so `expires` no longer implies `claim`.
Each value is asked about on its own: two rows in the summary, and in the
comment a sentence about the deadline plus a claim link appended only when there
is something to claim. Otherwise a fused note would render a claim link with an
empty href on someone's pull request. A `ttl` run shows the expiry and no claim
link; an anonymous run shows both.

**No e2e leg for `ttl` itself.** The flag either reaches the CLI or it does not,
and ship's own suite owns every consequence; a leg here would deploy an
expiring row to prove a string was concatenated.

### The run summary belongs to the action

A deploy action that says nothing on the run page makes every serious consumer
write its own table, so the action writes `$GITHUB_STEP_SUMMARY`: deployment,
URL and, where the wire carries them, the claim link and the rendered expiry.
**No opt-out knob**: a summary is inert, and nothing can depend on it.

- **Inside the Deploy step, not a step of its own.** Each step gets its own
  summary FILE, which the job concatenates, so a table split across two steps
  does not render as one. The step count is fenced besides, which makes a step
  a design change.
- **Every row is the wire's own.** There is no `Domain` row: with a domain,
  `URL` *is* the domain's URL.
- **The expiry is rendered once**, in the Deploy step, and exposed as an
  internal `expires-at` step output the PR comment consumes, so one `date`
  incantation exists. `outputs:` does not carry it.

### Everything is read off the wire

The action never reconstructs `url` from the deployment id and never hardcodes
a duration: **this file has no compiler and no test runner, so the only facts
it can state truthfully forever are the ones it reads off the response.** The
comment renders `expires` as a real date, since a relative duration is stale
the moment it is read.

Restatements that remain, each decided:

- **Prose, tolerated:** the `my.shipstatic.com/api-key` link (owned by
  `MY_API_KEY_URL` in `@shipstatic/types`) and the password range 6 to 128 in
  the input description and README (owned by `PASSWORD_CONSTRAINTS`; the CLI
  validates with the real rule and relays its message). Bash and Markdown
  cannot import, and prose that teaches a value keeps the value.
- **Env-var names, held by the CI fence:** the yaml copies `SHIP_API_URL` and
  `SHIP_TOKEN`, whose vocabulary `SHIP_ENV` in `@shipstatic/types` owns. The
  drift is loud (a wrong name makes the authed leg deploy anonymously and fail
  its inverse assertion), so the e2e legs hold it rather than a copy compare.

### `api-url` has no default in `action.yml`, deliberately

The production endpoint is `DEFAULT_API` in `@shipstatic/types`. Writing it
here would be a second copy in a file that cannot import it, so the input has no
`default:`; the SDK normalizes an empty `SHIP_API_URL` to absence and the CLI's
own default applies. **The action does not decide the endpoint**, so it can
never disagree about it.

### The idempotency key

`<owner>/<repo>-<run id>-<job id>`, derived in step bash because an `action.yml`
`default:` is a static string and cannot hold an expression. `GITHUB_RUN_ID` is
stable across re-runs and fresh per push, so **Re-run jobs** replays the
original 201 rather than deploying twice. Each segment matters:

- **job**: two jobs in one run may deploy two sites; one key would replay the
  first site's 201 into the second job.
- **repository**: a run id is unique only within a repository, and the API's
  replay namespace is the account, which many repositories can share. Without
  it, one repository could be handed another's deployment.

An anonymous deploy's key is ignored by the API (a key replays only within an
account of its own), so a keyless re-run is a new deployment. Matrix legs
deploying different content and two invocations in one job have no ambient
discriminator and take the override input; the README shows both.

`SHIP_IDEMPOTENCY_KEY` is a CLI-only env var on the `SHIP_PASSWORD` tier, added
to `@shipstatic/ship` for this consumer. A `--idempotency-key` flag is ship's
own product call, not this repo's.

### The sticky PR comment

One comment per PR, updated in place, found by a hidden marker
(`<!-- shipstatic/action -->`) rather than `gh pr comment --edit-last`, which
edits the identity's last comment whatever wrote it, so two workflows commenting
as `github-actions[bot]` would clobber each other.

`github-token` defaults to `${{ github.token }}`: an expression in an input
default evaluates in the CALLER's context (the `actions/stale` and
`actions/labeler` convention), so the comment appears with no configuration
wherever permissions allow, and `continue-on-error` keeps a locked-down org
green. **The step's non-empty test is the opt-out**: `github-token: ''` skips
it. Deleting that guard would turn opting out into a step that fails and is
swallowed, a default-on side effect on someone else's pull requests with no
way to say no. The input also serves the cross-repo token case.

### No deployment record, no `environment:` input

A job that declares `environment:` already makes GitHub write the deployment
record, with the environment the consumer named, so a step writing one could
only add a SECOND record. The consumer's own `environment:` names the
environment one level up, where GitHub reads it. The audience that declares
no environment is the quickstart audience, which does not read the
deployments sidebar.

## The fence

`.github/workflows/ci.yml`, tests-only on both branches:

| Job | What it holds |
|---|---|
| `lint` | actionlint over `.github/workflows/**`; the install line names ship major 3; the action is three shell steps (Install, Deploy, Comment) |
| `e2e-anonymous` | Deploys `fixture/` keyless to dev; asserts the four outputs, that `url` addresses `deployment`, that claim and expires are both present, that the URL serves 200, and that a second same-job invocation with the same derived key is a SECOND deployment (an anonymous key is ignored, so one stranger's site is never replayed to another) |
| `e2e-authenticated` | Deploys with a token and a domain; asserts the deployment is not claimable or expiring, reads `via` and both labels back, and that `url` is the domain's, serves 200, and the domain points at this run's deployment. A second same-job invocation REPLAYS the first, which catches an unread `SHIP_IDEMPOTENCY_KEY` (it would keep succeeding and duplicate) |

Deliberate:

- **The e2e domain is DERIVED, never written down.** The authed leg links
  `action-e2e.${API_URL#https://api.}`, built from the one environment value it
  already holds, because this repo is public and production is the only
  hostname a tracked file carries. A second variable would be a second owner
  for a fact `SHIP_API_URL` already decides. The first run creates the domain,
  every later run repoints it, and `links` in the read-back row shows it.
- **The authed leg asserts WHOSE account it deployed into**, against
  `CI_ACCOUNT_EMAIL`. Every other check describes the deployment, and an owned,
  labelled row looks the same in any account, so a credential swapped for a
  person's would stay green. **Neither value is ever echoed.** The expected one
  is a SECRET: the runner echoes a step's whole `env:` block into the log
  unmasked before the step runs, so a `vars.` value publishes itself, while
  secrets are masked to `***`.
- **The authed leg's other assertion is the INVERSE of the anonymous one.**
  Fail-closed anonymity means a credential the CLI does not read still yields a
  clean 201. Only *no claim, no expiry* proves the token was read.
- **The authed leg's rows accumulate, accepted.** It adds one owned row per push
  and owned deployments never expire, while the plan cap counts every row
  whatever its status, so the CI account fills on a schedule. A prune step is
  refused: a first-party
  workflow carrying private bash for a product the platform does not offer is
  anti-dogfooding, and the day the account's cap fires is usage data for a
  retention feature (`prune --keep`, or an account-level policy). Then the deploy refuses with the platform's own message,
  and the sweep is by hand on the `ci-fence` label the leg deploys with.
- **The jobs refuse an unconfigured environment.** With no `SHIP_API_URL` the
  CLI falls back to production, and `development` CI would deploy junk there on
  every push.
- **actionlint does not read `action.yml`**, only workflows. A bespoke extractor
  to shellcheck the composite's `run:` blocks is refused: one that silently
  found zero blocks would be a green fence proving nothing. The step count
  stands in: whoever must change that number rereads the shell.

**Fork pull requests skip both e2e jobs**, by a job-level `if:`, the only place
that can. GitHub withholds environment secrets and variables from a fork, so an
empty `SHIP_API_URL` there is expected, and a red e2e on a community PR would be
a lie. A maintainer re-runs the change from a branch.

**The jobs read a `development` GitHub Environment:** a `SHIP_API_URL`
variable, a `CI_ACCOUNT_EMAIL` secret and, optionally, a `SHIP_TOKEN` secret.
Without `SHIP_TOKEN` the authed leg skips with a `::notice`; without the other
two the jobs fail. The token is an **API key, not a deploy token**, because the
read-back (`ship deployments get`) is not a deploy, and it belongs to a
dedicated CI account: a public repo's CI never holds a person's credential.
`CI_ACCOUNT_EMAIL` cannot be a tracked literal: a non-production address in a
public repo fails the platform's public-value check.

## Tooling standard

Applied as far as a repo with no manifest can take it:

| Present | Absent, and why |
|---|---|
| This file, `AGENTS.md` | Biome: nothing here it parses |
| Pre-commit credential scan (`scripts/githooks/pre-commit`) | `packageManager` / Node pin: no manifest |
| actionlint in CI | Coverage: no runtime to measure |
| A tests-only `ci.yml` on both branches, with a failure notice to the operator's alerts channel | |
| `renovate.json` (org preset; it updates `ci.yml`'s pinned actions, actionlint included) | |

The hook has no `prepare` script to install it. Run once per clone:

```bash
git config core.hooksPath scripts/githooks
```

The repo is **public**: production is the only public value, and the dev API URL
arrives from the `development` GitHub Environment, never from a file.

## What must not change

- **Composite, never a bundled JS action.** Bundling the SDK would freeze the
  client per action release, so every CLI fix would need a release here.
  Install-then-invoke under majors-pin-majors delivers every 3.x fix with zero
  releases, and **tracking the platform is this product's whole job**: an
  editor extension bundles because the editor owns its release train, and this
  action's train is the platform's. A bundled action would also be a third
  full-bar surface (suite, fences, coverage) for two CLI invocations. Costs
  accepted: about 5 to 10 s of install per run, and coupling to npm's
  availability.
- **`SHIP_VIA=git`**: the action IS the `git` surface, whatever invoked the
  workflow, and it is why that member exists in `DeploymentVia`.
- **The automatic short-SHA label**: ambient attempt metadata, like `SHIP_VIA`
  and the derived key.
- **The `SHIP_PASSWORD` empty-unset guard**: defence in depth for whatever CLI
  the pin resolves, though the CLI normalizes empty to absence itself.
- **The fenced step shape**, and `continue-on-error` on the PR comment: a failed
  comment never fails a deploy that succeeded.
- **Anonymous deploy as a first-class path**: the keyless quickstart is a
  product demo.
- **Output names within a major.** v1's `id`/`url`/`claim` froze with it;
  `deployment`/`url`/`claim`/`expires` are frozen from `v2.0.0` on.

## Recorded absences

- **No `cli-version` input.** Majors-pin-majors IS the contract; an override
  would let a consumer float, the failure above wearing a knob.
- **No `spa-detect` / `path-detect` toggles.** The SDK's defaults are right, and
  a repo needing otherwise ships a `ship.json`, which detection defers to. A
  toggle would be a second way to say what the config file owns.
- **No `setup-node` step.** Hosted runners carry Node 20 or later; the README
  states the requirement for self-hosted ones. Installing our own would slow
  every consumer for the exception's benefit.
- **No domain-link output.** The linked URL is `https://<domain>` by
  construction; an output would restate an input. The deployment is the
  product; `domain` is routing for it.
- **No validation of `domain` or `ttl`.** Both pass through. The CLI refuses the
  combined flags before any upload, the API makes the state inexpressible, and
  the credential refusal is the CLI's too. A third validator could only disagree
  with the two that decide, including on the day the platform relaxes the rule.
  Every message relays.
  The labels loop follows the same law: it trims edges and refuses nothing.
- **No `via` knob** (law) and **no timeout inputs** (the CLI owns its budgets).

## Consumers

`shipstatic/action-example`: five workflows, all on `@v3` and green against
production, including `deploy-no-account.yml`. Its domain belongs to a dedicated
account, since a domain is linkable only by the account that owns it.

The platform's own web apps ship through `@v3`: a push to `development` deploys
the dev site, and **the merge to `main` IS the production deploy**.

## Pointers

| What | Where |
|---|---|
| The whole product | `action.yml` |
| The fence | `.github/workflows/ci.yml`, `fixture/` |
| Fail-closed anonymity, `SHIP_VIA`, `SHIP_PASSWORD`, `SHIP_IDEMPOTENCY_KEY`, the `--json` shapes the `jq` paths bind | `@shipstatic/ship` (its `CLAUDE.md` and README) |

---

*This file provides agent guidance. User-facing documentation lives in `README.md`.*
