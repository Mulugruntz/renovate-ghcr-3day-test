# Renovate ghcr.io 3-day cooldown test repo

A throwaway repo for observing how Renovate handles `minimumReleaseAge` on
`ghcr.io` Docker images, and which Dockerfile syntaxes it actually detects.

Everything is pinned to a deliberately old release: **`ghcr.io/astral-sh/uv:0.12.7-trixie-slim`**
(digest `sha256:92d38da2…`, built 2026-08-27), so there is plenty for Renovate to do.

## The headline finding — read this first

**`minimumReleaseAge` does not work for `ghcr.io` images.**

Renovate's `docker` datasource only reports release timestamps for **Docker Hub**,
where it reads Docker Hub's proprietary `tag_last_pushed` tag-API field. Straight
OCI registries — ghcr.io, ECR, GAR, Quay, Harbor — expose no equivalent, so
Renovate has no date to measure an age against.

From Renovate's own source, `lib/modules/datasource/docker/index.ts`:

```
releaseTimestampSupport = true
releaseTimestampNote =
  'Only supported on Docker Hub: The release timestamp is determined from
   the `tag_last_pushed` field in the results. ...'
```

What that means in practice, observed by running Renovate 44.95.0 against this repo:

```
DEBUG: Marking 9 release(s) as pending, as they do not have a releaseTimestamp
       and we're running with minimumReleaseAgeBehaviour=timestamp-required
```

Because the default `minimumReleaseAgeBehaviour` is `timestamp-required`, a release
with no timestamp is never considered stable. Combined with
`internalChecksFilter: "strict"`, the update is withheld **indefinitely** — not for
three days. You get a permanently pending entry on the Dependency Dashboard and no
PR, ever.

So the honest answer to "can we have a 3-day waiting period for ghcr pulls?" is:
not via the docker datasource. Your options:

| Option | Effect |
| --- | --- |
| `minimumReleaseAge` + `internalChecksFilter: strict` (LANE A here) | ghcr updates blocked forever. Almost certainly not what you want. |
| Add `minimumReleaseAgeBehaviour: "timestamp-optional"` | Undated releases count as stable, so ghcr updates flow immediately. No cooldown, but nothing is blocked. |
| Leave `internalChecksFilter` at its default (`none`) | The PR is raised right away and merely carries a pending `renovate/stability-days` status check. A branch-protection rule on that check is what actually enforces the wait. |
| Drive the version from `github-releases` / `github-tags` (LANE C here) | A genuine cooldown. These datasources do provide timestamps. |
| Use `schedule` instead | What the Renovate docs recommend for pacing. Not a per-release age, but predictable. |

The docs are also blunt about the general idea, in `configuration-options.md`:

> Do _not_ use `minimumReleaseAge` to slow down fast releasing project updates.

That applies to `uv`: it published 0.12.7 → 0.12.16 in the 22 days to 2026-09-18,
roughly a release every 2.4 days, and about half the gaps are under 3 days.

### Second finding: the hold targets the latest version only

Renovate does not fall back to "newest release that has aged out". With this repo it
emits exactly one update, to `0.12.16-trixie-slim`, flagged `pendingChecks: true`.
If 0.12.16 is superseded by 0.12.17 during the wait, the pending target moves to
0.12.17 and the clock restarts on the new version. Intermediate versions are skipped.

### Third finding: pure digest updates can never age out

`Dockerfile.unsupported` case 8d pins a digest with no tag. Renovate logs:

```
DEBUG: digest update of ghcr.io/astral-sh/uv has no releaseTimestamp to age against
       "minimumReleaseAgeBehaviour": "timestamp-required"
```

A digest-only update has no release timestamp by nature, so under `strict` it is
always pending. If you want digest refreshes to flow while version bumps wait, give
digests their own rule with `minimumReleaseAge: null`.

Also note what that untagged pin resolves to: Renovate calls
`getDigest(https://ghcr.io, astral-sh/uv, undefined)` and follows the **`latest`**
tag — here `sha256:adc68cd7…`, which is *not* a `trixie-slim` image. An untagged
digest pin will happily be re-pointed at a different image variant.

## Dockerfile syntax support — measured, not guessed

Produced by `renovate --platform=local --dry-run=extract` on Renovate 44.95.0.

| File | Syntax | Detected? |
| --- | --- | --- |
| `Dockerfile.from` | `FROM image:tag@digest` | yes |
| `Dockerfile.from-platform` | `FROM --platform=$BUILDPLATFORM image` | yes |
| `Dockerfile.from-platform` | `COPY --from=<stage alias>` | correctly ignored |
| `Dockerfile.copy-from` | `COPY --from=image` | yes |
| `Dockerfile.copy-from` | `COPY --link --from=image` | yes |
| `Dockerfile.run-mount` | `RUN --mount=from=image,…` | yes |
| `Dockerfile.run-mount` | `RUN --mount=type=bind,from=image,…` | yes |
| `Dockerfile.run-mount` | `RUN --mount=type=bind,readonly,from=image,…` | yes |
| `Dockerfile.run-mount` | `RUN --network=none --mount=type=bind,from=image,…` | yes |
| `Dockerfile.run-mount` | **a separate `--mount=type=cache,…` before the `from=` mount** | **NO** |
| `Dockerfile.arg-full-image` | `ARG IMG=…` + `FROM ${IMG}` | yes (rewrites the ARG line) |
| `Dockerfile.arg-registry` | `ARG REGISTRY=ghcr.io` + `FROM ${REGISTRY}/…` | yes |
| `Dockerfile.arg-version` | `ARG V=0.12.7` + `FROM …:${V}-trixie-slim` | detected, rewrite unreliable |
| `Dockerfile.arg-inline-default` | `${VAR:-image}` in FROM / COPY --from / RUN --mount | yes, all three |
| `Dockerfile.unsupported` | `COPY --from=${IMG}` | no — `contains-variable` |
| `Dockerfile.unsupported` | `RUN --mount=from=${IMG}` | no — `contains-variable` |
| `Dockerfile.unsupported` | `FROM $UNDEFINED/…` | no — `contains-variable` |
| `Dockerfile.unsupported` | `FROM image@digest` (no tag) | yes, but follows `latest` |
| `Dockerfile.unsupported` | `FROM scratch`, `COPY --from=0` | correctly ignored |

### The `RUN --mount` trap

This is the one worth acting on. The extractor's regex is:

```
^RUN(?:<esc-nl>| |\t|#.*\n|--[a-z]+(?:=[a-zA-Z0-9_.:-]+?)?)+--mount=(?:\S*=\S*,)*from=(?<image>[^, ]+)
```

The pre-flag alternation `--[a-z]+(?:=[a-zA-Z0-9_.:-]+?)?` excludes `=` from a flag's
value, so a preceding `--mount=type=cache` can never be consumed, and the match fails.
**The `from=` mount must be the first `--mount=` flag on the RUN instruction.**

Not detected — and this is the pattern the uv docs suggest for caching:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,from=ghcr.io/astral-sh/uv:0.12.7-trixie-slim@sha256:…,source=/uv,target=/usr/local/bin/uv \
    uv sync
```

Detected — identical build, mounts swapped:

```dockerfile
RUN --mount=type=bind,from=ghcr.io/astral-sh/uv:0.12.7-trixie-slim@sha256:…,source=/uv,target=/usr/local/bin/uv \
    --mount=type=cache,target=/root/.cache/uv \
    uv sync
```

Worth grepping your real repos for: a `--mount=type=cache` ahead of a `--mount=…from=`
means that image is silently never bumped.

### `ARG` + partial interpolation

`Dockerfile.arg-version` is the case to watch in the PR diff. Renovate resolves the
ARG and detects the dep (`currentValue=0.12.7-trixie-slim`), but the resolved string
appears nowhere in the file, and the ARG line holds `0.12.7`, not
`0.12.7-trixie-slim`, so auto-replace has no anchor. Prefer the
`Dockerfile.arg-full-image` shape — whole image reference in one ARG — which works
cleanly.

## What else is in here

Non-Dockerfile files that hit the same `docker` datasource, so you can confirm the
`ghcr.io/**` rule reaches them too. All were confirmed extracted:

| File | Manager | Note |
| --- | --- | --- |
| `compose.yaml` | docker-compose | matched by default patterns |
| `.devcontainer/devcontainer.json` | devcontainer | `depType: image` |
| `.github/workflows/renovate-targets.yml` | github-actions | `container:` and `services:` are both extracted; `workflow_dispatch` only, so it never runs |
| `k8s/deployment.yaml` | kubernetes | the kubernetes manager ships with **empty** file patterns — invisible until configured, which `renovate.json` does here |
| `versions.env` | regex custom manager | LANE A vs LANE C comparison |
| `dockerfiles/Dockerfile.control-dockerhub` | dockerfile | Docker Hub control, see below |

If you only want the hold on Dockerfiles, add `"matchManagers": ["dockerfile"]` to
the rule — otherwise it covers compose, k8s, devcontainer and Actions too.

## How to run it

```bash
git init && git add -A && git commit -m "renovate ghcr 3-day hold test"
gh repo create renovate-ghcr-3day-test --private --source=. --push
```

Then enable the Renovate GitHub App on the repo, or run it locally:

```bash
LOG_LEVEL=debug npx --package=renovate renovate --platform=local --dry-run=lookup
```

`--dry-run=lookup` does extraction and version lookup without touching branches
or PRs, which is enough to see every finding above. `--dry-run=extract` is faster
if you only care about the syntax matrix.

Running locally, LANE C in `versions.env` is skipped with
`skipReason: github-token-required` — Renovate refuses the GitHub datasources
without a token. Export one to exercise it:

```bash
GITHUB_COM_TOKEN=$(gh auth token) LOG_LEVEL=debug npx --package=renovate renovate --platform=local --dry-run=lookup
```

On a repo running the Renovate GitHub App a token is always present, so LANE C
needs no extra setup there.

### What to expect

- **The alpine control PR opens immediately.** Docker Hub supplies timestamps and
  every alpine release is years old. Its presence alongside stuck ghcr entries is
  the proof that the registry, not your config, is the variable. Verified on
  2026-09-18: `alpine 3.19.0 -> 3.24.1`, `pendingChecks` unset, while
  `ghcr.io/astral-sh/uv 0.12.7-trixie-slim -> 0.12.16-trixie-slim` came back
  `pendingChecks: true` under the identical 3-day rule.
- **Every ghcr.io entry sits on the dashboard under "Pending Status Checks"** and
  never becomes a PR.
- **All 20-odd ghcr occurrences collapse into 2 branches**, since they are all the
  same dependency: one version branch (`renovate/ghcr.io-astral-sh-uv-0.x`) and one
  digest branch (`renovate/ghcr.io-astral-sh-uv`).
- **The PR diff is the real support matrix.** When a PR does open, whichever of the
  14 package files it rewrites are the supported syntaxes; the untouched ones are not.
  To get that diff now, drop `internalChecksFilter` or set
  `minimumReleaseAgeBehaviour: "timestamp-optional"`.

`prHourlyLimit`, `prConcurrentLimit` and `branchConcurrentLimit` are all set to `0`
(unlimited) on purpose. The default `prHourlyLimit: 2` throttles PR creation and is
very easy to mistake for a working cooldown.

## Version timeline used here

| uv release | published | ghcr `created` (trixie-slim) |
| --- | --- | --- |
| 0.12.7 | 2026-08-27 | 2026-08-27T21:24:13Z |
| 0.12.13 | 2026-09-10 | 2026-09-10T18:53:03Z |
| 0.12.14 | 2026-09-15 | 2026-09-15T01:49:22Z |
| 0.12.15 | 2026-09-15 | 2026-09-15T11:34:50Z |
| 0.12.16 | 2026-09-18 | 2026-09-18T00:05:28Z |

Those `created` values come from the image config blob. They are genuine build times
here, but note that reproducible-build tooling (ko, Bazel, Nix, a pinned
`SOURCE_DATE_EPOCH`) often stamps a fixed or epoch date — the base Debian layer in
these very images carries `debian.sh … '@1787529600'`. Renovate does not read this
field for `minimumReleaseAge` anyway; it is here so you can date the images yourself:

```bash
docker buildx imagetools inspect ghcr.io/astral-sh/uv:0.12.7-trixie-slim \
  --format '{{json .Image}}' | jq -r '.["linux/amd64"].created'
```

## Versions this was checked against

Renovate 44.95.0, extractor logic read from `main`. `managerFilePatterns` is the
current option name; on Renovate older than v40 use `fileMatch` instead.
