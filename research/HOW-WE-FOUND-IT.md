# HOW WE FOUND IT — the Bandits FPS hunt, start to finish

The short story of how we got from *"Week One drops to ~8 FPS"* to a working override. Same
evidence-first rules as the rest of `research/`, but told in order — including the wrong turns,
which are kept in **on purpose**. The dead ends were where most of the learning happened.

Read `FINDINGS.md` for the settled conclusions and `OPTIMIZATION-PLAN.md` for the plan. This file
is the *narrative*: how the conclusions were actually reached.

---

## 1. The symptom — and the one clue in it

The complaint was simple: **Bandits: Week One ran at ~8 FPS**, on B42 (`42.18` builds). Not in one
cursed spot — **everywhere**, and the crowd refilled wherever the player walked.

That last detail was the only clue we had at the start, and it turned out to be the important one:
the framerate tracked the **number of NPCs on screen**, not the location. A performance problem
that follows the *crowd* and not the *map* is an algorithm problem, not a bad-tile or
broken-texture problem.

> *Strip the jargon and it's this:* the game slowed down whenever a lot of people were nearby, and
> sped back up when they thinned out. So the cause wasn't "a bad spot in town" — it was the
> people themselves, or rather the work the game does for each one of them.

---

## 2. Two mods, all Lua — splitting engine from town

The Bandits "engine" isn't a compiled black box — it's **plain Lua**. Once we stopped assuming
otherwise and actually opened the folders, the shape became clear. There are **two** mods:

| Mod | Workshop ID | What it owns |
|-----|-------------|--------------|
| **Core** (`Bandits`) | `3268487204` | The tick loop + combat (`BanditUpdate.lua`) and the entity cache (`BanditZombie.lua`). The machinery. |
| **Content** (`BanditsWeekOne`) | `3403180543` | The Week One scenario: population control, the civilian/cop brains, events, world dressing. The town. |

Knowing which mod owned what told us where to *not* look. The per-tick cost was going to live in
the **core's** loop, not in the content's scenario scripts.

> *Plainly:* one mod is the engine that makes the people tick; the other is the town they live in.
> Once we knew that, we knew the slowdown was an engine problem, so that's the file we put under
> the microscope.

---

## 3. Following the heartbeat — what runs *every* tick

The core's per-entity heartbeat is `OnBanditUpdate` (`BanditUpdate.lua:1899`, registered at
`:2471`). It fires **once per zombie per tick** — and Bandits actually hangs **two** handlers on
every entity every tick before any "decision" happens.

Tracing inward, `GenerateTask` (`:1810`) is where the per-tick work lands. The key discovery:

- `ManageCombat` (`:1842`) runs **every single tick**, with **no `HasTask` guard** — it fires even
  while an NPC is mid-walk, mid-anything.
- The NPC **brains** (the per-type decision trees) only run when the task queue is **empty**
  (`:1866`). A walking NPC is busy consuming a Move task, so its brain doesn't re-fire.

That second point quietly **cleared the brains as suspects.** The intuitive guess — "the AI is
thinking too hard" — was wrong. The brains barely run. The cost was something dumber and more
relentless.

> *Put simply:* we timed the game's pulse and found that one specific job runs on *every single
> beat*, for *every* person — while the thing everyone assumes is expensive (the "thinking") barely
> runs at all. We'd been blaming the wrong part.

---

## 4. The smoking gun — `ManageCombat` is O(N²)

Inside `ManageCombat` (`:903`) sits the loop that explains the 8 FPS. Every tick, **each** bandit
runs this (`:1048`):

```
for every zombie + bandit in the cell:
    check distance, gated < 57    (:1053)  -- 57 tiles ≈ the whole cell; filters almost nothing
    AreEnemies(them, me)          (:1055)  -- a function call on every single pair
    if enemy: CanSee(them)        -- a line-of-sight raycast per enemy pair. expensive.
```

So each bandit looks at **everyone** in the cell, every tick. With N people that's N checks per
bandit, and N bandits doing it → **N² per tick**. At ~100 NPCs that's **~10,000 checks per frame**,
each at minimum a function call, with a raycast on top for every hostile pair. And because the
`< 57` gate is basically the whole cell, the cost **never tapers with distance** — someone across
the map counts as much as someone in your face. (The flee branch even made a **second** full pass,
`:1238`, to build its run-away vector — double the loop on the worst-case path.)

This is the 8 FPS. Not a leak, not a bad asset — a textbook quadratic scan running flat-out every
frame.

> *In plain English:* imagine every person in a crowd stopping, several times a second, to size up
> *every other person* in the whole area — "do I know them? are they a threat? can I see them?" —
> and nobody ever skips anyone, even someone a block away. Ten people is fine. A hundred people
> means ten thousand of these little check-ins per frame, and the game grinds to a crawl.

---

## 5. The catch — everything worth fixing is locked away

The obvious move — write a small mod that swaps in a smarter `ManageCombat` — turned out to be
impossible. **All** the hot functions (`ManageCombat`, `GenerateTask`, `OnBanditUpdate`, the cache
builders) are declared `local` — file-private. A separate mod literally **cannot reference or
override them.** Only the global tables (`Bandit`, `BanditBrain`, `BanditZombie`, `BanditUtils`, …)
are reachable from outside.

That left exactly two ways to change the loop:

1. **Edit the core file directly** — but a Workshop update overwrites it.
2. **Shadow the whole file** — ship our own copy of `client/BanditUpdate.lua` and load **after** the
   core (`require=Bandits2`) so the game uses ours. An override wearing a mod costume.

We chose the override: pinned to **`42.18`**, every change tagged `-- PERF OVERRIDE:` so the diff stays
greppable and re-portable when the core updates. This constraint, more than anything, shaped the
whole project — it's *why* this is a maintained override and not a tidy little add-on.

> *Plainly:* the slow code was behind a door with no outside handle — we couldn't reach in and
> tweak it from a normal mod. So instead we made our own copy of the whole page, fixed our copy,
> and told the game to read ours instead of theirs. The downside: when they update the original, we
> have to re-copy and re-apply our changes by hand.

---

## 6. Measure, don't guess — and the fix the author half-wrote

Before changing a line, we got real numbers. Conveniently, **the original author had left a
speedometer in the code**: per-tick timers bucketed into `<1ms / <5ms / ≥5ms` (`:1901`,
`:2111-2121`) and a `perf` printer commented out at `:2483`. We switched it on and read **actual
ms-per-bandit** in a dense scene — a baseline *and* a regression check for every change after.

Two finds made it clear we were on the right track, not guessing:

- The author had **already sketched the fix themselves** — a commented-out adaptive update-skip at
  `:1977` ("up to 100 zombies update every tick, 800+ every 1/16 tick"). They knew. They just
  never finished it. We were effectively completing their to-do list.
- Every optimization below was kept or thrown out **based on the timer**, not on vibes.

> *In everyday terms:* we didn't guess at what was slow and start ripping things out — we turned on
> the stopwatch the author had already built into the code, took a "before" time, and then made the
> stopwatch prove each change actually helped. And it turned out the original author had even left
> behind a rough sketch of the fix; we mostly finished what they'd started.

---

## 7. The fixes — in order, with the timer judging each one

We changed only what the numbers justified, re-measuring after each:

| Tag | Change | Why it works |
|-----|--------|--------------|
| **B2** | Tighten the distance gate; let melee/out-of-ammo NPCs skip the line-of-sight raycasts on far targets | kills the most expensive check for almost every pair |
| **B1** | Count-based throttle — peaceful NPCs run the scan every Nth tick, **staggered per-NPC**; anyone hostile or in combat still scans every tick | turns "every frame" into "a few times a second" for the calm majority, without dulling real fights |
| **flee-dedup** | Build the flee vector during the main scan instead of a second full pass | deletes the worst-case double-loop |
| **B3** | A coarse spatial grid so each scan walks only **nearby** buckets, not the whole cell | this is the one that actually bends the O(N²) curve down |

And — just as important — the one we **said no to**: **B4**, gating `ApplyVisuals`. It *looked*
expensive and tempting, but the timer measured it at only **~4%** of bandit-update cost. So we left
it alone. Chasing it would have added risk for almost nothing.

**Result:** the dense Week One scene went from **~8 FPS at ~100 NPCs** to **smooth at ~300**.

> *Plainly:* we fixed the slow thing in four steps — first stop checking distant people you can't
> reach, then let calm bystanders check the crowd a few times a second instead of every frame, then
> stop doing the run-away math twice, then divide the area into a grid so each person only checks
> their own neighborhood. After each step we re-timed it to make sure it really helped. And we
> deliberately *didn't* "fix" one thing that looked slow but measured as nearly free.

---

## 8. What we'd tell the next person

The investigation in four lessons, in case it saves the next session a night:

1. **Open the folders before you theorize.** We nearly wrote this off as "compiled and untouchable"
   when it was plain Lua one drawer over. Grep first, conclude second.
2. **Time the loop, don't blame the obvious suspect.** The "AI brains" were innocent; a dumb
   every-tick scan was guilty. The stopwatch said so.
3. **Find the real constraint early.** "The hot functions are `local`" decided the entire shape of
   the solution (an override, not an add-on). The sooner you know your constraint, the less you waste.
4. **Let the measurement say yes *and* no.** It greenlit four changes and vetoed a fifth. A fix you
   can't measure is a guess you haven't caught yet.
