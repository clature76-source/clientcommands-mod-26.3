# Client Commands for Minecraft 26.3 — NeoForge port and Fabric add-on

Two unofficial builds of [ClientCommands](https://github.com/Earthcomputer/clientcommands) by
Earthcomputer, both for **Minecraft 26.3**.

| Work | Loader | Source | Ready-to-use jar |
|---|---|---|---|
| **NeoForge port** — the whole mod, moved to the NeoForge loader | NeoForge 26.3 | repository root | [releases](https://github.com/clature76-source/clientcommands-mod-26.3/releases) |
| **Fabric add-on** — charoite-aware enchantment prediction for Mielon's The Sift | Fabric | [`fabric-charoite/`](fabric-charoite/) | [`fabric-charoite/dist/`](fabric-charoite/dist/) |

Adds several useful client-side commands to Minecraft. All commands and features are the work of the
original project. This repository holds two pieces of work built on top of it — a port to another
loader, and a compatibility add-on for another mod. Each has its own section below.

**This is a private, fan-made project.** It exists for hobbyists who want to use ClientCommands on
NeoForge, or with The Sift on Fabric, and for no other purpose.

## Social

- Original project: https://github.com/Earthcomputer/clientcommands
- Upstream Discord: https://discord.gg/Jg7Bun7
- Upstream Patreon: https://www.patreon.com/earthcomputer

## Installation

### NeoForge port

1. Install [NeoForge for Minecraft 26.3](https://neoforged.net/) (developed against
   `26.3.0.51-beta`, tested on `26.3.0.46-beta`).
2. Download the mod jar from this repository's
   [releases](https://github.com/clature76-source/clientcommands-mod-26.3/releases)
   or build it yourself, and move it to the mods folder (`.minecraft/mods`).

Nothing else needs to be installed. **Fabric API is not required**: the Forgified Fabric API
modules and the Fabric-only libraries this mod uses are jar-in-jar bundled inside the mod jar.

### Fabric add-on

1. Install Fabric Loader 0.18.4 or newer, and
   [Fabric API](https://modrinth.com/mod/fabric-api) for Minecraft 26.3.
2. Install **Mielon's The Sift** (mod id `the_sift`), version 1.1.2 for MC 26.3. The add-on does
   nothing without it, but is harmless to leave installed.
3. Take `clientcommands-2.16.2-fabric-charoite.jar` from
   [`fabric-charoite/dist/`](fabric-charoite/dist/), or build it from
   [`fabric-charoite/fabric-charoite.patch`](fabric-charoite/fabric-charoite.patch), and move it to
   the mods folder.

`betterconfig`, `clientarguments` and `simplewaypoints` are bundled inside the jar.

## Requirements

| Work | Java | Game | Loader |
|---|---|---|---|
| NeoForge port | 25 | Minecraft 26.3 | NeoForge 26.3 |
| Fabric add-on | 25 | Minecraft 26.3 | Fabric Loader 0.18.4+ and Fabric API |

## Building

### NeoForge port

```
./gradlew clean jar
```

`clean` is not optional — see "Rebuilding after a build.gradle change" below.

The mod jar is written to `build/libs/clientcommands-<version>.jar`.

### Fabric add-on

`fabric-charoite/fabric-charoite.patch` applies to upstream's `fabric` branch at commit `5eb08cc`:

```
git clone https://github.com/Earthcomputer/clientcommands
cd clientcommands
git checkout fabric
git apply /path/to/fabric-charoite.patch
./gradlew build
```

## Disclaimer

- **Private project, hobby use only.** Both the port and the add-on are made by a fan, for fans. They
  are not affiliated with, endorsed by, or supported by Earthcomputer or the ClientCommands project,
  nor by the authors of any mod they interoperate with.
- **No commercial use is intended or supported by this work.** It is published for personal and
  hobby use. The author does not grant, and cannot grant, any commercial licence to the original
  work.
- **The author accepts no liability** for any damage, data loss, ban, or legal consequence arising
  from the use of this work. Use it at your own risk.
- **No warranty** of any kind is provided. See the licence below.

## Licence

Licensed under **LGPL-3.0-or-later**, the same licence as the original project (see `LICENSE`).

The original work is Copyright (c) 2021 Earthcomputer. This repository is a derivative work and is
distributed under the same terms. If you redistribute it, keep the licence and copyright notice
intact. No part of any third-party mod is redistributed here.

> On the disclaimer above: LGPL-3.0-or-later permits commercial use of the licensed work, and a
> derivative work cannot add a restriction the licence forbids. The "no commercial use" statement
> therefore describes the intent and support scope of *this work*, not a legal term of the licence —
> the licence itself governs what you may do with the code.

## How the NeoForge port works

This is not a rewrite: the original sources are kept as close to upstream as possible, and the
loader-specific layers are adapted around them.

- A NeoForge `@Mod` entry point (`ClientCommands`) mirrors Fabric's `ClientModInitializer`, and
  invokes the Fabric entry points of the embedded Fabric-only libraries, in the order the Fabric
  loader would use.
- Fabric API surface that has a working NeoForge port is used as-is and bundled with `jarJar`
  (Forgified Fabric API: `fabric-api-base`, `fabric-command-api-v2`, plus the Forgified Fabric
  Loader). Keeping them as nested jars is required: each is itself a NeoForge mod with its own
  `META-INF/neoforge.mods.toml`, so unpacking their classes while dropping that metadata makes
  NeoForge abort startup with "contains mod entrypoint class ... which does not exist".
- The parts of the Fabric API that cannot run on 26.3 (client lifecycle events, level rendering
  events, render state data) are implemented natively on top of the corresponding NeoForge events.
  Those implementations live under `net.fabricmc.fabric.api.client.*` so the original sources keep
  compiling unchanged.
- Command registration is bridged: the original commands are built on a Brigadier tree rooted at
  `FabricClientCommandSource`, and `compat/CommandTreeBridge` rebuilds that tree onto NeoForge's
  vanilla `CommandSourceStack` dispatcher, adapting the source at execution time. The parsed
  argument map is carried across the rebuild, otherwise every command that reads an argument fails
  with "No such argument ... exists on this command".
- The embedded libraries' entry points are invoked exactly once. `RegisterClientCommandsEvent`
  fires again for every command tree the server sends, and `ModConfigBuilder.build` throws if a
  config id is registered twice, so re-entering would disconnect the player.
- Access widening is done with a native NeoForge access transformer
  (`META-INF/accesstransformer.cfg`), converted from the original access widener.
- Mixin configs are declared explicitly in `META-INF/neoforge.mods.toml`; NeoForge does not
  discover them from `fabric.mod.json` the way Fabric does.

### Rebuilding after a build.gradle change

`unpackGameContent` is a Gradle `Copy` task, which only adds files. Classes unpacked by a previous
build stay in `build/classes/java/main` and are packaged into the next jar even after the source of
that unpacking is removed. Always run `clean` after changing which libraries are unpacked.

## How the Fabric add-on works

This is a patch, not a port: the loader is unchanged and upstream's code is barely touched.

The Sift adds **charoite** as an alternative to lapis in the enchanting table's fuel slot. While
charoite is there, the mod raises every non-zero cost it reports by **20**, and raises the level of
every enchantment it generates whose maximum level is above 1 by **1**. The enchantments are still
rolled from the original cost — the +20 is display-only — but upstream ClientCommands knows nothing
about either effect, so with charoite in the slot the seed crack stops matching and `/cenchant`
reports enchantments one level below what the player receives.

The add-on mirrors both effects inside ClientCommands' own simulation:

- Charoite is looked up by item id (`the_sift:charoite`) in `BuiltInRegistries.ITEM`, lazily. **The
  mod is never a hard dependency** — with The Sift absent the lookup yields nothing, every
  conversion is the identity, and behaviour is exactly upstream's.
- Costs convert as displayed = generated + 20 and generated = `max(1, displayed − 20)`, matching the
  mod's own conversion including its clamp; a zero cost stays zero.
- Levels are raised by 1 only where the enchantment's maximum level is above 1, matching the mod's
  own condition.
- All three prediction paths are updated — the seed brute force, the enchanting-screen overlay, and
  the `/cenchant` search — and `/cenchant` accepts one level above the vanilla maximum so a boosted
  level can be typed at all.

See [`fabric-charoite/README.md`](fabric-charoite/README.md) for the file list and the verification
status.

## Known limitations

- **NeoForge runtime verification is incomplete.** The build succeeds, the jar structure is correct,
  and the failures recorded in the commit history have been fixed, but not every command has been
  exercised in a live game. Please report problems together with a log.
- **Fabric add-on runtime verification is incomplete.** The build, the jar contents, the
  no-mod-is-installed behaviour and the cost/level conversions are all verified, but enchanting with
  charoite has not been exercised in a live game.
- **`src/test` is not part of the build.** The test sources need test-only dependencies (ASM, Guava,
  jspecify) that the NeoForge build does not declare yet. `src/main` compiles cleanly.
- **`fabric-rendering-v1` is not bundled.** A few Fabric API types from that module are provided by
  hand-written sources under `src/main/java/net/fabricmc/`, because bundling the module would add
  mixins with more risk than benefit. Functionally equivalent, but not the upstream implementation.
- **Merging upstream is manual.** When the upstream `fabric` branch moves, the compatibility layer
  under `compat/` has to be re-merged, and the Fabric add-on patch has to be rebased onto it. The
  command bodies themselves are almost untouched, so the cost is contained.

## Contributing

Bug reports and fixes for the port or the add-on are welcome. Changes to the mod's features or
commands belong upstream: https://github.com/Earthcomputer/clientcommands

To contribute translations, see the upstream
[translation contribution guidelines](docs/TRANSLATING.md).
