# Porting notes: Minecraft 1.21.11

This branch is a **config-only, unverified** port of Veil (from the 1.21.1
baseline) to target Minecraft **1.21.11**. It updates build coordinates
(Minecraft/Fabric/NeoForge/mapping/compat-mod versions) but **has not been
compiled**, because this environment's network egress policy blocks every
host the build needs (`maven.fabricmc.net`, `maven.neoforged.net`,
`libraries.minecraft.net`, `piston-meta.mojang.com`, `repo.spongepowered.org`,
`maven.parchmentmc.org`, `maven.blamejared.com`, `maven.su5ed.dev`). Only
Maven Central was reachable, so none of the Minecraft/Loom/NeoForge
toolchain could be resolved here, let alone built against.

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
