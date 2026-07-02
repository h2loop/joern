# h2loop security fixes (fork of joernio/joern)

This fork force-upgrades vulnerable **transitive** dependencies that get bundled
into `joern-cli/lib` and were flagged HIGH/CRITICAL by Trivy when scanning the
code-ingestion image. None of these were declared directly in Joern's build, so
the fix is a `ThisBuild / dependencyOverrides` block in `build.sbt`.

## Fixed (JVM jars — via dependencyOverrides in build.sbt)

| Package | Was | Now | Notes |
|---|---|---|---|
| com.google.protobuf:protobuf-java | 3.20.1 & 3.21.8 | 3.25.5 | stays on 3.x, avoids the 4.x break; ~43 HIGH |
| io.undertow:undertow-core | 2.3.18.Final | 2.3.21.Final | CVE-2025-12543 (CRITICAL) + 3 HIGH |
| org.bouncycastle:bcprov-jdk18on | 1.78 | 1.81 | CVE-2025-14813 (CRITICAL) |
| org.msgpack:msgpack-core | 0.9.1 | 0.9.11 | 15 HIGH |
| com.squareup.okhttp3:okhttp | 4.7.2 | 4.9.2 | 1 HIGH |
| commons-io:commons-io | 2.11.0 | 2.16.0 | pin any older transitive |
| org.codehaus.plexus:plexus-utils | 3.2.1 | 3.6.1 | 1 HIGH |

All bumps stay within the same major/API line → drop-in patch/minor upgrades.

## Fixed (native Go astgen — rebuilt with a newer Go)

The largest remaining bucket (~70 HIGH + 1 CRITICAL, reported as Go `stdlib`
`v1.21.12`, e.g. CVE-2025-68121) came from the prebuilt `goastgen` binary
(`gosrc2cpg`), which upstream compiled with go1.21.12. Both upstream releases
(v0.1.0 and v0.1.1) use that same old toolchain, so a version bump alone does
nothing.

Fix: `goastgen` was rebuilt from source with **go1.25.6** (same source, hardened
toolchain) for all platforms and published as
`h2loop/astgen-monorepo` release **`go-astgen/v0.1.1-h2loop1`**. This fork now
pulls that build:
- `gosrc2cpg/src/main/resources/application.conf`: `goastgen_version` -> `0.1.1-h2loop1`
- `gosrc2cpg/build.sbt`: `goAstGenDlUrl` -> `h2loop/astgen-monorepo`
Source change (go directive floor -> 1.24.13) lives on
`h2loop/astgen-monorepo@security/goastgen-go-bump`.

Note: the other astgen binaries (jssrc2cpg=node, csharpsrc2cpg=.NET,
swiftsrc2cpg=Swift) are not Go and were not flagged; they're already at their
latest upstream versions.

## Verifying
```
sbt clean stage        # or: sbt createDistribution
# then scan the produced joern-cli, or rebuild code-ingestion against this fork
```
This change is not yet build-verified in this fork; CI must run `sbt stage` and
re-scan to confirm the resolved versions bundle correctly and nothing breaks.
