# Porting notes: Minecraft 26.3

This branch is a port of Veil (from the 1.21.1 baseline) to target Minecraft
**26.3**. The build *configuration* has now been exercised against real CI
(GitHub Actions, which can reach the Minecraft/Fabric/NeoForge Maven hosts
that this dev sandbox's network policy blocks), and two real, confirmed
architecture changes have been applied as a result - see "Confirmed via CI
and primary sources" below. Source-level mixin/API compatibility has still
not been verified end-to-end; see "What was *not* attempted".

## Why 26.3 is a bigger jump than it looks

Minecraft 26.3 is on Mojang's new `YY.minor` versioning scheme (no more
`1.21.x`), and both Fabric and NeoForge changed their own conventions to
match:

- **Minecraft ships unobfuscated from 26.1 onward.** This is the big one:
  Fabric Loom dropped its remapping step entirely for this line. See below.
- **Java 25 is required** (26.1 and newer). `java_version` in
  `gradle.properties` is bumped from 21 to 25. The repo's Gradle wrapper is
  already 9.7.1, which satisfies NeoForge's documented Gradle 9.1+
  requirement for this line - no wrapper change needed.
- **NeoForge version strings are now 4 components**: the first three encode
  the Minecraft version (`26.3.0`), the fourth is the NeoForge release
  (e.g. `26.3.0.1-beta`). This is a different shape than the old
  `21.1.219`-style version used for 1.21.1.
- Mod/mapping ecosystem support for 26.3 is only a few weeks old as of this
  port (26.3 shipped mid-September 2026) and several dependencies (Parchment,
  NeoForge itself) are still beta or unpublished for it.

## Confirmed via CI and primary sources

A real `./gradlew build` run on GitHub Actions failed at configuration time
with: `Could not find fabric-api version: UNKNOWN-VERIFY-FABRIC-API-FOR-26.3`
(the placeholder this port originally left). Chasing that down turned up a
much bigger issue than a missing version number:

- **Fabric Loom has no remapping step for Minecraft 26.x.** Confirmed by
  cloning `FabricMC/fabric-loom` directly (latest release at port time:
  `v1.18.3`) and reading its source/test fixtures:
  - The plugin id to use is `net.fabricmc.fabric-loom` (→
    `LoomNoRemapGradlePlugin`), not `net.fabricmc.fabric-loom-remap` (→
    `LoomRemapGradlePlugin`, for 1.21.11 and older, which is "the last
    obfuscated version").
  - No `mappings { ... }` block at all - there's nothing to map.
  - Mod dependencies use plain `implementation`/`compileOnly`, not
    `modImplementation`/`modCompileOnly` (confirmed against Loom's own
    `UnobfFabricAPITest` fixture).
  - There's no `remapJar` task; the normal `jar` task is the final artifact
    (`buildSrc/src/main/groovy/multiloader-loader.gradle` updated to match -
    it previously assumed fabric always has a `remapJar` task).
  - `fabric_version` is now a confirmed real value (`0.162.0+26.3`, from the
    Fabric API GitHub releases page), used as a single `implementation
    "net.fabricmc.fabric-api:fabric-api:${fabric_version}"` dependency rather
    than the old per-module `fabricApi.module(...)` calls, since it's
    unconfirmed whether per-module artifacts are still published under the
    unobfuscated build pipeline.
  - `settings.gradle` and root `build.gradle` updated to use the
    `net.fabricmc.fabric-loom` plugin/marker group instead of
    `net.fabricmc.fabric-loom-remap`.
- **NeoForge's `additionalRuntimeClasspath` helper is gone from Minecraft
  1.21.11 onward.** Also confirmed by a real CI failure (on the sibling
  `1.21.11` branch): `Tried to add a dependency to configuration
  ':neoforge:additionalRuntimeClasspath', but there is no additional
  classpath anymore for Minecraft 1.21.11. Add the dependency to a standard
  configuration such as implementation or runtimeOnly.` Fixed here too (same
  underlying ModDevGradle change, and 26.3 is well past 1.21.11) by dropping
  the `additionalRuntimeClasspath(...)` wrapper in `neoforge/build.gradle`'s
  `jarJar(api(...))` calls and depending directly.
- **`imguimc` resolved via a manually-built, manually-hosted Maven repo.**
  No official publish exists for 26.3 on either loader (checked via its own
  CI history - its `build` workflow passes on the `26.3` branch, but
  `release`, the one that actually publishes to Maven/Modrinth/CurseForge,
  is manual-trigger-only and all 24 recorded runs are on `main`). A JitPack
  build of that exact commit (`86d6fa3`, "Add 26.3 support") was tried next
  and also failed (generic "Build failed", no detailed log reachable) -
  likely this project's Stonecutter/Loom/ModDevGradle toolchain is more than
  JitPack's generic build environment handles. The working fix: built it on
  a real machine with real network access (`./gradlew :fabric:26.3:publishToMavenLocal`
  and the `:common:26.3` equivalent, run under a JDK 21/25 - the Gradle
  daemon itself can't run on JDK 27, a separate problem from the project's
  own Java 25 toolchain target), then pushed the resulting
  `~/.m2/repository/foundry/imguimc/...` tree to a `maven-repo` branch of
  `https://github.com/RATR2/imguimc` (a fork, since the upstream repo can't
  be pushed to). `buildSrc/.../multiloader-common.gradle` now points at
  `https://raw.githubusercontent.com/RATR2/imguimc/maven-repo/` for the
  `foundry.imguimc` group, which serves the exact same coordinates
  (`imguimc-{common,fabric}-26.3:2.0.5`) the build already expected - no
  dependency-declaration changes needed, just a different repository.
  NeoForge still has no imguimc support for 26.3 at all upstream (no
  `neoforge/versions/26.3` directory in source, not just no publish), so it
  stays disabled there. This whole dependency is genuinely optional either
  way - no source in `common/`, `fabric/`, or `neoforge/` imports the
  external `foundry.imguimc` package directly. Caveat: that `maven-repo`
  branch is a manually-pushed, one-off snapshot, not a maintained feed - if
  `imguimc_version` is ever bumped, it'll need to be rebuilt and re-pushed
  the same way, or (better, once it exists) swapped for a real upstream
  publish.
- **Minecraft 26.3 renamed/moved the entire rendering API** from
  `com.mojang.blaze3d.*` to a new `com.mojang.renderpearl.*` package
  (`api`/`backend.opengl`/`backend.vulkan` subpackages) - confirmed directly
  from that same imguimc commit's `stonecutter.gradle.kts`, which lists the
  exact old-path -> new-path mapping it applies for `current.parsed >= "26.3"`
  (covers buffers, pipeline, shaders/uniforms, vertex, textures, the OpenGL
  and a new Vulkan backend). **This is a big deal for Veil specifically**:
  152 of Veil's own source files import `com.mojang.blaze3d.*` directly.
  Several of those imports (`Uniform`, `Program`, `Shader`, `EffectProgram`,
  `ProgramManager`, `GlStateManager`) aren't in imguimc's rename list at all,
  meaning they were likely already replaced by the `RenderPipeline` system
  before 26.3 (matching the real `fabric:validateAccessWidener` errors seen
  while porting the sibling `1.21.11` branch, against those same classes).
  So Veil's 26.3 port has the same core blocker as 1.21.11, *plus* this
  additional package rename on top. Not attempted here - see "What was not
  attempted" below.

## Still unverified / best-effort

- `gradle.properties`: `minecraft_version_range`, `neoforge_version`,
  `neoforge_version_range`, `sodium_version` are best-available values found
  via web search, not verified against authoritative Maven metadata.
- `common/build.gradle`, `fabric/build.gradle`, `neoforge/build.gradle`: the
  Parchment mappings overlay is disabled (commented out) because 26.3 is too
  new for a Parchment release to exist yet, and because Mappings aren't
  meaningful anymore on the unobfuscated line anyway.
- `fabric/build.gradle`, `neoforge/build.gradle`: optional compile-time compat
  dependencies (Iris, ReplayMod, Flashback, CameraTweaks, Forgified Fabric
  API) have inline `PORT TODO` comments; Iris-for-Fabric was bumped to the
  best version found (1.11.7+26.3), the others (including Iris-for-NeoForge)
  could not be confirmed for 26.3 and are left as-is (stale, will almost
  certainly fail to resolve).

## What was *not* attempted

No source-level mixin/API migration was attempted. Veil hooks deeply into
Minecraft's renderer internals via mixins across `common`, `fabric`, and
`neoforge`, and those internals changed substantially between 1.21.1 and
26.3 (notably the `RenderPipeline`/`RenderType` rework that landed well
before 26.3, and the loss of obfuscation itself, which changes how mixin
targets and refmaps work under the hood even though Veil's own mixin source
files don't hardcode anything obfuscation-specific). Expect real compile
and/or mixin-apply errors once this reaches that point in CI.

## To finish this port

1. Watch the next CI run on this branch and work through whatever the build
   actually reports next - that loop (real error → fix → push → re-run) is
   far more reliable than guessing ahead of time, as this round showed.
2. Resolve the remaining `PORT TODO` values against authoritative sources
   once reached.
3. Delete this file (or fold its remaining caveats into the changelog) once
   the port is verified working.
