# FINDINGS — Bandits NPC performance deep-dive

Evidence-first. All `file:line` references are into the **`42.18`** builds:

- Core: `Bandits/42.18/media/lua/...` (workshop `3268487204`)
- Content: `BanditsWeekOne/42.18/media/lua/...` (workshop `3403180543`)

(See `README.md` for how to resolve those to absolute paths on your machine.)

---

## 1. Architecture: two mods, all Lua

There is **no compiled engine.** (We chased that red herring for a while — the engine is plain
Lua in a *separate* mod we initially didn't open. `ZombieBuddy.jar` in the game root is an
**unrelated** mod and is **not** part of Bandits.)

- **Core (`mods/Bandits`)** — defines the globals the content uses: `Bandit`, `BanditBrain`,
  `BanditZombie`, `BanditUtils`, `BanditPrograms`, `BanditPlayer`, `ZombiePrograms` (base),
  `ZombieActions`. Owns the **tick loop and combat** (`BanditUpdate.lua`) and the entity **cache**
  (`BanditZombie.lua`).
- **Content (`mods/BanditsWeekOne`)** — the Week One scenario: population control
  (`BWOPopControl.lua`), the civilian/cop/etc. **brains** (`ZombiePrograms/WeekOne/ZP*.lua`),
  scheduler, events, world dressing.

Key core files:

| File | Role |
|------|------|
| `Bandits/42.18/media/lua/client/BanditUpdate.lua` | **The per-tick AI loop + combat.** 2483 lines. The hot path. |
| `Bandits/42.18/media/lua/client/BanditZombie.lua` | Builds/maintains the entity caches. |
| `Bandits/42.18/media/lua/shared/BanditBrain.lua` | Brain get/set (trivial ModData wrapper) + ammo/task predicates. |
| `Bandits/42.18/media/lua/shared/BanditPrograms.lua` | Shared sub-behaviors (ATM, Events, Symptoms, Weapon.Switch…). |
| `Bandits/42.18/media/lua/shared/BanditUtils.lua` | Helpers: `AreEnemies`, `GetMoveTask`, `GetClosestBanditLocation`, IDs… |

---

## 2. What a Bandits NPC is

A "bandit" (civilian, cop, criminal, partier — all of them) is a vanilla `IsoZombie` converted by
`Banditize()` (`BanditUpdate.lua:158`):

- `zombie:setVariable("Bandit", true)` — the flag everything keys off (`:167`).
- Player walk/run/limp speeds, walk type set to player animations (`:169-179`).
- `setVariable("ZombieHitReaction","Chainsaw")` — a hack to skip the engine's `testDefense`
  (which assumes moodles a zombie lacks) and avoid black-screen crashes (`:181-184`).
- Brain attached via `BanditBrain.Update(zombie, brain)` → stored at
  `zombie:getModData().brain` (`BanditBrain.lua:3-11`).

**Inventory model (confirmed):** a bandit's gear is a **data manifest**, not a live container.
`bandit.weapons` and `bandit.loot` are precomputed tables (`BanditsWeekOne/.../BWOBanditCreator.lua:64,79`).
The equipped weapon is real; the "backpack" only materializes into a lootable corpse on death.
This is a deliberate optimization (avoids real `ItemContainer`s on 100+ live NPCs).

---

## 3. The tick model

### Two handlers fire per zombie, per tick

`OnZombieUpdate` is a vanilla event that fires **once per zombie per update tick**. Bandits hooks
it **twice**:

1. `BanditZombie.onZombieUpdate` — cache upkeep (`BanditZombie.lua:191`).
2. `BanditUpdate.OnBanditUpdate` — the AI (`BanditUpdate.lua:2471`).

So every entity carries two Lua handlers every tick before any AI "decision" happens.

### OnBanditUpdate flow (`BanditUpdate.lua:1899`)

For an **active, near-player bandit**, every tick:

1. Early-outs: server / ragdoll / reanimated-for-grapple (`:1903-1927`).
2. Banditize/Zombify reconciliation against cluster data (`:1963-1973`).
3. `if uTick % 2 == 0 then UpdateZombies(zombie)` (`:1984`) — but `UpdateZombies` returns
   immediately for bandits (`:1504`), so this cost lands on **regular zombies**, not bandits.
4. `if not zombie:getVariableBoolean("Bandit") then return end` (`:2001`).
5. `setUseless(false)` if in `CacheLightB`, else `setUseless(true); return` — i.e. **distant
   bandits are skipped** (`:2006-2011`). Good: only near-player bandits run the full path.
6. Per tick: force walk-type/anim (`:2039`), suppress zombie sounds (`:2044`),
   **`Bandit.ApplyVisuals` (`:2052`)**, speech cooldown (`:2069`), `ManageActionState` (`:2073`).
7. `GenerateTask(bandit, uTick)` (`:2096`) — see below.
8. `ProcessTask` to advance the current task/animation (`:2098-2102`).
9. `uTick = uTick + 1` (`:2109`); whole function timed into `<1ms / <5ms / ≥5ms` buckets
   `sum1/sum2/sum3` (`:2111-2121`).

> `uTick` is a **single module-level counter** (`:1898`) shared across all bandits, wrapping at 16
> (`:1923`). It's used only to stagger cheap subsystems: torch at `uTick==1` (`:2056`), fire at
> `uTick==2` (`:2064`), `UpdateZombies` at `% 2` (`:1984`). It is **not** a per-entity throttle and
> it does **not** gate combat.

### GenerateTask — what actually runs, and the one gate (`BanditUpdate.lua:1810`)

```lua
local tasks = {}
ManageEndurance(bandit)                       -- runs EVERY tick (:1816)
if #tasks == 0 then ManageHealth(bandit)  end -- (:1828)
if #tasks == 0 then ManageCombat(bandit)  end -- (:1842)  <-- the expensive one
if #tasks == 0 then ManageCollisions(bandit) end -- (:1855)
if #tasks == 0 and not Bandit.HasTask(bandit) then
    -- THE BRAIN. Only fires when the task queue is EMPTY:
    local res = ZombiePrograms[program.name][program.stage](bandit)  -- (:1870)
end
```

**Critical takeaways:**

- The **brain decision tree** (`ZPInhabitant.Main` etc.) only runs when the NPC has **no queued
  task** (`:1866`). A walking NPC is consuming a Move task; the brain doesn't re-fire until it
  completes. **The brains are NOT the per-tick cost.**
- `ManageCombat` runs **every tick** (the `tasks` local is fresh each call, so `#tasks==0` is true
  going in; the only thing that can pre-empt it is an endurance/health task, which are rare).
  There is **no `HasTask` guard on ManageCombat** — it runs even while the bandit is mid-walk.

---

## 4. The cache it scans (`BanditZombie.lua`)

- `onZombieUpdate` (`:44`, on `OnZombieUpdate`) updates one entity's light record per tick.
  For **bandits** it also calls `GetBrain` and **`getRoomId`** (square→room→roomDef→idString)
  **every tick** (`:78-79`).
- `flush` (`:97`, on `EveryOneMinute`) fully rebuilds all four caches from the cell zombie list,
  and toggles vanilla **tiered zombie updates** once bandit or zombie count ≥ 50 (`:151-157`).
- `CacheLight` contains **all** entities (zombies **and** bandits); `CacheLightB` bandits only;
  `CacheLightZ` zombies only. All are id-keyed Lua hash tables.

---

## 5. `ManageCombat` — the O(N²) hotspot (`BanditUpdate.lua:903`)

Per active bandit, per tick:

1. **Flags** (heal/reload/resupply), gated behind `not HasActionTask` — cheap (`:928-954`).
2. **Player scan** (`:963-1044`) — only if `brain.hostile/hostileP`. Loops players, `CanSee` +
   weapon range. Cheap unless hostile.
3. **Entity scan (THE COST)** — `for id, potentialEnemy in pairs(BanditZombie.CacheLight)`
   (`:1048`), i.e. iterate **every zombie+bandit in the cell**. Per entry:
   - Manhattan distance, gate **`< 57`** (`:1053`) — 57 tiles ≈ whole cell, filters ~nothing.
   - **`BanditUtils.AreEnemies(potentialEnemy.brain, brain)` — a function call on every pair**
     (`:1055`).
   - **Enemy branch** (`:1059-1142`): load real object `cache[id]`, `isAlive`, `getHealth`,
     **`bandit:CanSee(...)` (LOS raycast)**, `getSquare:getLightLevel`, sqrt distance, weapon-range
     instancing. Expensive.
   - **Friendly branch** (`:1146-1153`): cheap distance-squared counting.
4. **Action resolution** (`:1168+`). Note the **flee** branch (`:1224-1358`) does a **second full
   pass** over the same list to build a repulsion vector (`:1238`).

### Cost model

Inner loop is **O(N)** per bandit → **O(N²) per tick** across N bandits, where N = all
zombies+bandits in the cell. At ~100 NPCs ≈ **10,000 iterations/tick**, each with at least an
`AreEnemies` call, and a `CanSee` raycast per *enemy* pair. The `< 57` gate means cost does **not**
taper with distance. **This is the 8 FPS.**

(Pre-outbreak Week One is mostly peaceful same-clan civilians → `AreEnemies` is false → most pairs
hit the cheap friendly branch. It's still N² function-calls + table indexing every tick. After the
outbreak, real zombies enter `CacheLight` too and the enemy/`CanSee` branch fires far more.)

---

## 6. Other per-tick costs (secondary, but real)

- **`Bandit.ApplyVisuals(bandit, brain)` every tick** per bandit (`:2052`) — re-stamps the human
  model/clothing. Worth measuring; likely trimmable to on-change.
- **`getRoomId` every tick** per bandit in the cache upkeep (`BanditZombie.lua:79`) — room rarely
  changes; cacheable.
- **Zombies pathing to bandits** — `UpdateZombies` (`BanditUpdate.lua:1492`), every other tick
  (`% 2`), makes each regular zombie find the nearest bandit (`GetClosestBanditLocation`) and
  `pathToCharacter` toward it (`:1618-1637`). A second per-tick drain that scales with zombie count.

---

## 7. The disabled adaptive throttle (the author's own blueprint)

`BanditUpdate.lua:1977-1987` — the author **wrote and commented out** an adaptive update-skip:

```lua
-- Up to 100 zombies, update every tick,
-- 800+ zombies, update every 1/16 tick.
-- local zcnt = BanditZombie.GetAllCnt()
-- if zcnt > 600 then zcnt = 600 end
-- local skip = math.floor(zcnt / 50) + 1
if uTick % 2 == 0 then
    UpdateZombies(zombie)
end
```

It was replaced by a flat `% 2`, and **there is no throttle at all on the bandit-side
`ManageCombat` scan.** This commented block is essentially the design sketch for the real fix.

---

## 8. THE constraint that shapes everything: hot functions are `local`

In `BanditUpdate.lua`, **all** the hot functions are file-private `local function`:

- `ManageCombat` (`:903`), `GenerateTask` (`:1810`), `UpdateZombies` (`:1492`),
  `OnBanditUpdate` (`:1899`), `ProcessTask`, the `Manage*` helpers.
- In `BanditZombie.lua`: `onZombieUpdate` (`:44`), `flush` (`:97`).

A separate mod **cannot reference or override** these. The only **global** seams are the tables and
their methods: `Bandit.*`, `BanditBrain.*`, `BanditZombie.*` (incl. the cache tables),
`BanditUtils.*`, `BanditPrograms.*`, `ZombiePrograms.*`, `ZombieActions.*`.

⇒ Algorithmic fixes inside `ManageCombat`/`GenerateTask` are **only** reachable by **editing the
core file** (or shadowing the whole file via load order — a fork in disguise). Everything else must
work through the global tables or the content mod. See `OPTIMIZATION-PLAN.md`.

---

## 9. The N driver: population (`BanditsWeekOne/.../BWOPopControl.lua`)

`BWOPopControl.UpdateCivs` (`:484`, on `EveryOneMinute`) targets:

- `StreetsNominal = 46`, `InhabitantsNominal = 75` (`:584-585`)
- scaled by `density` (building density, capped **2.2×**, `:644-645`) and an hour-of-day factor
  (up to **1.2×** at 8am/5pm, `:495-505`) and sandbox `*PopMultiplier`s.

⇒ In a dense area at a busy hour: **100+ live human NPCs**, continuously refilled wherever the
player goes (hence low FPS *everywhere*). This is the **N** in O(N²) and the easiest lever to pull
from a standalone mod.

---

## 10. Instrumentation the author left in (use it!)

- `OnBanditUpdate` wrapped in `getTimestampMs`, bucketed `<1 / <5 / ≥5 ms` into `sum1/2/3`,
  `iter1/2/3` (`BanditUpdate.lua:1901,2111-2121`). A `perf` printer on `EveryOneMinute` is
  commented at `:2483`.
- `BanditZombie.lua:24-25,87-88` accumulates `sum/invocations` for cache upkeep.
- The civilian brains (`ZP*.lua`) are full of commented `getTimestampMs()` phase timers.

⇒ Re-enabling these (uncomment the `perf` print) gives **real per-bandit ms** in-game to validate
any change. This should be step 1 of any optimization work.
