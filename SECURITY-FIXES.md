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
| com.google.protobuf:protobuf-java-util | 3.25.5 | 3.25.9 | separate artifact, found bundled at 3.25.5; kept in lockstep |
| io.undertow:undertow-core | 2.3.18.Final | 2.3.26.Final | CVE-2025-12543 (CRITICAL) + 3 HIGH |
| org.bouncycastle:bcprov-jdk18on | 1.78 | 1.85 | **inert** -- see below; fixed via `Versions.jRuby` instead |
| org.msgpack:msgpack-core | 0.9.1 | 0.9.12 | 15 HIGH |
| com.squareup.okhttp3:okhttp | 4.7.2 | 4.12.0 | 1 HIGH |
| commons-io:commons-io | 2.11.0 | 2.16.0 (`Versions.commonsIo`) | raises older transitives to upstream's own pin |
| org.codehaus.plexus:plexus-utils | 3.2.1 | 3.6.1 | 1 HIGH |

### bcprov: nested inside jruby-complete, not a resolved dependency

Trivy located the vulnerable copy: `bcprov-jdk18on 1.78` sits **inside**
`org.jruby.jruby-complete-9.4.9.0.jar`, which vendors its own BouncyCastle under
`META-INF/jruby.home/lib/ruby/stdlib/`. It is not a node in the sbt dependency
graph, so `dependencyOverrides` cannot touch it -- that override is inert (kept
only as a guard should bcprov ever become a real dependency), and a
`find -name "*.jar"` never saw it because it is a jar inside a jar.

The actual fix is a patch-level bump of `Versions.jRuby`, 9.4.9.0 -> 9.4.15.0,
which vendors bcprov/bcpkix **1.84** -- a listed fix version for CVE-2025-14813
(1.80.2, 1.81.1, 1.84). Verified by unzipping both jars and diffing the vendored
BouncyCastle. Staying on the 9.4 line avoids the 9.4 -> 10.x major jump.

## 2. Native Go astgen — rebuilt with a newer Go

The largest bucket (~70 HIGH + 1 CRITICAL, reported as Go `stdlib v1.21.12`, e.g.
CVE-2025-68121) came from the prebuilt `goastgen` binary (`gosrc2cpg`), which
upstream compiled with go1.21.12. Both upstream releases (v0.1.0 and v0.1.1) use
that same old toolchain, so a version bump alone does nothing.

**The first attempt at this did not work.** Release `v0.1.1-h2loop1` rebuilt the
binary with go1.25.6, but Trivy on the resulting image (run 30583692200) still
reported **13 findings (12 HIGH + 1 CRITICAL)** against it -- including
CVE-2025-68121 itself. That CVE is fixed in 1.24.13 and **1.25.7**; go1.25.6
predates the fix on its own line. Twelve further HIGH stdlib advisories landed
after 1.25.6. So the count went from ~70 to 13, not to zero.

Current fix: rebuilt with **go1.26.5**, published as
**`go-astgen/v0.1.1-h2loop2`**, which is at or above every listed fix version for
all 13. Verified independently: `trivy rootfs` on the new `goastgen-linux` reports
**0 vulnerabilities** (was 13). This fork pulls that build:
- `gosrc2cpg/src/main/resources/application.conf`: `goastgen_version` -> `0.1.1-h2loop2`
- `gosrc2cpg/build.sbt`: `goAstGenDlUrl` -> `h2loop/astgen-monorepo`

Source change lives on `h2loop/astgen-monorepo@security/goastgen-go-bump` (rebased
onto upstream `main`): `go-astgen/go.mod` now pins `toolchain go1.26.5` explicitly
rather than relying on the `go` directive floor plus whatever `GOTOOLCHAIN`
resolves to, and `go-astgen-release.yml` builds with 1.26.5. Upstream has published
no newer `go-astgen` release (checked 2026-07-31).

The h2loop2 binaries were cross-compiled locally (`CGO_ENABLED=0`, same ldflags as
the release workflow) because **GitHub Actions is not activated on the
`h2loop/astgen-monorepo` fork** -- a fork-level flag that can only be flipped in
the web UI, not through the API. Worth enabling so future rebuilds are reproducible
in CI.

The image workflow now *asserts* `go1.26.5` in the shipped binary rather than
printing the version, so a stale toolchain fails the build.

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

### Image size

Upstream `ghcr.io/joernio/joern` (amd64) is **2.63 GB compressed / ~5.1 GB
uncompressed** in 3 layers, almost all of it one 2596 MB layer.

The first revision of this Dockerfile came out ~1.3 GB heavier, because it did
`COPY --from=builder /dist/joern-cli.zip` and then `rm`-ed the zip in the *next*
`RUN`. A COPY is its own layer, so the delete only hid the data -- the zip stayed in
the image permanently. Upstream never hits this because `Dockerfile.alma` curls and
deletes the zip inside a single `RUN`.

The zip is now bind-mounted from the builder
(`RUN --mount=type=bind,from=builder,...`), so it never lands in a layer at all, and
the unzip plus `chown -R` share one layer so the tree is not duplicated either. CI
asserts a 6.5 GB uncompressed ceiling to catch this class of regression.

A `.dockerignore` also excludes `.github/`, `ci/` and root `*.md` from the build
context: the builder does `COPY . /src`, so without it editing a workflow or this
document invalidates the sbt layer and forces a ~25 minute recompile. `.git` is
deliberately kept, since sbt-dynver needs the tags to stamp a version.

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

The first gate looked only at `joern-cli/lib` and reported both missing.
`okhttp` was simply in a *different* lib directory -- the distribution has ~15,
one per frontend -- and the whole-tree scan on run 30581433009 found it correctly
overridden at 4.12.0.

`bcprov` is genuinely **absent from the distribution**: no jar of any version
anywhere under `/opt/joern`. `dependencyOverrides` only pins deps already in the
graph, so that override is an inert no-op, and the original CVE-2025-14813 finding
must have come from somewhere other than the Joern distribution (a `find -name
"*.jar"` also would not see jars nested inside a zip, e.g. the ghidra bundle).
The Trivy report is authoritative on this; the override is harmless either way.

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
