# h2loop security fixes (fork of joernio/joern)

This fork removes the HIGH/CRITICAL vulnerabilities Trivy flagged in the Joern
layer of the `code-ingestion` image. There are three parts: vulnerable **bundled
transitive JVM deps**, the prebuilt **Go `goastgen` binary**, and the fact that
the upstream **container image never builds from source** at all.

Rebased onto upstream `master` on 2026-07-31.

## 1. Bundled transitive JVM deps (`dependencyOverrides` in `build.sbt`)

These get bundled into `joern-cli/lib` but are declared nowhere in Joern's build,
so the fix is a `ThisBuild / dependencyOverrides` block. Each bump stays within
the same major/API line (drop-in patch/minor), and each is the latest release in
its line as of 2026-07-31.

| Package | Was | Now | Notes |
|---|---|---|---|
| com.google.protobuf:protobuf-java | 3.20.1 & 3.21.8 | 3.25.9 | stays on 3.x, avoids the 4.x break; ~43 HIGH |
| io.undertow:undertow-core | 2.3.18.Final | 2.3.26.Final | CVE-2025-12543 (CRITICAL) + 3 HIGH |
| org.bouncycastle:bcprov-jdk18on | 1.78 | 1.85 | CVE-2025-14813 (CRITICAL) |
| org.msgpack:msgpack-core | 0.9.1 | 0.9.12 | 15 HIGH |
| com.squareup.okhttp3:okhttp | 4.7.2 | 4.12.0 | 1 HIGH |
| commons-io:commons-io | 2.11.0 | 2.16.0 (`Versions.commonsIo`) | raises older transitives to upstream's own pin |
| org.codehaus.plexus:plexus-utils | 3.2.1 | 3.6.1 | 1 HIGH |

## 2. Native Go astgen — rebuilt with a newer Go

The largest bucket (~70 HIGH + 1 CRITICAL, reported as Go `stdlib v1.21.12`, e.g.
CVE-2025-68121) came from the prebuilt `goastgen` binary (`gosrc2cpg`), which
upstream compiled with go1.21.12. Both upstream releases (v0.1.0 and v0.1.1) use
that same old toolchain, so a version bump alone does nothing.

Fix: `goastgen` was rebuilt from source with **go1.25.6** (same source, hardened
toolchain) for all platforms and published as `h2loop/astgen-monorepo` release
**`go-astgen/v0.1.1-h2loop1`**. This fork pulls that build:
- `gosrc2cpg/src/main/resources/application.conf`: `goastgen_version` -> `0.1.1-h2loop1`
- `gosrc2cpg/build.sbt`: `goAstGenDlUrl` -> `h2loop/astgen-monorepo`

Source change (go directive floor -> 1.24.13) lives on
`h2loop/astgen-monorepo@security/goastgen-go-bump`. Upstream has published no
newer `go-astgen` release since (checked 2026-07-31), so this is still current.

The other astgen binaries (jssrc2cpg=Node, csharpsrc2cpg=.NET, swiftsrc2cpg=Swift,
rust2cpg=Rust, ruby=gem, php-parser=PHP) are not Go and were not flagged; they are
already at their latest upstream versions.

## 3. The image must be built from source (`ci/Dockerfile.h2loop`)

**This is why the fixes above previously had no effect.** Upstream's
`ci/Dockerfile.alma` -- the recipe behind `ghcr.io/joernio/joern` -- does not
compile Joern. It downloads the prebuilt `joern-cli.zip` from the latest
`joernio/joern` GitHub release via `joern-install.sh`. A fork can change
`build.sbt` all it likes and the published image will never reflect it.

`ci/Dockerfile.h2loop` is a two-stage build that fixes this:
- **builder**: JDK 21 + sbt, runs `sbt joerncli/Universal/packageBin` against this
  working tree. Every frontend's native helper is *downloaded* as a prebuilt
  release asset by sbt, so no Rust/Ruby/Node/Swift toolchain is needed.
- **runtime**: same `almalinux/9-minimal` base, same package set, same
  `ENV`/user/`WORKDIR`/`CMD` as `ci/Dockerfile.alma`, plus a `microdnf update` so
  OS packages land on current patch levels. The zip's top-level directory is
  `joern-cli`, so unzipping into `/opt/joern` reproduces upstream's
  `/opt/joern/joern-cli` layout exactly.

That layout matters: `code-ingestion-service` resolves frontends as
`dirname($JOERN_CLI_PATH)/<tool>` with `JOERN_CLI_PATH=/opt/joern/joern-cli/joern`,
so the image is a drop-in replacement for `ghcr.io/joernio/joern`.

Published to ECR as
`501235162365.dkr.ecr.us-east-1.amazonaws.com/hydron-platform/joern:hardened` by
`.github/workflows/h2loop-image.yml` (linux/amd64 only -- all first-party h2loop
images are amd64). It authenticates with the same GitHub OIDC pattern as
`h2loop/hydron-cli`, so the repo needs:

| kind | name | value |
|---|---|---|
| secret | `AWS_ROLE_ARN` | role trusting `repo:h2loop/joern:*`, with ECR push rights on `hydron-platform/joern` |
| var | `AWS_REGION` | `us-east-1` |

**No such role exists yet.** The account's only GitHub-OIDC roles are
`h2loop-github-deploy` (trusts `repo:h2loop/platform:ref:refs/heads/release_*`) and
`h2loop-litellm-github-deploy` (scoped to `litellm_gcp_cloud_run`), so one of them
needs `repo:h2loop/joern:*` added to its trust policy -- or a new role -- by someone
with IAM write access. `DeveloperAccess` cannot do it.

Until then the workflow still builds, gates and scans; instead of pushing it
exports the image as a `joern-image` artifact so it can be loaded and pushed to ECR
with a developer's own SSO credentials. The ECR repository
`hydron-platform/joern` already exists (created 2026-07-31, `scanOnPush=true`).

The workflow builds, then **gates on**:
1. all 12 frontend entrypoints `code-ingestion` invokes being present and executable,
2. `c2cpg.sh` actually producing a CPG,
3. no known-vulnerable version of an overridden dependency surviving *anywhere* in
   the distribution (requested != bundled),

then Trivy-scans and pushes.

### On `bcprov` and `okhttp`

Run 30579709736 found **no `bcprov` or `okhttp` jar of any version** in
`joern-cli/lib`. `dependencyOverrides` only pins what is actually in the dependency
graph, so an override for something upstream does not pull in is a no-op -- it does
not invent a jar. Two possibilities, and the rebuilt gate distinguishes them:
either those jars live in one of the *other* ~15 lib directories (each frontend's
staged output is mapped in under its own subdirectory by `joern-cli/build.sbt`, so
the first version of this check was looking in one of fifteen places), or the
original Trivy findings came from somewhere other than the Joern distribution and
the overrides for them were never load-bearing.

Either way the correct assertion is the absence of the flagged versions anywhere in
the tree, not the presence of specific new ones.

## Building / verifying locally

Needs ~8 GB of RAM for the builder (`.sbtopts` asks for a 4 GB heap alone), so a
Docker VM with 4 GB will OOM -- use CI or a bigger machine.

```
docker build -f ci/Dockerfile.h2loop -t joern-h2loop:local .
trivy image --severity HIGH,CRITICAL --ignore-unfixed joern-h2loop:local
```

## Status

- [x] Overrides resolve (all seven versions confirmed present on Maven Central)
- [x] Image builds from source; all 12 frontends present; `c2cpg.sh` produces a CPG
      (run 30579709736)
- [x] protobuf 3.25.9, undertow 2.3.26.Final, msgpack 0.9.12, commons-io 2.16.0 and
      plexus-utils 3.6.1 confirmed bundled in `joern-cli/lib`
- [x] Rebased onto upstream `master` (2026-07-31), no conflicts
- [x] Image build path that actually carries the fixes
- [ ] CI run green (build + smoke + bundled-version gate + scan) -- **not yet run**
- [x] ECR repo `hydron-platform/joern` created (us-east-1, scanOnPush)
- [x] `AWS_REGION` var set on this repo
- [ ] `AWS_ROLE_ARN` secret + IAM trust policy for `repo:h2loop/joern:*` -- **needs IAM write access**
- [ ] Image in ECR (bootstrap via the exported tarball until the role exists)
- [ ] Verified on the dev cluster (see caveat below)
- [ ] `code-ingestion-service/Dockerfile` `FROM` flipped to the new image by digest

### Caveat for the dev-cluster test

`eks-dev` pulls `code-ingestion` from **GCP Artifact Registry**
(`asia-south1-docker.pkg.dev/h2loop-dev/saas-dev-images/code-ingestion`), and that
image's build pipeline authenticates to GCP via Workload Identity Federation only
-- it holds no AWS credentials. So a `FROM` pointing at a private ECR repo will
fail in that build. Before the dev test, one of these has to happen:

1. mirror the hardened image to GAR as well (dev builds pull from GAR, ECR stays
   the canonical copy for `eks-dev` core images and onprem), or
2. add AWS OIDC credentials to `code-ingestion-service`'s build workflow, or
3. rebuild `code-ingestion` by hand for the dev test.
