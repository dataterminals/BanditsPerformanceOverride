# Bandits Performance Override

A **performance override** for the Project Zomboid **Bandits (V2)** framework for **B42**, aimed at
clawing back framerate in dense human-NPC scenes (e.g. **Bandits: Week One**, which can drop to
~8 FPS).

> **Status: working (v0.2.0).** B1–B3 implemented, measured, and validated in-game: a dense
> Bandits: Week One scene went from ~8 FPS at ~100 NPCs to smooth at **~300**. The override is kept as a
> minimal, greppable diff against vanilla Bandits 42.18 for easy re-porting.

---

## What it does

The dominant cost is **`ManageCombat`** in the core `BanditUpdate.lua`: an un-throttled,
every-update, all-pairs proximity + line-of-sight scan run **per active bandit** over **every
zombie + bandit in the cell** — i.e. **O(N²)**. At ~100+ NPCs that's the framerate killer.

This override rewrites that scan with four changes (each tagged `-- PERF OVERRIDE:`):

1. **Count-based throttle** — peaceful, non-hostile NPCs run the scan only once every Nth update
   (staggered per-NPC so the crowd desyncs across frames). Anyone in active combat or flagged
   `hostile`/`hostileP` keeps scanning every update, so reactions stay sharp.
2. **Shooter-safe distance gate** — melee / out-of-ammo NPCs skip the expensive line-of-sight
   raycasts on far targets they could never reach, while NPCs with a loaded ranged weapon keep the
   full engagement radius.
3. **Flee de-duplication** — the flee branch no longer makes a second full-cell pass; its repulsion
   vector is accumulated during the main scan.
4. **Spatial grid** — a coarse bucket grid (rebuilt at most every 150ms) means each scan only walks
   nearby cells instead of the whole cell, turning the per-scan cost from **O(N)** into roughly
   **O(local density)**. This is the change that bends the O(N²) curve.

The full investigation and evidence — plus the levers we measured but deliberately *didn't* take
(e.g. gating `ApplyVisuals`, which profiled at only ~4% of cost) — live in **[`research/`](research/)**.

---

## How it works (and the catch)

The hot functions in `BanditUpdate.lua` are `local` (file-private), so a normal mod **cannot**
monkey-patch them. The only mechanism is **file shadowing**: this mod ships its own copy of
`media/lua/client/BanditUpdate.lua` and loads **after** the core mod (`require=Bandits2`) so the
game uses our copy instead.

That makes this a **maintained override**, not an additive patch:

- **Pinned to Bandits `42.18`.** When the core mod updates, our copy goes stale and must be
  **re-synced**: diff the new vanilla file against ours and re-apply our changes.
- Every change we make is marked with a greppable `-- PERF OVERRIDE:` comment so the patch set is easy
  to find and re-port.
- **Confirmed working:** B42 does let a second mod win the file-shadow for client Lua by load
  order. `require=Bandits2` makes this mod load after the core, so the game uses our copy (the
  in-game load marker in the console confirms it).

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
  media/lua/client/BanditUpdate.lua   # the override (vanilla 42.18 baseline + PERF OVERRIDE changes)
research/                             # the original performance investigation (preserved)
  README.md  FINDINGS.md  OPTIMIZATION-PLAN.md
  HOW-WE-FOUND-IT.md                  # narrative of the hunt: how we figured it out
```

---

## Credits

Built on top of the **Bandits (V2)** mod (`Bandits2`, workshop `3268487204`). All performance
analysis in `research/` references the `42.18` builds of Bandits and BanditsWeekOne.
