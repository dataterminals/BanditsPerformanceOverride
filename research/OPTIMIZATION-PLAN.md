# OPTIMIZATION PLAN — un-junking Bandits FPS

Goal: claw back framerate in a dense Bandits: Week One scene without breaking behavior.

Read `FINDINGS.md` first. The single most important fact driving this plan:

> **The hot functions (`ManageCombat`, `GenerateTask`, `OnBanditUpdate`, the cache builders) are
> `local` (file-private).** A standalone mod cannot patch them. Only the global tables
> (`Bandit`, `BanditBrain`, `BanditZombie`, `BanditUtils`, `BanditPrograms`, `ZombiePrograms`,
> `ZombieActions`) are reachable from outside.

So options split into **(A) fully standalone** and **(B) requires overriding the core file.**

---

## Principle: measure first

Before changing anything, get real numbers in-game:

1. Open `Bandits/42.18/media/lua/client/BanditUpdate.lua`.
2. Uncomment the `perf` printer registration at the bottom (`:2483`,
   `-- Events.EveryOneMinute.Add(perf)`) and whatever `perf` function it references (find it near
   the bucket counters `sum1/sum2/sum3`). If a `perf` function isn't defined, write a tiny one that
   prints `sum1/iter1`, `sum2/iter2`, `sum3/iter3` and resets them.
3. Stand in a dense area, read the console: this tells you the **per-bandit ms** and how many
   bandits land in the `≥5ms` bucket. That's your baseline and your regression check.

> Note: this is an edit to the core mod for measurement only. If you want measurement without
> editing the core, you can instead add your own `Events.OnZombieUpdate` handler in a standalone
> mod that times nothing of theirs but counts `BanditZombie.CacheLightBCnt` / `CacheLightZCnt`
> each minute — useful for correlating FPS with N.

---

## Bucket A — fully standalone mod (no core edits)

These ship as a normal separate mod, survive core updates, and are the safe first deliverable.

### A1. Reduce N (population) — **highest leverage**
O(N²): halving the crowd **quarters** the combat cost.

- Target: `BanditsWeekOne/.../BWOPopControl.lua` values `StreetsNominal` (46),
  `InhabitantsNominal` (75), the density cap (2.2), and/or the `*PopMultiplier` sandbox vars.
- `BWOPopControl` is **content-side and the functions/values are reachable** (it's a global table,
  and the sandbox multipliers are first-class sandbox options).
- Cleanest standalone approach: a mod that **lowers the `BanditsWeekOne` sandbox pop multipliers**
  (no code override needed if exposed as sandbox vars), or wraps `BWOPopControl.UpdateCivs` if it's
  global. **Verify** whether `StreetsPopMultiplier` / `InhabitantsPopMultiplier` are sandbox vars
  (they're referenced as `SandboxVars.BanditsWeekOne.*` in `UpdateCivs`) — if so this is trivial
  and needs no Lua at all, just a preset.
- Expected: large, predictable win; some loss of "city feels alive" density.

### A2. Memoize `BanditUtils.AreEnemies`
It's **global**, called once per pair in the O(N²) loop, and clan/hostility relationships change
rarely.

- Replace `BanditUtils.AreEnemies` with a wrapper that caches results keyed by
  `(clanA, hostileA, clanB, hostileB)` (or brain identity + a generation counter bumped when
  hostility flips). Invalidate cheaply.
- Slots in cleanly from a standalone mod that loads after core.
- Expected: shaves the per-pair constant factor (still O(N²) iterations, but each is cheaper).

### A3. Disable per-tick subsystems via sandbox vars
`GenerateTask` runs extra per-tick work gated by sandbox flags:
`General_LimitedEndurance` (`:458`), `General_BleedOut` (`:484`), `General_Infection` (`:500`).

- A sandbox preset turning these off trims fixed per-bandit cost. Cheap, behavior-affecting,
  reversible. Good for a "performance preset" mod.

### A4. (Investigate) tiered zombie updates threshold
`BanditZombie.flush` flips vanilla `setOptionTieredZombieUpdates` at count ≥ 50 (`:151-157`).
Worth testing whether forcing it **on** earlier (or always) helps the zombie-side cost — but note
the author's comment that it stops off-screen bandits doing non-move actions. Behavior tradeoff.

**Bucket A combined expectation:** plausibly **30–50%** back in a dense scene, mostly from A1.

---

## Bucket B — requires overriding the core (`BanditUpdate.lua`)

These are the real algorithmic fixes. They live inside `local function ManageCombat` /
`GenerateTask`, so they can only be done by **editing the core file** or **shadowing the whole file
via mod load order** (ship your own `client/BanditUpdate.lua` that loads after Bandits — an override
wearing a mod costume, fragile across core updates). Decide consciously whether you're willing to
maintain that.

### B1. Throttle + cache the combat scan — **the main fix**
Don't run the O(N²) scan every tick. Cache its result in the brain and recompute every N ticks.

- Store on the brain: `brain.threat = { enemy, enemies, friendlies, expire }`.
- At the top of `ManageCombat`: if `now < brain.threat.expire` **and** not in active combat, reuse
  cached counts / chosen enemy and skip the loop.
- Pick N adaptively: **N≈1–2** when `hostile/hostileP` or enemies were present last scan;
  **N≈15–20** for a peaceful civilian with no threat.
- **Stagger by `id % N`** so all bandits don't scan on the same frame (spreads spikes — directly in
  the spirit of the author's disabled adaptive block at `:1977`).
- Expected: turns the dominant cost from "every tick" into "every Nth tick" for the common peaceful
  case. Likely the biggest single win after A1.

### B2. Tighten the `< 57` Manhattan pre-filter (`:1053`)
`escapeDist` is 10; nothing past ~12 Manhattan matters. Drop the gate to ~12–15 so the expensive
enemy/`CanSee` body short-circuits for almost all pairs. One-line change, low risk. Pairs well with
B1.

### B3. Spatial bucketing — kills the O(N²)
Maintain a coarse grid (`floor(x/10), floor(y/10)` → list of ids) alongside `CacheLight`, and have
the scan query only neighboring buckets → ~O(N·k).

- The grid can be **built from a standalone mod** (hook `OnZombieUpdate` yourself; it's global
  data on `BanditZombie`), but **consuming** it requires editing `ManageCombat` (Bucket B).
- Bigger change; do after B1/B2 prove out. This is the "do it right" endgame.

### B4. Trim `ApplyVisuals` / `getRoomId` to on-change
- `ApplyVisuals` every tick (`:2052`) and `getRoomId` every tick (`BanditZombie.lua:79`) are likely
  reducible to "only when changed." Measure with the perf hook before/after.

---

## Recommended sequencing

1. **Measure** (perf hook) → baseline.
2. **A1** (population) → re-measure. Probably the biggest, easiest gain. Possibly enough on its own.
3. **A2 + A3** (memoize `AreEnemies`, sandbox trims) → re-measure.
4. **Decision point:** is Bucket A enough? If not, commit to an **override** of `BanditUpdate.lua`.
5. **B2** (tighten gate) + **B1** (throttle/cache) → re-measure. This is the real fix.
6. **B3** (spatial grid) only if still needed at very high N.

---

## Open questions to resolve on resume

- [ ] Are `StreetsPopMultiplier` / `InhabitantsPopMultiplier` real sandbox vars exposed in the
      Week One sandbox UI? (If yes, A1 is a no-code preset.) Check
      `BanditsWeekOne/.../sandbox-options.txt` (or the `SandboxVars.BanditsWeekOne` definitions).
- [ ] Is `BWOPopControl` a global table (overridable) or local? (Determines A1 implementation.)
- [ ] Does a `perf` function already exist near `BanditUpdate.lua:2483`, or do we write one?
- [ ] How is the core mod loaded relative to content — confirm load order so a file-shadow of
      `BanditUpdate.lua` would actually win (for Bucket B as a "mod").
- [ ] Single-player only? (`OnBanditUpdate` early-returns on `isServer`; multiplayer has extra
      paths — `ManageSocialDistance`, window hacks. Keep MP in mind if relevant, else ignore.)
- [ ] Confirm post-outbreak behavior: once real zombies populate `CacheLight`, does the enemy/
      `CanSee` branch dominate? (Changes the relative value of B2/B3.)

---

## Next actions (cold-resume checklist)

1. [ ] Locate the two mods on this machine (see `README.md` → "Locating the mods").
2. [ ] Re-enable the perf print in `BanditUpdate.lua` and capture a baseline ms/bandit + N in a
       dense scene.
3. [ ] Answer the "Are pop multipliers sandbox vars?" question → build the **A1 standalone pop mod
       / preset**. Re-measure.
4. [ ] Add **A2** `AreEnemies` memoization in the same standalone mod. Re-measure.
5. [ ] Make the **override-or-not** decision for Bucket B.
6. [ ] If overriding: implement **B2** then **B1**; re-measure after each.

---

## Risks / caveats

- **Core updates** will overwrite any direct edits to `mods/Bandits` and can break a file-shadow
  override. Keep override changes minimal and documented (diff-friendly).
- **Behavior changes**: lowering N and throttling scans changes the feel (less dense, slightly less
  reactive combat). That's the intended trade; tune to taste.
- **Don't optimize the brains** (`ZP*.lua`) for FPS — they're not the per-tick cost (see
  `FINDINGS.md §3`). Effort there is mostly wasted for performance.
- **Validate with the perf hook**, not vibes — every step should re-measure ms/bandit.
