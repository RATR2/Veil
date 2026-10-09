# Porting notes: Minecraft 1.21.11

This branch is a port of Veil (from the 1.21.1 baseline) to target Minecraft
**1.21.11**. The build config was originally written without being able to
compile anything, because this dev sandbox's network policy blocks every
host the build needs (`maven.fabricmc.net`, `maven.neoforged.net`,
`libraries.minecraft.net`, `piston-meta.mojang.com`, `repo.spongepowered.org`,
`maven.parchmentmc.org`, `maven.blamejared.com`, `maven.su5ed.dev`) - only
Maven Central was reachable there. A real `./gradlew build` run on GitHub
Actions (which *can* reach those hosts) has since confirmed one real
architecture issue and fixed it - see "Confirmed via CI" below.

## Confirmed via CI

A real CI build failed at configuration time in the `neoforge` subproject:
`Tried to add a dependency to configuration ':neoforge:additionalRuntimeClasspath',
but there is no additional classpath anymore for Minecraft 1.21.11. Add the
dependency to a standard configuration such as implementation or
runtimeOnly.` ModDevGradle's `additionalRuntimeClasspath` helper no longer
exists starting with Minecraft 1.21.11. Fixed in `neoforge/build.gradle` by
dropping that wrapper from the `jarJar(api(...))` calls and depending
directly.

The same CI run got the `fabric` subproject through configuration (using
the `net.fabricmc.fabric-loom-remap` plugin, correct for 1.21.11 - it's
still obfuscated, unlike 26.1+) with only non-fatal-looking warnings
(`Failed to parse fabric.mod.json`, a block of `Cannot remap ... because it
does not exist in any of the targets [...]` lines). The build never reached
fabric's own build tasks before failing in `neoforge`, so those warnings
haven't been chased down yet - watch the next CI run.

## What changed

- `gradle.properties`: `minecraft_version`, `minecraft_version_range`,
  `fabric_version`, `fabric_loader_version`, `neoforge_version`,
  `neoforge_version_range`, `sodium_version` updated to best-available values
  found via web search (not verified against the authoritative Maven
  metadata - see TODOs inline).
- `common/build.gradle`, `fabric/build.gradle`, `neoforge/build.gradle`: the
  Parchment mappings overlay is disabled (commented out) because no 1.21.11
  Parchment release could be confirmed to exist. Falls back to official
  Mojang mappings only.
- `fabric/build.gradle`, `neoforge/build.gradle`: optional compile-time compat
  dependencies (Iris, ReplayMod, Flashback, CameraTweaks, Forgified Fabric
  API) have inline `PORT TODO` comments; Iris was bumped to the best version
  found (1.10.6), the others could not be confirmed for 1.21.11 and are left
  as-is (stale, will almost certainly fail to resolve).

## What was *not* attempted

No source-level API migration was attempted. Minecraft's renderer internals
changed substantially between 1.21.1 and later 1.21.x releases (notably the
`RenderPipeline`/`RenderType` rework), and Veil hooks deeply into those
internals via mixins across `common`, `fabric`, and `neoforge`. Expect real
compile errors in the mixin classes once a toolchain can actually resolve -
this port only gets the build *configuration* pointed at the new version.

## To finish this port

1. From a machine/CI with normal network access, verify every value flagged
   `PORT TODO` against the real Maven/Fabric-meta/NeoForge metadata.
2. Run `./gradlew build` and work through compile/mixin errors, particularly
   around rendering internals that changed since 1.21.1.
3. Delete this file (or fold its remaining caveats into the changelog) once
   the port is verified working.
