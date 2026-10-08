# HUNTED — a Game AI demo (AI in Gaming, Unit 3)

A top-down stealth/chase game in a single file (`index.html`: HTML5 canvas + vanilla JS, no libraries, no network).
**Run it:** double-click `index.html`. Click the canvas or press Enter to start.

You are a scout in a ruined arena at night. Collect **5 relay beacons**, then reach the **green extraction signal**.
Hunters patrol, listen and chase. Each wave adds more (and sharper) enemies. 3 hearts, game over at 0.

## Controls
| Key | Action |
|---|---|
| WASD / arrows | Move (acceleration-based, slight inertia). Touch: drag to move + on-screen buttons |
| Shift | Sprint (limited stamina) — **loud**: hunters can hear it |
| E or mouse click | Throw a **decoy noise** (6 s cooldown); hunters go to investigate it |
| **F1** | Debug / teaching overlay (also a button) |
| **K** | Kinematic ⇄ Steering movement for all enemies (live comparison) |
| N / H | A* open/closed nodes / region grid + hierarchical route (both switch the overlay on) |
| T | Stress test: +10 hunters (try 40+ to see the frame-time and request-queue stats) |
| M / P / R | Mute / Pause / Restart |

Reading the enemies fairly: **"?"** above a unit = suspicious, **"!"** = it is chasing you. Hunters carry a visible
flashlight cone and red eyes in the dark. Hunters only see you inside the cone with a clear line of sight; they hear sprinting
(walking is almost silent, walls muffle sound).

## Enemies (all built from a few simple behaviours — the Complexity Fallacy)
| Unit | Behaviours | FSM |
|---|---|---|
| **Stalker** | patrol (A*), pursue, path-follow, wander | Patrol → Suspicious → Chase → Search → Patrol |
| **Flanker** | same as Stalker, but Chase aims at a slot point ahead/beside the player (blackboard) | same FSM |
| **Brute** | arrive (committed charge), mass-weighted separation (shoves others) | Patrol → Suspicious → Charge → Tired → Search |
| **Scout drone** (flying) | wander, evade, pings the blackboard so hunters turn Suspicious | Wander ⇄ Evade, Wander → Ping |

## Feature → Unit 3 concept map (for the viva)

### 1. Kinematic movement vs. steering behaviours
* **Where:** `Agent.integrate()` and the `Steering` object.
* **Kinematic (K on):** velocity is *assigned* directly from the desired direction — no acceleration, instant snap turns. Enemies look robotic, twitchy.
* **Steering (default):** every behaviour returns a *force* (`desiredVelocity − velocity`); forces are blended
  `wGoal·goal + wAvoid·avoid + wSep·separation`, clamped to `maxForce`, integrated into velocity (and heading turns at a limited rate).
* **Behaviours implemented:** `seek, flee, arrive` (slowing radius), `pursue` (predicts player position from velocity, with small aim noise),
  `evade`, `wander` (circle projection + jitter), `avoidObstacles` (3 feeler rays against wall tiles; push along the wall normal), `separation`
  (mass-weighted), path following (waypoint + look-ahead). Weights live in `CONFIG.steer`.
* **Debug:** green = goal (seek/pursue/arrive/wander), red = avoidance, blue = separation, white = final blended force; cyan circle = pursue prediction.

### 2. Pathfinding
* **A\*** (`AStar`): 8-directional, no corner cutting, octile heuristic, binary-heap open list, typed-array closed set (stamped, no clearing).
  Path **smoothing** by string-pulling with a body-width line-of-sight test (`smoothPath`, `Grid.clearLine`).
* **Hierarchical pathfinding** (`RegionGraph`): map split into 10×10-tile regions; portals (entrances) on shared borders; abstract graph with intra-region
  costs pre-computed by bounded floods. High-level A* gives the region route; **lazy refinement** then runs detailed A* only inside the current + next region
  (`PathService.refine`). Panel shows **nodes expanded flat vs. hierarchical** (flat is measured only while debug is on). Press **H** to see regions and each agent's route.
* **Time-aware / continuous-time ideas:**
  * `ReservationTable`: planned paths reserve **chokepoint** tiles (1-tile doors) with a time window; later planners pay a penalty (detour), and agents
    wait briefly at a reserved choke tile if its holder is closer (`choke waits` counter).
  * **Local replanning** (`Agent.pathTo`): a path is recomputed only if the target moved > 3 tiles, or a 0.4–0.8 s per-agent timer expired *and* the target moved,
    and a blocked next waypoint triggers only a **local repair** of the next few waypoints. Never every frame.
  * `PathService`: global request queue, **at most K=3 requests per tick**, priority ordered (chase > suspicious > search > patrol).

### 3. Decision making: FSM + weighted randomness
* `FSM` class; every enemy has one. Transitions use thresholds (an *awareness* value that fills while the player is visible and decays otherwise).
* **Weighted randomness** (`weightedPick`): in Chase within attack range → attack 70 % / reposition 20 % / retreat 10 %, re-rolled every ~1 s;
  patrol destination and pause lengths are weighted-random; search points are random around the last known position.
* **Fair, not cheating:** vision cone (100°, 260 px) + wall line of sight, hearing with wall muffling, human reaction delay (0.2–0.5 s × wave factor),
  aim error on pursuit prediction, limited memory (~5 s, then Search around the last *known* position). No unit ever reads the player's position unless it saw/heard it.
* **Telegraphing:** "?" / "!" markers, sound cues, visible cones, drone ping ring, Brute wind-up ring.

### 4. The Complexity Fallacy
Each unit is 3–4 tiny behaviours + a 4–5 state FSM. The "smart" feel is emergent:
* Flankers + a Stalker **pincer** you via a tiny blackboard (assigned angle slots) — no planner.
* A drone's ping shifts nearby hunters from Patrol to Suspicious — one rule, swarm-level effect.
* A Brute's rhythm (wind-up → charge → tired) is learnable; mass-weighted separation makes it plow through weaker units.
* Wave scaling (`CONFIG.wave`): faster reaction, bigger hearing radius, longer memory, better flank coordination, more patrol nodes, denser map, more units —
  speed only rises ≤ 14 %.

## Code map (`index.html`, top to bottom)
`CONFIG` · `Vec2`/helpers · `Input` · `Sfx` (Web Audio) · `Grid` + `buildArena` (procedural, flood-fill connectivity check) · `BinaryHeap`/`AStar`/`RegionGraph`/
`ReservationTable`/`PathService` · `Steering` · `FSM` · `Blackboard` · `Agent` · `Hunter`/`Stalker`/`Flanker`/`Brute` · `Drone` · `Player` · `Spawner` · `Game` ·
`Renderer` (pre-rendered map, line-of-sight fog) · `DebugOverlay` · `HUD` · game loop (fixed 60 Hz timestep with accumulator).

## Tuning
Everything is in `CONFIG` at the top of the script. Wave 1 = 2 stalkers + 1 drone (learnable); wave 2 adds a flanker; wave 3 adds a second flanker,
a Brute and more drones (tense). `CONFIG.wave(w)` controls the scaling.

## Verification performed (headless Chromium)
No console errors; 90 s random-walk sims and a bot that completes waves; zero ground enemies ever inside a wall tile; arenas for waves 1–8 always connected;
44 enemies: ~0.1–0.2 ms update, ~3 ms total frame, 60 FPS; the game can be won (extraction → next wave) and lost (0 hearts → game-over screen).
