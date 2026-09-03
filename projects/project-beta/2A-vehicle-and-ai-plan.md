# Member 2A Plan — Vehicle + Handler AI (Level 2: REDLINE)

Scope, straight from the two docs: vehicle acceleration/steering/braking/collision/drift, boost + heat, traffic/hazard spawning (reusing the shared chunk system), Handler AI states with telegraphed attacks, damage/health, the mountain-bridge ram that ends the level, and difficulty tuning. This plan breaks that into buildable chunks aligned to the team's Day1-3 / Week1 / Week2 / Week3 schedule.

---

## 1. What you actually own vs. what you consume

You **build**: vehicle controller, Handler AI (in Level 2), damage/health system, the bridge-ram trigger, difficulty tuning.

You **consume** (don't rebuild these — coordinate with owners):
- `Input` — shared keyboard/mouse system (1A)
- `GameState` — shared health/distance/boost fields (1A)
- Chunk streamer — traffic/hazard spawning reuses this pool-and-recycle system, written by 1A for Level 1 (2B owns the highway art side of it)
- `Level` interface — your `Level02` class needs `init(scene, assets, input)`, `update(dt, gameState)`, `teardown()`

Before you write a line of gameplay code, get explicit agreement on:
- What fields live on `GameState` for Level 2 (health, heat, distance, boost-cooldown) — you'll be the biggest consumer/writer of these
- The exact `Input` API you'll call for throttle/steer/handbrake/boost
- Which physics engine the team picked (Rapier or cannon-es) — don't start prototyping in the other one

## 2. Architecture — two state machines, one physics body

Keep vehicle *control* and Handler *behavior* as separate FSMs that both read/write `GameState` each frame, rather than coupling them directly. This is what lets you tune them independently and lets 3A later swap in the combat FSM without touching your code.

```
Level02 (implements Level interface)
├── VehicleController      // player car: physics + input mapping
├── HandlerAI               // pursuer FSM
├── HazardSpawner           // wraps shared chunk streamer
├── DamageSystem             // shared by both car and Handler interactions
└── BridgeRamTrigger         // level-end scripted moment
```

**Vehicle FSM** (arguably overkill as a full FSM — a simple physics controller with modifiers is enough):
- Driving → Drifting (handbrake held) → Boosting (heat < max, boost held) → Disabled (health = 0)

**Handler FSM** (this is the one worth diagramming properly):
- `Approach` → `Harass` (ram/tyre-shot/drone attacks on cooldown) → `Telegraph` (brief wind-up before each attack, visible to player) → `Recover` (brief vulnerability/cooldown after attack) → `RamFinale` (scripted, triggers the bridge sequence)

## 3. Build order (do NOT build in doc order — build in this order)

1. **Static car that drives** — box collider, WASD → force/torque, no AI, no traffic. Prove input → physics → screen works end to end.
2. **Handler as a dumb follower** — no attacks yet, just a second physics body that path-follows/chases the player at a fixed distance. Proves the FSM skeleton and that two bodies can coexist without exploding.
3. **One real attack (ram)** — telegraph → execute → recover, with damage hooking into `GameState.health`. This is your riskiest single feature; get it working greybox before anything else.
4. **Traffic/hazard spawning** via the chunk streamer — start with static obstacles, add moving traffic once static works.
5. **Boost/heat system** — this is nearly free once throttle input exists; a heat float that drains on boost-hold and regenerates otherwise.
6. **Remaining attack types** (tyre shots, drones) — additive once the first attack's telegraph/execute/recover pattern is proven.
7. **Difficulty tuning pass** — attack frequency, Handler aggression curve over distance/time, traffic density.
8. **Bridge-ram finale** — scripted trigger volume near the level's end chunk; hands off to 3A's transition.

Building the ram attack before traffic/hazards is deliberate: it's the piece most likely to reveal that your damage/health design doesn't work, and you want that surprise in week 1-2, not week 3.

## 4. Detailed breakdown by system

### Vehicle controller
- Acceleration/braking: simple forward force along the car's local axis, scaled by throttle input; separate reverse handling
- Steering: angular velocity or torque scaled by speed (less steering authority at high speed feels better than a fixed turn rate)
- Drift: on handbrake, reduce lateral friction on rear wheels/rear of chassis so the car slides instead of gripping
- Collision: let the physics engine handle car-vs-geometry; you handle car-vs-Handler and car-vs-hazard as *game events* (damage), not just physics response
- Suggested starting values: tune on a flat greybox test track, not the real highway geometry — isolates handling feel from level design

### Boost / heat
- `heat` float 0-100 on `GameState`, drains at a fixed rate while boost is held, regenerates when not boosting, boost disabled above a heat ceiling until it cools below a re-enable threshold (avoids instant re-tap spam)
- Boost = temporary force multiplier + maybe FOV kick for feedback

### Traffic & hazard spawning
- Don't write a second spawner — extend/parameterize the shared chunk streamer with a "hazard density" and "traffic lane occupancy" table per chunk type
- Traffic AI can be dead simple: fixed-lane, fixed-speed, no avoidance — the Handler is the only "smart" actor in this level
- Hazards (roadworks, toll booms, jack-knifed tanker) are static or simple scripted-trigger obstacles, not physics-simulated

### Handler AI — the FSM in detail
| State | Trigger | Behavior | Exit |
|---|---|---|---|
| Approach | level start / after Recover | close distance to player | reaches harass range |
| Harass | in range, cooldown elapsed | pick an attack, hold formation | attack selected |
| Telegraph | attack selected | visible wind-up (headlight flash, engine rev, drone lock-on beep) — **must be readable and dodgeable** | telegraph timer ends |
| Execute | telegraph ends | perform ram/tyre-shot/drone attack | attack resolves (hit or dodged) |
| Recover | attack resolves | brief vulnerable/cooldown window | cooldown ends → Approach |
| RamFinale | bridge trigger volume reached | scripted, takes control from FSM | level end |

Key design constraint from the pitch doc: **attacks must be telegraphed clearly enough to dodge**. Budget real time for playtesting telegraph timing — too short feels unfair, too long feels sluggish. This is the #1 thing that makes or breaks whether Level 2 feels good vs. cheap.

### Damage / health
- `GameState.playerHealth` (or `carIntegrity`) as a shared field — UI (3B) reads it, you write to it
- Each Handler attack type defines its own damage amount; traffic/hazard collisions can also chip health for consistency
- Health = 0 → fail state (car destroyed), per the pitch's own fail-state definition

### Bridge-ram finale
- A trigger volume near the end of the highway chunk sequence forces Handler into `RamFinale`, plays a short scripted ram, and hands off to 3A's Level2→Level3 transition (car goes over the edge, cut to riverbank)
- Coordinate the handoff contract with 3A explicitly: what does your code call, what does theirs expect to receive (probably just "trigger transition, Level02.teardown() runs")

## 5. Difficulty tuning checklist
- Handler approach speed vs. player max speed (should be close enough to feel relentless, not off-screen)
- Attack cooldown length and telegraph duration (playtest, don't guess)
- Traffic density vs. Handler attacks — don't stack "dodge Handler" and "dodge traffic" demands in the same window without intent
- Heat/boost balance — boost should meaningfully help escape Handler pressure, not be decorative

## 6. Mapping onto the team's 3-week schedule
- **Day 1-3**: agree GameState/Input contract with 1A; box-car prototype driving on a flat plane
- **Week 1 (Alpha)**: Handler dumb-follower FSM working; car integrated into shared Level02 skeleton
- **Week 2 (Beta)**: one full attack (ram) working end-to-end with damage; boost/heat in; traffic spawning via chunk streamer; this is what gets shown at the beta checkpoint
- **Week 3 (Content)**: remaining attack types, full difficulty tuning pass, bridge-ram finale scripted and handed off to 3A, polish

## 7. Rubric it pays for (so you know what to protect if time runs short)
Per the pitch's rubric map, your work is the **primary owner** of Gameplay & Experience (25% — the single biggest category) and a **supporting** owner of Control & Playability and Innovation. If time gets tight, protect the ram attack + telegraph feel before you protect the second and third attack types — one good, readable attack beats three sloppy ones for this category.

## 8. Known risk (name it now, per the pitch's own risk section)
Combat/AI feel is inherently the hardest thing to get right by feel rather than by spec. Greybox it early (week 1-2, not week 3), and if the FSM feels bad, the documented fallback is to simplify Handler behavior to fewer, more scripted attack beats rather than a fully reactive AI — that's an acceptable cut per the team's own cut list, not a failure.
