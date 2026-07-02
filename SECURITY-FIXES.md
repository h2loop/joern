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

## NOT fixed here (needs upstream / a newer astgen build)

The largest remaining bucket (~70 HIGH + 1 CRITICAL, reported as Go `stdlib`
`v1.21.12`, e.g. CVE-2025-68121) comes from the **prebuilt native `astgen`
binaries** that Joern downloads at build time (e.g. `goastgen` for `gosrc2cpg`),
which were compiled with an old Go toolchain. These are **not** controllable from
this sbt build — they require the upstream astgen repos (joernio/*astgen) to be
rebuilt with Go >= 1.24.13, or bumping the astgen binary versions once such a
release exists. Tracked separately.

## Verifying
```
sbt clean stage        # or: sbt createDistribution
# then scan the produced joern-cli, or rebuild code-ingestion against this fork
```
This change is not yet build-verified in this fork; CI must run `sbt stage` and
re-scan to confirm the resolved versions bundle correctly and nothing breaks.
