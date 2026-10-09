# Porting notes: Minecraft 26.3

This branch is a **config-only, unverified** port of Veil (from the 1.21.1
baseline) to target Minecraft **26.3**. It updates build coordinates
(Minecraft/Fabric/NeoForge/mapping/compat-mod versions) but **has not been
compiled**, because this environment's network egress policy blocks every
host the build needs (`maven.fabricmc.net`, `maven.neoforged.net`,
`libraries.minecraft.net`, `piston-meta.mojang.com`, `repo.spongepowered.org`,
`maven.parchmentmc.org`, `maven.blamejared.com`, `maven.su5ed.dev`). Only
Maven Central was reachable, so none of the Minecraft/Loom/NeoForge
toolchain could be resolved here, let alone built against.

## Why 26.3 is a bigger jump than it looks

Minecraft 26.3 is on Mojang's new `YY.minor` versioning scheme (no more
`1.21.x`), and both Fabric and NeoForge changed their own conventions to
match:

- **Java 25 is required** (26.1 and newer). `java_version` in
  `gradle.properties` is bumped from 21 to 25. The repo's Gradle wrapper is
  already 9.7.1, which satisfies NeoForge's documented Gradle 9.1+
  requirement for this line - no wrapper change needed.
- **NeoForge version strings are now 4 components**: the first three encode
  the Minecraft version (`26.3.0`), the fourth is the NeoForge release
  (e.g. `26.3.0.1-beta`). This is a different shape than the old
  `21.1.219`-style version used for 1.21.1.
- Mod/mapping ecosystem support for 26.3 is only a few weeks old as of this
  port (26.3 shipped mid-September 2026) and several dependencies (Fabric
  API, Parchment, NeoForge itself) are still beta or unpublished for it.

## What changed

- `gradle.properties`: `java_version` → 25; `minecraft_version`,
  `minecraft_version_range`, `fabric_loader_version`, `neoforge_version`,
  `neoforge_version_range`, `sodium_version` updated to best-available
  values found via web search (not verified against authoritative Maven
  metadata). `fabric_version` (Fabric API) could not be found at all and is
  left as an explicit placeholder string, not a guess - see TODO below.
- `common/build.gradle`, `fabric/build.gradle`, `neoforge/build.gradle`: the
  Parchment mappings overlay is disabled (commented out) because 26.3 is too
  new for a Parchment release to exist yet. Falls back to official Mojang
  mappings only.
- `fabric/build.gradle`, `neoforge/build.gradle`: optional compile-time compat
  dependencies (Iris, ReplayMod, Flashback, CameraTweaks, Forgified Fabric
  API) have inline `PORT TODO` comments; Iris-for-Fabric was bumped to the
  best version found (1.11.7+26.3), the others (including Iris-for-NeoForge)
  could not be confirmed for 26.3 and are left as-is (stale, will almost
  certainly fail to resolve).

## What was *not* attempted

No source-level API migration was attempted. Veil hooks deeply into
Minecraft's renderer internals via mixins across `common`, `fabric`, and
`neoforge`, and those internals changed substantially between 1.21.1 and
26.3 (notably the `RenderPipeline`/`RenderType` rework that landed in the
1.21.6-ish timeframe, well before 26.3). This port only gets the build
*configuration* pointed at the new version - expect the mixin classes to
need real rework once a toolchain can actually resolve.

## To finish this port

1. From a machine/CI with normal network access, resolve the real values for
   every `PORT TODO`, especially `fabric_version` (Fabric API for 26.3),
   which has no placeholder-free value in this commit at all.
2. Run `./gradlew build` with a JDK 25 toolchain available and work through
   compile/mixin errors, particularly around rendering internals.
3. Delete this file (or fold its remaining caveats into the changelog) once
   the port is verified working.
