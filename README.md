# PZ Bandits — Performance Research & Un-junking Plan

Investigation into why a **Project Zomboid B42 "Bandits: Week One"** playthrough runs at
~8 FPS, and a plan to claw performance back via mods.

This repo is a **cold-start handoff**: it is written so a fresh session on another machine
(no chat history) can pick the work up exactly where it was left.

---

## TL;DR of where we landed

- The framerate problem is **not** the NPC "brains" (the per-type decision trees). Those only
  fire when an NPC's task queue drains.
- The real cost is an **un-throttled, every-tick, all-pairs proximity + line-of-sight scan**
  (`ManageCombat`) run **per bandit, per tick**, over **every zombie+bandit in the cell** — i.e.
  **O(N²) per tick**. With ~100 NPCs around the player that is the frame killer.
- The hot functions are all `local` (file-private), so a normal standalone mod **cannot**
  monkey-patch them. That constraint shapes the whole plan (see `OPTIMIZATION-PLAN.md`).
- Highest-leverage **fully-standalone** win: **reduce N** (population) — O(N²) means halving the
  crowd quarters the cost. The real algorithmic fix (throttle/cache/spatial-grid) requires
  forking one core file.

Full evidence with file:line references is in **`FINDINGS.md`**.
The actionable plan is in **`OPTIMIZATION-PLAN.md`**.

---

## How to resume on a new machine

1. **Read the docs in order:** `README.md` → `FINDINGS.md` → `OPTIMIZATION-PLAN.md`.
2. **Locate the game files** (paths differ per machine — see below).
3. Continue from the "Next actions" list at the bottom of `OPTIMIZATION-PLAN.md`.

### Locating the mods on any machine

Project Zomboid's Steam **App ID is `108600`**. Workshop mods live under:

```
<SteamLibrary>/steamapps/workshop/content/108600/<workshopId>/mods/<ModName>/
```

The two mods that matter (find them by **workshop ID**, the drive/library path will differ):

| Role | Workshop ID | Mod folder | Notes |
|------|-------------|-----------|-------|
| **Core engine** (the AI, the tick loop, combat) | `3268487204` | `mods/Bandits/` | All plain Lua. Ships a folder per game version (`42.12`…`42.18`). **Use `42.18`.** |
| **Content pack** (Week One scenario, populations, civilian brains) | `3403180543` | `mods/BanditsWeekOne/` | Built on top of the core. Uses `42.18/media/...`. |

On the **desktop where this was authored**, that resolved to:

```
H:\SteamLibrary\steamapps\workshop\content\108600\3268487204\mods\Bandits\42.18\
H:\SteamLibrary\steamapps\workshop\content\108600\3403180543\mods\BanditsWeekOne\42.18\
```

On the **laptop the library is probably on a different drive/letter.** To find it quickly,
search the filesystem for one of the workshop IDs, e.g. (PowerShell):

```powershell
Get-ChildItem -Path C:\,D:\,E:\ -Recurse -Directory -Filter 3268487204 -ErrorAction SilentlyContinue |
  Select-Object FullName
```

or look in `steamapps\libraryfolders.vdf` for registered library paths.

> **All file references in these docs use mod-relative paths** (e.g.
> `Bandits/42.18/media/lua/client/BanditUpdate.lua:1842`) so they're valid regardless of where
> your Steam library lives. Prefix with your machine's `<SteamLibrary>/steamapps/workshop/content/108600/<id>/mods/`.

---

## Author's playthrough context (the symptom)

- Scenario: **Bandits: Week One** (pre-apocalypse simulation; civilians, cops, criminals etc.
  are reskinned zombies running player animations).
- Symptom: **~8 FPS everywhere**, not localized to one spot.
- Game build: **B42** (`42.18` mod builds active).

---

## Glossary

- **Bandit** — any Bandits-framework human NPC. Mechanically an `IsoZombie` flagged
  `setVariable("Bandit", true)`, re-skinned with the player model + human animations, driven by a
  Lua "brain" stored in its `getModData().brain`.
- **Brain** — per-NPC state table in ModData: `id`, current `program`, `tasks` queue, `weapons`
  (a data manifest, not a real inventory), flags (`hostile`, `endurance`, `infection`, …).
- **Program** — a per-NPC-type behavior state machine (e.g. `ZombiePrograms.Inhabitant.Main`).
  Returns `{status, next, tasks}`.
- **Task** — one queued atomic action (Move, Smack, Shoot, DoorLock, …) consumed by the engine.
- **CacheLight / CacheLightB / CacheLightZ** — global tables of lightweight `{id,x,y,z,brain}`
  records for all entities / bandits only / zombies only.

---

## Repo conventions

- Docs are Markdown, evidence-first, with `file:line` citations into the **`42.18`** mod builds.
- Nothing here is game code; it's notes/plans. No mod is built yet.
