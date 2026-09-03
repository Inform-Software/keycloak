# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

INFORM's fork of Keycloak. `origin` is `Inform-Software/keycloak`, `upstream` is `keycloak/keycloak`.

`main` is kept as a near-pure mirror of upstream. Only two files are fork-owned:
`.github/workflows/publish-backport-image.yml` and this one — everything else on `main` should
match `upstream/main`. Preserve that. Fork changes belong on backport branches and tags, not on
`main`: every edit to an upstream-owned file becomes a merge conflict on the next upstream sync,
so only touch one when the change is genuinely fork-specific and worth paying that cost forever.

Everything under `.github/copilot-instructions.md` applies here too (toolchain, focused-build
patterns, testing strategy); this file covers what that one doesn't.

## Backporting and publishing an image

The fork exists to ship patched Keycloak images. `.github/workflows/publish-backport-image.yml`
is dispatched manually with a git **tag**, builds Keycloak from source at that ref, and pushes
`ghcr.io/inform-software/keycloak:<tag>`.

The consumer is `inform-cloud-services/co-keycloak-container-green` — its `Dockerfile` takes
`KEYCLOAK_IMAGE` / `KEYCLOAK_VERSION` build args and its `.github/workflows/backport.yaml`
sets `keycloak_image: 'ghcr.io/inform-software/keycloak'`. It runs `kc.sh build` itself, so the
image published from here is deliberately the *unoptimized* base image.

### Case A — rebuilding an unchanged upstream release

Just dispatch the workflow with `tag: 26.6.6`. Nothing else. Every upstream release tag already
points at its own "Set version to X" commit (e.g. `26.6.6` → `85af03cdf8`) with the pom version
already stamped, so the pom, the tarball, the image tag and the `version` label all agree.

### Case B — fork changes on top of a release

One long-lived branch per patched upstream release, in upstream's own `release/*` namespace:
`release/<version>-inform` is upstream tag `<version>` plus cherry-picks, and each published
build is a tag `<version>-inform.<n>` on it. Cherry-pick from the upstream **release-line**
commit, not from `main` — the two variants differ (the 26.6 backport of CVE-2026-18963 keeps a
`setEmailVerified` call that `main` had already moved elsewhere), and taking `main`'s is a silent
regression.

Take the whole patch set for the flow, not just the CVE commit. When the change re-enables an
attack surface, cherry-pick every upstream fix that landed on it — `26.6.4-inform.1` took
`869c3fd21f` (rollback when reset-credentials token verification fails, no CVE ID, 26.6.5)
alongside the CVE fix `62dd952e6c` (26.6.6). Find them by **flow, not by path**: a `git log`
scoped to the obvious directory misses them, because `869c3fd21f` lives in
`services/managers/AuthenticationManager.java` and `services/ErrorPageException.java`, nowhere
near `authenticators/resetcred/`. Anything protective left out on purpose goes in the PR body as a
documented exclusion — for that tag, `fccbaba040`, which changes the `BruteForceProtector` SPI
signature and lockout semantics and would invalidate the tested baseline.

The patch goes in through a **PR against the release branch**, so the change is reviewed and the
reasoning is on the record for the vulnerability register:

```bash
git fetch upstream --tags
git branch release/26.6.4-inform 26.6.4          # base: the bare upstream tag, no commits of ours
git switch -c fix/cve-2026-18963 release/26.6.4-inform
git cherry-pick -s 62dd952e6c                    # upstream's own 26.6-line commit
git diff --stat release/26.6.4-inform HEAD       # expect only the intended files
git show HEAD | git patch-id --stable            # must match `git show 62dd952e6c | git patch-id --stable`
git push -u origin release/26.6.4-inform
git push -u origin fix/cve-2026-18963
gh pr create --base release/26.6.4-inform --head fix/cve-2026-18963   # body: why THIS upstream commit
```

A base branch with none of our commits on it is the point: the PR diff is then exactly the patch.
**Merge with rebase** (or fast-forward locally and push) — squashing rewrites the commit, losing
the upstream author and their `Signed-off-by` and breaking the `patch-id` equality with upstream
that is the whole audit argument; a merge commit leaves the branch head no longer "upstream tag
plus one upstream commit". Then re-check the `patch-id` on the merged head, tag it, and dispatch:

```bash
git switch release/26.6.4-inform && git pull --ff-only
git show HEAD | git patch-id --stable            # same id as before the merge
git tag 26.6.4-inform.1 && git push origin 26.6.4-inform.1
```

Then dispatch with `tag: 26.6.4-inform.1` and no `image-tag`.

CI on the PR is free (the fork is public) and is the regression evidence for the cherry-picks —
see **CI on backport branches** below for which checks to read and which are known false positives.

### CI on backport branches

Six workflows can fire on a `fix/**` or `release/**` branch; everything else in
`.github/workflows/` is `workflow_dispatch`/`schedule`/`workflow_call` only, and `Weblate Sync` is
filtered to `main`. Three of the six are **disabled in the fork's Actions settings** — a
repository setting, deliberately not a file edit, so it creates no upstream merge conflict:

| Workflow                 | State        | Why                                                       |
|--------------------------|--------------|-----------------------------------------------------------|
| `Keycloak CI`            | enabled      | All server-side coverage; the only checks that matter here |
| `Keycloak Operator CI`   | enabled      | Relevant if a backport touches `operator/`                |
| `CodeQL`                 | enabled      | Static analysis of the backported code (push event only)  |
| `Keycloak JavaScript CI` | **disabled** | Cannot pass on a backport branch — see below              |
| `Keycloak Documentation` | **disabled** | Asciidoc only; nothing reaches the image, and its `External links check` fails on flaky live HTTP |
| `Keycloak Guides`        | **disabled** | Same                                                      |

Re-enable with `gh workflow enable "<name>" --repo Inform-Software/keycloak`.

**Why JS CI can never pass here.** `js-ci.yml` hardcodes `keycloak-999.0.0-SNAPSHOT.tar.gz` in
eight places — the `mv` at :58, the artifact path, and the `tar xfvz` plus `kc.sh` path in each of
the three E2E jobs. Upstream never trips over it because CI only ever runs on trees whose pom is
`999.0.0-SNAPSHOT`: `release/26.x` branches stay at the snapshot version, and the stamped pom
exists only on the `Set version to X` tag commits, which upstream never builds. A backport branch
starts *at* such a tag, so its pom is a real version and the `mv` fails. The three E2E jobs
(Admin UI, Account UI, admin-client against a live server) then never run — they report `skipped`,
so that coverage was never there to lose. What disabling does cost is four jobs that do work on a
stamped tree: `Admin Client`, `UI Shared`, `Account UI` (lint + build) and `Admin UI` (lint + unit
tests + build). Those only matter for a cherry-pick that touches `js/`, `themes/` or
`rest/admin-ui-ext/` — the fork has done one (`fix/45333-user-admin-events-ui-fix`). For such a
backport, either re-enable JS CI for that PR or run the equivalent locally:
`pnpm -C js install && pnpm -C js/apps/admin-ui lint && … test && … build`.

**Which checks to read.** The **pull_request**-event `Keycloak CI` run, and inside it:

- `Forms IT (chrome)` / `Forms IT (firefox)` — `forms-suite` is `org.keycloak.testsuite.forms.**`,
  so this is where `ResetPasswordTest` and friends actually run. **Not** `Base IT`, whose
  `base-suite` excludes that package.
- `Base IT (1..6)` — broad arquillian coverage; the best signal for collateral damage from a
  cherry-pick that touches shared code.
- `Testsuite Deprecation Check` — passes as long as the pick adds no new file under `testsuite/`
  and under 100 lines to any single one.
- `Status Check - Keycloak CI` — the aggregate; `if: always()` over 25 job groups.

**Known false positive: `Keycloak CI / SSSD` on push events.** `conditional.sh` only diffs for
refs matching `refs/pull/N/merge`; for anything else it prints "Not a pull request, marking
everything as changed" and forces every conditional job on. So a push to a backport branch runs
`SSSD`, which needs a FreeIPA server plus the weekly `ipa-data-<year>-<week>` cache that only
upstream's own runs populate, and fails in its `Run tests` step after ~2 minutes. It drags
`Status Check - Keycloak CI (push)` red with it. Nothing to fix fork-side; ignore it unless a
backport actually touches `federation/sssd/`, in which case test manually. The flip side of the
same behaviour is that the push run exercises groups the PR run skips (store model, volatile
sessions, external Infinispan, Quarkus UT, admin v2, cluster compatibility), so it is worth
reading despite the red aggregate.

**Do not put required status checks in a branch ruleset.** The push and pull_request runs publish
check runs with the *same name* on the *same SHA*, so a required `Status Check - Keycloak CI`
resolves nondeterministically to whichever finished last — and the fork's PAT gets
`403 Resource not accessible` on the Checks API, so a stuck merge is hard to diagnose. Gate on a
human reading the PR-event run instead.

**Do not run `set-version.sh`** — leave the pom at the upstream version. The image is scanned by
Amazon Inspector via ECR, which matches `org.keycloak:*` jar coordinates against advisory ranges;
a non-standard qualifier orders ambiguously (semver puts a prerelease *below* `26.6.4`, Maven's
`ComparableVersion` puts an unknown qualifier *above* it), so stamping can re-open every advisory
fixed in exactly `26.6.4` as a fresh finding. Canonical coordinates keep the report identical to
stock upstream plus the one backported CVE, which is a false positive that must be suppressed
either way — a backported fix is invisible to SBOM scanning regardless of the version string.

Identity therefore lives outside the jars: the git tag, the image tag, and the `version` /
`org.opencontainers.image.version` / `.revision` labels the workflow writes from the dispatch
input. That is what the consumer expects — its `Dockerfile` documents `KEYCLOAK_VERSION` (base
image tag) diverging from `KEYCLOAK_EXTENSIONS_VERSION` on a backport base, and sets the
admin-console footer string itself via `<keycloak.base.version>`. The accepted cost is that
`kc.sh --version` inside the container reads the plain upstream version, so patched vs stock is
told apart only by tag and labels.

The unstamped tarball is `keycloak-<version>.tar.gz` (`dist.archive.{file,dir}.version` default to
`${project.version}`), which the workflow's `cp quarkus/dist/target/keycloak-*.tar.gz` and the
Dockerfile's `tar -xvf … keycloak-*.tar.gz` / `mv /tmp/keycloak/keycloak-*` handle as globs.

Naming: reuse upstream's version string when the build is identical to upstream, otherwise
`-inform.<n>`. Tags `22.0.1-INFORM-1` and `22.0.3-INFORM-1` predate this convention and were
pom-stamped; they stay as they are.

### Constraints and traps in the publish workflow

- **Tags only**, though nothing enforces it. A branch name containing `/` is illegal in a Docker
  tag and fails at the push step *after* the ~20-minute build; a slash-free branch name is worse
  still — it succeeds and publishes a mutable, mislabelled image.
- **`linux/amd64` only** — a consequence of the `ubuntu-latest` runner, not an explicit pin: no
  `platforms:` key is set, and none is wanted, because the consumer pins
  `ARG IMAGE_PLATFORM=linux/amd64`.
- **Local composite actions resolve against the checked-out tag, not `main`.** This is
  load-bearing: `java-setup`'s default JDK is 17 on 22.0.2+, 21 on 26.0–26.5, 25 on 26.6+, so
  each tag builds on its own era's JDK for free. The flip side: `java-setup` does not exist
  before 22.0.2, so `22.0.1-INFORM-1` cannot be dispatched as-is — it dies at `Setup Java` with
  "Can't find action.yml".
- For the same reason, **do not replace the inlined build steps with
  `uses: ./.github/actions/build-keycloak`** and **do not add `pnpm-store-cache`**. The former
  differs per tag (22.x's copy uses the retired `actions/upload-artifact@v3`); the latter does not
  exist before 26.0.6, and a missing local action is a fatal step error. There is a comment in the
  workflow saying so — leave it there.
- `set-version.sh` is not part of the flow any more (see Case B), so its `ENV KEYCLOAK_VERSION`
  sed on `quarkus/container/Dockerfile` never fires here. It would be a no-op anyway from 26.2.0
  on, when that Dockerfile switched to `ARG KEYCLOAK_VERSION=999.0.0-SNAPSHOT`, and the workflow
  passes `--build-arg KEYCLOAK_VERSION` explicitly regardless. Left alone on purpose — it is an
  upstream-owned file.

## Building

Use `./mvnw`, never system Maven. JDK 17, 21 or 25 (compiler release is 17).

| Goal                          | Command                                                                     |
|-------------------------------|-----------------------------------------------------------------------------|
| Everything, no tests          | `./mvnw clean install -DskipTests`                                          |
| Server distribution only      | `./mvnw -pl quarkus/deployment,quarkus/dist -am -DskipTests clean install`   |
| One module, fast              | `./mvnw install -Pdistribution -DskipTests -DskipExamples -DskipTestsuite -DskipAdapters -DskipDocs -pl <module> -am` |
| Formatting check / apply      | `./mvnw -Pdocs,distribution,operator spotless:check` (`spotless:apply`)      |
| Operator (excluded by default)| add `-Poperator`                                                            |

The distribution lands in `quarkus/dist/target/`. Add `-DskipProtoLock=true` if the
proto-schema-compatibility check fails behind a proxy.

CI's canonical recipe is `.github/actions/build-keycloak` — the licenses-processor plugin must be
installed first or the main build emits warnings:

```bash
./mvnw install -Pdistribution -am -pl distribution/maven-plugins/licenses-processor
./mvnw install dependency:resolve -V -e -DskipTests -DskipExamples \
  -DexcludeGroupIds=org.keycloak -Dsilent=true -DcommitProtoLockChanges=true
```

## Testing

New tests go under `tests/` (Keycloak Test Framework). `testsuite/` is deprecated — see
`testsuite/DEPRECATED.md`; do not add tests there.

- Single test: `./mvnw test -pl tests/base -Dtest=MyProviderTest`
- Which modules have unit tests: `.github/scripts/find-modules-with-unit-tests.sh`
- Guides: `tests/docs/README.md`, `test-framework/docs/`, `docs/tests.md`, `docs/tests-db.md`

## Frontend

`js/` is a PNPM workspace (Node 24+). `pnpm install` from `js/`;
`pnpm -C apps/admin-ui lint`; `pnpm build` for the whole workspace. The Java build drives these
via frontend-maven-plugin, so a plain `./mvnw install` also builds the UIs.

## Layout

Server runtime is `quarkus/` (`quarkus/server` runs the Quarkus augmentation that pre-bakes
`kc.sh build`; `quarkus/dist` assembles the tar.gz/zip). Domain logic is in `services/`, `model/`,
`server-spi*`, `core/`. The container build context is `quarkus/container/` — see its `README.md`
for the three ways to build the image locally, and `ubi-null.sh`, which carves the minimal UBI
rootfs (runtime JDK is **21**, even though CI builds with 25).

## Conventions

Conventional-commit subjects (`chore:`, `fix:`, `feat:`) and DCO sign-off (`git commit -s`) — see
the fork's own commits. Upstream expects every PR to map to an issue and to keep a focused scope.

## Finding things in the code

This is a large monorepo; the fastest route to most questions is the table below rather than a
repo-wide grep.

| Looking for                                   | Start here                                                                 |
|-----------------------------------------------|----------------------------------------------------------------------------|
| An Admin REST endpoint                        | `services/src/main/java/org/keycloak/services/resources/admin/`             |
| An Account REST endpoint                      | `services/.../services/resources/account/`                                 |
| Login / registration / logout HTTP handling   | `services/.../services/resources/LoginActionsService.java`, `RealmsResource.java` |
| Authentication flow engine                    | `services/.../authentication/` (`AuthenticationProcessor`, `DefaultAuthenticationFlow`) |
| A specific authenticator or required action   | `services/.../authentication/authenticators/`, `.../requiredactions/`      |
| OIDC / SAML endpoints, token issuance, mappers| `services/.../protocol/oidc/`, `.../protocol/saml/`                        |
| Identity brokering / social login             | `services/.../broker/` (`oidc`, `saml`, `oauth`, `provider`)               |
| A `kc.sh` / `--option` config option          | `quarkus/config-api/src/main/java/org/keycloak/config/*Options.java` (24 files), wired in `quarkus/runtime/.../configuration/` |
| A DB table or column                          | entity in `model/jpa/src/main/java/org/keycloak/models/jpa/entities/`, migration in `model/jpa/src/main/resources/META-INF/jpa-changelog-*.xml` |
| Caching / user + auth sessions                | `model/infinispan/`                                                        |
| LDAP / Kerberos / SSSD user federation        | `federation/{ldap,kerberos,sssd,ipatuura}/`                                |
| A provider interface (SPI) to implement       | `server-spi/`, `server-spi-private/` under `org/keycloak/{models,storage,provider,sessions,credential,userprofile}` |
| Every implementation of an SPI                | grep `META-INF/services/<factory FQN>` files, not `implements` — that is how providers are discovered (`services/src/main/resources/META-INF/services/` alone holds 95) |
| A string shown on a login page                | `themes/src/main/resources/theme/base/login/messages/messages_en.properties`, then the matching `*.ftl` |
| Admin console UI                              | `js/apps/admin-ui/src/`, split by console section (`clients`, `realm-settings`, `user-federation`, `identity-providers`, `authentication`, `events`, `sessions`, `organizations`) |
| Typed admin REST client (TS)                  | `js/libs/keycloak-admin-client/`                                           |
| Crypto / FIPS                                 | `crypto/{default,fips1402,elytron}/`                                       |
| Authorization services (UMA, policies)        | `authz/`, `services/.../authorization/`                                    |

Module roles at a glance: `services/` holds nearly all runtime logic; `server-spi*` defines the
plug-in contracts; `model/` is persistence and caching; `core/` and `common/` are shared
value types and utilities; `quarkus/` is the runtime wiring (`runtime` + `deployment` +
`config-api`, then `server` → `dist` → `container`); `themes/` is server-rendered UI; `js/` is
the React consoles.
