# Renovate `minimumReleaseAge` on container images — what actually works

A test repo for one question: **can Renovate hold Dockerfile image bumps for 3 days?**

Every image below is pinned to a deliberately old release,
`astral-sh/uv 0.12.7-trixie-slim` (digest `sha256:92d38da2…`, built 2026-08-27),
in both registries, so there is always plenty for Renovate to do.

All findings were produced by running **Renovate 44.95.0** against this repo
(`--platform=local`), plus the live Mend-hosted app on the GitHub repo itself.
Nothing here is inferred from docs alone.

---

## TL;DR

`minimumReleaseAge` needs a **release timestamp per version**. Renovate's `docker`
datasource can only obtain one from **Docker Hub's** proprietary tag API field
`tag_last_pushed`. There are therefore two independent ways to fail, and this repo
hits both:

| Source | Timestamps? | Why |
| --- | --- | --- |
| `ghcr.io/astral-sh/uv` | no | ghcr is a plain OCI registry. No code path exists. Unfixable. |
| `docker.io/astral/uv` | no | Registry is supported, but the repo has **5448 tags** and anonymous Docker Hub refuses pagination past offset 1000. Renovate 403s and silently falls back to the timestamp-less API. **Fixable.** |
| `docker.io/library/alpine` | yes | Same registry, only **224 tags**, stays under the cap. Works out of the box. |

With no timestamps and the default `minimumReleaseAgeBehaviour=timestamp-required`,
every candidate is marked pending. Combined with `internalChecksFilter: "strict"`,
the update is withheld **indefinitely — not for three days.** You get a permanently
parked entry on the Dependency Dashboard and no PR, ever.

---

## Is it the registry, or how Astral built the image?

Neither, quite — and this was the most misleading part of the investigation.

**It is not how the image was built.** Renovate never reads the image config
`created` field, nor `org.opencontainers.image.created`, nor `label-schema.build-date`.
Grepping the whole docker datasource: zero references. `releaseTimestamp` is assigned
in exactly one place, from `tag_last_pushed`. The only labels it consumes are
`image.revision`, `image.source` and `image.url` — all for changelog links. You can
read `created` yourself, and it is a genuine build time here, but it is invisible to
`minimumReleaseAge`.

**For ghcr it is genuinely the registry.** No OCI registry exposes per-tag push dates,
and Renovate would need a manifest + config-blob fetch per candidate tag (~2N requests)
to synthesise them. So ghcr can never work, no matter what Astral do.

**For Docker Hub it is Astral's tag count, not their build.** `astral/uv` publishes
5448 tags (every uv version × every Python version × every base OS). Renovate pages
the Hub tag API and gets refused:

```
GET hub.docker.com/v2/repositories/astral/uv/tags?ordering=last_updated&page=11&page_size=100
  → 403  "pagination offset too large for anonymous requests; sign in to page further"
DEBUG: Docker: error fetching data from DockerHub
```

Page 10 returns 200, page 11 is 403 — the anonymous cap is exactly 1000 items. On
failure Renovate falls back to the plain registry `/v2/tags/list`, which carries no
timestamps. That fallback is silent: nothing in a hosted run tells you the cooldown
has quietly become a permanent block.

### Proof it is the pagination and nothing else

Same image, same registry, same digest, same rule — only the page cap changed:

```bash
RENOVATE_DOCKER_MAX_PAGES=10 renovate --platform=local --dry-run=lookup
```

```
astral/uv  0.12.7-trixie-slim -> 0.12.14-trixie-slim
updateType=patch   pendingChecks=None        ← not pending; the PR opens
newDigest=sha256:45d66bb5a1027a9026e35743755860fb888b4b0a19fb0ba3e31e31b094005afe
```

No 403, no "Marking N release(s) as pending". It skipped `0.12.16` (published that
day) and `0.12.15` (~2d19h old) and landed on `0.12.14`, the newest release that had
aged past 3 days — **with the correct multi-arch index digest**. That digest is
byte-identical to the one ghcr serves for `0.12.14-trixie-slim`.

So the whole feature — cooldown *and* digest pinning, one dependency, one PR, no
custom managers — works natively. It just needs the tag list to be reachable.

---

## What to actually do

### If you self-host Renovate

Set the page cap and pull from Docker Hub. This is the clean answer.

```json
// config.js / self-hosted admin config — NOT repo renovate.json
{ "dockerMaxPages": 10 }
```

```dockerfile
FROM astral/uv:0.12.7-trixie-slim@sha256:92d38da241c7962f8f863e288cc1c39795b79b6553245f623a82db6be95bdae0
```

```json
{
  "packageRules": [
    {
      "matchDatasources": ["docker"],
      "matchPackageNames": ["astral/uv"],
      "minimumReleaseAge": "3 days",
      "internalChecksFilter": "strict"
    }
  ]
}
```

With 1000 tags ordered by `last_updated` descending, the newest versions are always
on page 1, so a cap costs you nothing in practice.

Authenticating Docker Hub via a `hostRules` entry for `hub.docker.com` should also
lift the cap — the 403 says "sign in to page further" — but **this was not tested
here** and Hub's `/v2/repositories` API may want a session JWT rather than a PAT.

### If you use the Mend-hosted app

`dockerMaxPages` is self-hosted-only, so you cannot reach it. Options, best first:

1. **Pull from a Docker Hub repo with under ~1000 tags.** Nothing to configure. This
   is why the `alpine` control PR opens normally.
2. **Custom datasource.** Verified working — a `customDatasources` entry can return
   `version`, `releaseTimestamp` *and* `digest` together, and a custom regex manager
   can capture both `currentValue` and `currentDigest`. See
   [Appendix: custom datasource](#appendix-custom-datasource-hosted-workaround).
3. **`schedule` instead.** What the Renovate docs recommend for pacing. Keeps digests
   and native behaviour; it just is not a per-release age gate.
4. **Give up the cooldown, keep the flow.** See the filter table below.

### Choosing `internalChecksFilter`

This is the setting that decides whether "no timestamp" means "wait" or "never".

| Value | Behaviour when a release is too new / undated |
| --- | --- |
| `none` | PR created anyway with the highest release, carrying a pending `renovate/stability-days` check. Branch protection on that check is what enforces the wait. |
| `flexible` | Falls back to an aged release if one exists; if *all* are pending, creates a PR with the highest pending one. **Best fit for undated ghcr images** — you keep getting PRs, and dated sources still get a real cooldown. |
| `strict` | PR suppressed entirely unless a non-pending version exists. On ghcr this means silence forever. |

`flexible`'s documented "flapping" risk does not apply to undated images: with no
timestamps nothing ever transitions from pending to passing, so there is nothing to
flap to.

Also available: `minimumReleaseAgeBehaviour: "timestamp-optional"` treats an undated
release as stable. That unblocks ghcr but removes the cooldown entirely — it does not
split the difference.

### One more trap

`prHourlyLimit` defaults to **2**. Throttled PR creation looks exactly like a working
cooldown. This repo sets `prHourlyLimit`, `prConcurrentLimit` and
`branchConcurrentLimit` to `0` so that variable is off the table.

---

## Dockerfile syntax support — measured

From `--dry-run=extract` on Renovate 44.95.0. The `dockerfiles/` and
`dockerfiles/dockerhub/` trees are identical apart from the registry.

| Syntax | Detected? |
| --- | --- |
| `FROM image:tag@digest` | yes |
| `FROM --platform=$BUILDPLATFORM image` | yes |
| `COPY --from=image` | yes |
| `COPY --link --from=image` | yes |
| `RUN --mount=from=image,…` | yes |
| `RUN --mount=type=bind,from=image,…` | yes |
| `RUN --mount=type=bind,readonly,from=image,…` | yes |
| `RUN --network=none --mount=type=bind,from=image,…` | yes |
| **`RUN --mount=type=cache,… --mount=…from=image,…`** | **NO — see below** |
| `ARG IMG=<full ref>` + `FROM ${IMG}` | yes — rewrites the ARG line |
| `ARG REGISTRY=…` + `FROM ${REGISTRY}/…` | yes |
| `ARG V=0.12.7` + `FROM …:${V}-trixie-slim` | detected, but rewrite unreliable |
| `${VAR:-image}` in FROM / COPY --from / RUN --mount | yes, all three |
| `COPY --from=${IMG}` | no — `contains-variable` |
| `RUN --mount=from=${IMG}` | no — `contains-variable` |
| `FROM $UNDEFINED/…` | no — `contains-variable` |
| `FROM image@digest` (no tag) | yes, but follows `latest` — see below |
| `FROM scratch`, `COPY --from=0`, `COPY --from=<stage>` | correctly ignored |

### The `RUN --mount` trap

The extractor's pre-flag pattern is `--[a-z]+(?:=[a-zA-Z0-9_.:-]+?)?`, which excludes
`=` from a flag's value. A preceding `--mount=type=cache` therefore cannot be consumed
and the whole match fails. **The `from=` mount must be the first `--mount=` on the
`RUN` instruction.**

Not detected — and this is the pattern the uv docs suggest for caching:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,from=astral/uv:0.12.7-trixie-slim@sha256:92d38da2…,source=/uv,target=/usr/local/bin/uv \
    uv sync
```

Detected — identical build, mounts swapped:

```dockerfile
RUN --mount=type=bind,from=astral/uv:0.12.7-trixie-slim@sha256:92d38da2…,source=/uv,target=/usr/local/bin/uv \
    --mount=type=cache,target=/root/.cache/uv \
    uv sync
```

Worth grepping real repos for: a `--mount=type=cache` ahead of a `--mount=…from=`
means that image is silently never bumped. This is independent of registry and of the
timestamp problem.

### `ARG` + partial interpolation

`Dockerfile.arg-version` is the one to watch in a PR diff. Renovate resolves the ARG
and detects the dep, but the resolved string appears nowhere in the file and the ARG
line holds `0.12.7`, not `0.12.7-trixie-slim`, so auto-replace has no anchor. Prefer
the `Dockerfile.arg-full-image` shape — whole reference in one ARG — which works.

---

## Digests: index vs architecture

`sha256:92d38da2…` is an **OCI image index** (multi-arch). It contains:

```
linux/amd64        sha256:bd3571a45f9fcd715ca26938a30ef0bbe85a813d4a7c3e9a7dc0e27997f1a442
linux/arm64        sha256:f0c9e8c3b86decaa94dbb7e134d3a3241b7cf4c8c66da2f1bc257d8036bca972
unknown/unknown    sha256:f68af0dd…   (buildx attestation)
unknown/unknown    sha256:29e9877b…   (buildx attestation)
```

Docker Hub's layer-browser UI deep-links to the **per-architecture** child, so the
digest in a Hub URL will not match the one you pin. Always pin the index digest —
which is what Renovate reads and writes. Pinning a child digest hard-locks that
`FROM` to one architecture.

To read build times, remember `.Image` is a **platform-keyed map** for multi-arch
images, so a bare `.created` yields `null`:

```bash
docker buildx imagetools inspect astral/uv:0.12.7-trixie-slim \
  --format '{{json .Image}}' | jq -r 'to_entries[] | "\(.key)\t\(.value.created)"'
```

### Untagged digest pins follow `latest`

`Dockerfile.unsupported` pins a digest with no tag. Renovate calls
`getDigest(registry, astral-sh/uv, undefined)` and follows **`latest`** — a different
image variant entirely, and a fast-moving target. It also logs:

```
digest update of ghcr.io/astral-sh/uv has no releaseTimestamp to age against
"minimumReleaseAgeBehaviour": "timestamp-required"
```

A pure digest update has no release timestamp by nature, so under `strict` it is
pending forever. If you want digest refreshes to flow while version bumps wait, give
digests their own rule with `minimumReleaseAge: null`.

---

## What is in this repo

| Path | Manager | Purpose |
| --- | --- | --- |
| `dockerfiles/Dockerfile.*` | dockerfile | syntax matrix, `ghcr.io` |
| `dockerfiles/dockerhub/Dockerfile.*` | dockerfile | same matrix, `docker.io` — isolates registry from syntax |
| `dockerfiles/Dockerfile.control-dockerhub` | dockerfile | `alpine:3.19.0`, 224 tags — the working control |
| `compose.yaml` | docker-compose | default patterns |
| `.devcontainer/devcontainer.json` | devcontainer | `depType: image` |
| `.github/workflows/renovate-targets.yml` | github-actions | `container:` and `services:` are both extracted; `workflow_dispatch` only, never runs |
| `k8s/deployment.yaml` | kubernetes | the kubernetes manager ships with **empty** file patterns — invisible until configured |
| `versions.env` | regex | `docker` vs `github-releases` side by side |

All the non-Dockerfile managers use the same `docker` datasource, so a
`ghcr.io/**` rule reaches them too. Add `"matchManagers": ["dockerfile"]` if you want
the hold confined to Dockerfiles.

## Observed result on the live repo

With `internalChecksFilter: "strict"` and a fresh Renovate binding:

```
## Pending Status Checks
 - Update astral/uv Docker digest to …
 - Update ghcr.io/astral-sh/uv Docker digest to …
 - Update astral/uv Docker tag to v0.12.16
 - Update docker.io/astral/uv Docker tag to v0.12.16
 - Update ghcr.io/astral-sh/uv Docker tag to v0.12.16

## Open
 - Update dependency astral-sh/uv to v0.12.14   (github-releases — real cooldown)
 - Update alpine Docker tag to v3.24.1          (Docker Hub, 224 tags — real cooldown)
```

Two PRs open, five parked. Note `astral/uv` is parked alongside ghcr — the Docker Hub
registry alone does not save you when the repo has too many tags.

Note also that all ~20 ghcr occurrences collapse into 2 branches, since they are all
one dependency: a version branch and a digest branch. **The PR diff is the real
support matrix** — whichever package files a PR rewrites are the supported syntaxes.
To get that diff now, tick a checkbox in the Pending Status Checks block; it forces
branch creation immediately.

## How to run it

```bash
LOG_LEVEL=debug npx --package=renovate renovate --platform=local --dry-run=lookup
```

```bash
RENOVATE_DOCKER_MAX_PAGES=10 LOG_LEVEL=debug npx --package=renovate renovate --platform=local --dry-run=lookup
```

The second command is the experiment above. `--dry-run=extract` is faster if you only
want the syntax matrix. Locally, `versions.env`'s `github-releases` entry is skipped
with `skipReason: github-token-required`; export `GITHUB_COM_TOKEN=$(gh auth token)`
to exercise it. The hosted app always has a token.

## Version timeline

| uv release | GitHub published | ghcr `created` (trixie-slim) |
| --- | --- | --- |
| 0.12.7 | 2026-08-27 | 2026-08-27T21:24:13Z |
| 0.12.13 | 2026-09-10 | 2026-09-10T18:53:03Z |
| 0.12.14 | 2026-09-15 | 2026-09-15T01:49:22Z |
| 0.12.15 | 2026-09-15 | 2026-09-15T11:34:50Z |
| 0.12.16 | 2026-09-18 | 2026-09-18T00:05:28Z |

uv shipped 0.12.7 → 0.12.16 in 22 days, about one release every 2.4 days, with roughly
half the gaps under 3 days. Renovate's own docs are blunt about this shape of
dependency:

> Do _not_ use `minimumReleaseAge` to slow down fast releasing project updates.

The cooldown also applies per version, not per package: it does not wait for a quiet
period, and intermediate versions are skipped. If `0.12.16` is superseded during the
wait, the target moves and the clock restarts on the new version.

---

## Appendix: custom datasource (hosted workaround)

Verified working. Feeds Renovate version + timestamp + digest from Docker Hub's tag
API in one request, staying under the anonymous pagination cap via `name=`. Digests
served by Hub are identical to ghcr's, so the resulting pin is valid for either.

```json
{
  "customDatasources": {
    "uvTrixieSlim": {
      "defaultRegistryUrlTemplate": "https://hub.docker.com/v2/repositories/astral/uv/tags?page_size=100&ordering=last_updated&name=trixie-slim",
      "format": "json",
      "transformTemplates": [
        "{ \"releases\": [results[$count($match(name, /^[0-9]+\\.[0-9]+\\.[0-9]+-trixie-slim$/)) > 0].{ \"version\": $substringBefore(name, \"-trixie-slim\"), \"releaseTimestamp\": tag_last_pushed, \"digest\": digest }] }"
      ]
    }
  },
  "customManagers": [
    {
      "customType": "regex",
      "managerFilePatterns": ["/(^|/)Dockerfile$/"],
      "matchStrings": [
        "FROM ghcr\\.io/astral-sh/uv:(?<currentValue>\\d+\\.\\d+\\.\\d+)-trixie-slim@(?<currentDigest>sha256:[a-f0-9]{64})"
      ],
      "depNameTemplate": "ghcr.io/astral-sh/uv",
      "datasourceTemplate": "custom.uvTrixieSlim",
      "versioningTemplate": "semver"
    }
  ],
  "packageRules": [
    {
      "matchPackageNames": ["ghcr.io/astral-sh/uv"],
      "minimumReleaseAge": "3 days",
      "internalChecksFilter": "strict"
    }
  ]
}
```

Result: `0.12.7 → 0.12.14`, not pending, `newDigest=sha256:45d66bb5…`.

Caveats:

- **Disable the dockerfile manager on those files**, or two deps fight over the same
  line. `"dockerfile": {"managerFilePatterns": []}` does **not** work; use
  `{"matchManagers": ["dockerfile"], "matchFileNames": ["path/to/Dockerfile"], "enabled": false}`.
- The `name=` filter is load-bearing: one request returns at most 100 tags, and
  filtering to `trixie-slim` covers 0.12.6 → 0.12.16. Another variant needs its own entry.
- A tag/digest mismatch cannot be committed: auto-replace substitutes both into the
  same string and re-checks, logging `Digest is not updated` and abandoning the update
  rather than committing half of it.
- `tag_last_pushed` is push time, not build time. Re-pushing a tag resets its clock —
  arguably correct for a cooldown. Renovate notes that digests inherit the tag's
  timestamp and so can look newer than they are ([renovate#38659](https://github.com/renovatebot/renovate/issues/38659)).

Prefer `dockerMaxPages` if you self-host; this appendix exists only because the hosted
app cannot set it.

---

## Checked against

Renovate 44.95.0 (local runs) and the Mend-hosted app, 2026-09-18. Extractor and
datasource logic read from `main`. `managerFilePatterns` is the current option name;
on Renovate older than v40 use `fileMatch`.
