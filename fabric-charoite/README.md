# Fabric add-on: charoite-aware enchantment prediction

An add-on for [ClientCommands](https://github.com/Earthcomputer/clientcommands) on **Fabric** and
**Minecraft 26.3**, which teaches ClientCommands' enchantment prediction about the charoite fuel
added by **Mielon's The Sift** (mod id `the_sift`, version 1.1.2 for MC 26.3).

This is a patch on top of upstream's `fabric` branch, **not a port** — the loader is unchanged and
upstream's code is barely touched.

## The problem

ClientCommands can crack the enchanting table's XP seed and predict, for each of the three slots,
exactly which enchantments the table will produce: this is what `/cenchant` uses, and what the
overlay drawn on the enchanting screen shows.

The Sift adds **charoite** as an alternative to lapis in the fuel slot. While charoite is in that
slot, the mod does two things on the server:

1. it raises every non-zero cost it reports by **20**, so the GUI shows a higher level requirement;
2. it raises the level of every enchantment it generates whose maximum level is above 1 by **1**.

The enchantments themselves are still rolled from the original cost — the +20 is display-only — but
both changes are invisible to upstream ClientCommands. With charoite in the fuel slot:

- the seed crack never matches anything, because it compares its own simulated costs against the
  inflated costs and its simulated clue levels against the boosted clues. Prediction silently stops
  working and the cracker resets itself;
- `/cenchant`, if it finds anything at all, reports enchantments one level *below* what the player
  actually receives.

## What this add-on does

It mirrors those two effects inside ClientCommands' own simulation, and changes nothing else.

- **Detection is soft.** The mod is never a hard dependency. Charoite is looked up by item id
  (`the_sift:charoite`) in `BuiltInRegistries.ITEM`, lazily and once. If The Sift is not installed
  the lookup yields nothing, every conversion becomes the identity, and behaviour is exactly
  upstream's. There is no class or entry point referenced from the other mod.
- **Costs.** Displayed cost = generation cost + 20; generation cost = `max(1, displayed − 20)`; a
  zero cost stays zero. This matches the mod's own conversion, including its clamp to 1.
- **Levels.** Every enchantment whose maximum level is above 1 is raised by 1; single-level
  enchantments (Silk Touch, Infinity, …) are left alone. This matches the mod's own condition.
- **All three prediction paths** are updated: the seed brute force, the enchanting-screen overlay,
  and the `/cenchant` search. `/cenchant` additionally accepts one level above the vanilla maximum
  while the mod is present, so a boosted level (Sharpness VI, for example) can be typed at all —
  otherwise the command's own argument parser rejects it.
- A line is added to the overlay while charoite is in the fuel slot, so the state is visible rather
  than implicit.

## Files

| File | Change |
|---|---|
| `src/main/java/net/earthcomputer/clientcommands/features/CharoiteCompat.java` | **new** — detection, the two cost conversions, the level boost |
| `src/main/java/net/earthcomputer/clientcommands/features/EnchantmentCracker.java` | the three prediction paths, plus the overlay line |
| `src/main/java/net/earthcomputer/clientcommands/command/arguments/ItemAndEnchantmentsPredicateArgument.java` | allow one level above the vanilla maximum when the mod is present |
| `src/main/resources/assets/clientcommands/lang/en_us.json`, `zh_cn.json` | the overlay string |

`fabric-charoite.patch` is the complete diff against upstream `fabric` at commit `5eb08cc`.

## Building

Apply the patch to upstream's `fabric` branch and build normally:

```
git clone https://github.com/Earthcomputer/clientcommands
cd clientcommands
git checkout fabric
git apply /path/to/fabric-charoite.patch
./gradlew build
```

The jar is written to `build/libs/clientcommands-<version>.jar`. A prebuilt jar is attached to the
[v2.16.2-fabric-charoite release](https://github.com/clature76-source/clientcommands-mod-26.3/releases/tag/v2.16.2-fabric-charoite)
rather than committed to the repository.

## Installing

Needs Fabric Loader 0.18.4 or newer, Fabric API, and Minecraft 26.3. `betterconfig`,
`clientarguments` and `simplewaypoints` are jar-in-jar bundled and do not need to be installed
separately. The Sift itself is a separate mod and is not included.

## Verification status

| Claim | Status |
|---|---|
| Builds, and the jar contains the add-on | **Verified** — `gradlew build` succeeds and the classes/resources are present in the jar |
| Without The Sift, every conversion is the identity and detection is off | **Verified** — exercised over costs −5..60, with Minecraft's registries bootstrapped |
| With The Sift, the +20/−20 mapping and the level boost | **Verified** — checked against the mod's own decompiled mixin |
| In-game prediction while actually enchanting with charoite | **Not verified** — not exercised in a live game |

## Licence

Same as the rest of this repository: **LGPL-3.0-or-later**. All commands and features remain the work
of the original ClientCommands project; this add-on only extends the prediction to account for
another mod's fuel. The Sift is a separate mod by its own author and no part of it is redistributed
here.
