# Bandits Performance Override

A **performance fork** of the Project Zomboid **Bandits (V2)** framework for **B42**, aimed at
clawing back framerate in dense human-NPC scenes (e.g. **Bandits: Week One**, which can drop to
~8 FPS).

> **Status: WORK IN PROGRESS (v0.1.0 scaffold).** The shipped `BanditUpdate.lua` is currently a
> byte-identical copy of vanilla Bandits 42.18 (the fork baseline). The optimization changes
> (see below) are applied on top of it.

---

## What it does (the plan)

The dominant cost is **`ManageCombat`** in the core `BanditUpdate.lua`: an un-throttled,
every-tick, all-pairs proximity + line-of-sight scan run **per active bandit**, over **every
zombie + bandit in the cell** — i.e. **O(N²) per tick**. At ~100 NPCs that's the framerate killer.

This mod throttles that scan for the common peaceful case:

- **Peaceful, non-hostile NPCs** re-scan only a few times per second instead of every frame.
- **Bandits in active combat, or flagged `hostile`/`hostileP`,** keep scanning every tick — so
  combat reactions stay sharp.
- Scans are **staggered per-NPC** so the work spreads across frames instead of spiking.

The full investigation, evidence, and the broader optimization menu (population reduction,
`AreEnemies` memoization, spatial bucketing, etc.) live in **[`research/`](research/)**.

---

## How it works (and the catch)

The hot functions in `BanditUpdate.lua` are `local` (file-private), so a normal mod **cannot**
monkey-patch them. The only mechanism is **file shadowing**: this mod ships its own copy of
`media/lua/client/BanditUpdate.lua` and loads **after** the core mod (`require=Bandits2`) so the
game uses our copy instead.

That makes this a **maintained fork**, not an additive patch:

- **Pinned to Bandits `42.18`.** When the core mod updates, our copy goes stale and must be
  **re-synced**: diff the new vanilla file against ours and re-apply our changes.
- Every change we make is marked with a greppable `-- PERF FORK:` comment so the patch set is easy
  to find and re-port.
- **Still to verify:** that B42 actually lets a second mod win the file-shadow for client Lua by
  load order. This is the gating assumption — test before relying on it.

---

## Repo / mirror workflow

Source of truth is this repo at `D:\Github Repositories\BanditsPerformanceOverride`. The game loads
from a mirror at `C:\Users\sylvi\Zomboid\mods\BanditsPerformanceOverride` containing **game files
only** (root `mod.info` + `42/`). After editing here, mirror the changed game files over so the
game picks them up.

---

## Layout

```
mod.info                              # root mod metadata (id=BanditsPerformanceOverride)
42/
  mod.info                            # versioned mod metadata (identical)
  media/lua/client/BanditUpdate.lua   # the fork (vanilla 42.18 baseline + PERF FORK changes)
research/                             # the original performance investigation (preserved)
  README.md  FINDINGS.md  OPTIMIZATION-PLAN.md
```

---

## Credits

Built on top of the **Bandits (V2)** mod (`Bandits2`, workshop `3268487204`). All performance
analysis in `research/` references the `42.18` builds of Bandits and BanditsWeekOne.
