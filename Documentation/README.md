# Procedural Dungeon Roguelike — Design & Build Guide

**Project:** Turn-based roguelike with wave-function-collapse level generation, fully deterministic seeds, and a data-driven ability system
**Status of this document:** planning + reference

---

## Table of contents

1. [Executive summary and scope](#1-executive-summary-and-scope)
2. [Reality check](#2-reality-check)
3. [Level generation architecture](#3-level-generation-architecture)
4. [WFC in practice](#4-wfc-in-practice)
5. [Validation and repair](#5-validation-and-repair)
6. [Determinism](#6-determinism)
7. [The turn scheduler](#7-the-turn-scheduler)
8. [The ability system](#8-the-ability-system)
9. [Effect resolution](#9-effect-resolution)
10. [Automated balance testing](#10-automated-balance-testing)
11. [Core roguelike systems](#11-core-roguelike-systems)
12. [Tech stack and setup](#12-tech-stack-and-setup)
13. [Repository layout](#13-repository-layout)
14. [Milestone ladder](#14-milestone-ladder)
15. [Reference implementations](#15-reference-implementations)
16. [Testing](#16-testing)
17. [Stretch goals](#17-stretch-goals)
18. [References](#18-references)

---

## 1. Executive summary and scope

### The original statement

> Turn-based roguelike with wave-function-collapse level generation, fully deterministic seeds, and a data-driven ability system.

Five findings reshape this:

1. **WFC cannot guarantee a playable level, and this is a known limitation rather than an implementation problem.** The research literature is direct about it: WFC "cannot guarantee solvability and does not scale well to larger levels." It is a *local* constraint solver, and a roguelike needs *global* guarantees — the exit must be reachable, there must be exactly one down-staircase, the key must be obtainable before its door. Papers published this year still describe guaranteeing global properties as an open problem. See section 2.1.
2. **The fix is architectural: WFC is a decorator, not an architect.** Generate the dungeon's macro structure with a graph-based method that gives you connectivity by construction, then use WFC to fill each region with locally-coherent detail. The most informative result in the literature compares a maze domain — where connectivity emerges from local rules and WFC works well — against a Zelda-style domain needing exactly one player, key, and door, where local patterns visibly fail. A roguelike is the Zelda case. See section 3.1.
3. **Canonical WFC is non-backtracking and greedy, so a contradiction means starting over.** Contradiction rate depends entirely on your tileset and grows with grid size. Scoping WFC to individual rooms rather than the whole level means a failure costs one room's worth of work instead of the entire dungeon. See section 4.3.
4. **"Deterministic seeds" needs hierarchical RNG streams, not one seeded generator.** With a single stream, adding one random call to level generation shifts every loot drop and every enemy placement for every existing seed. And the RNG state must live in the save file, or reloading becomes a reroll. See section 6.2.
5. **A data-driven ability system becomes a bad programming language in YAML unless you design against it.** The escalation is predictable: a damage number, then scaling, then conditions, then iteration. The answer is composable typed effect nodes — expressive through composition without being Turing-complete in data. See section 8.2.

And one payoff worth stating up front: **deterministic replay makes automated balance testing possible**, which is the only realistic way to balance a combinatorial ability system. That's the argument that justifies the determinism work beyond "seeds are nice." See section 10.

### Revised project statement

> A turn-based roguelike with a two-tier generator — graph-based macro layout guaranteeing connectivity and quest structure, WFC filling room interiors with bounded retry — validated and repaired before play; hierarchical seeded RNG with full state in saves and seed-plus-input replay; an energy-based turn scheduler with deterministic tie-breaking; and a composable-effect ability system with a defined resolution pipeline, driving a headless simulation harness for automated balance analysis.

### Explicit non-goals

- **Not real-time.** Turn-based is in the statement and it's what makes determinism tractable.
- **Not 3D.** Grid-based 2D. Every problem here is harder in 3D for no design benefit.
- **Not multiplayer.** Determinism would enable lockstep, but it's a separate project.
- **Not modding-first in v1.** Design the data format so mods are possible later; don't build a mod API yet.
- **Not procedural narrative.** Quest structure from templates, not generated stories.

---

## 2. Reality check

### 2.1 What WFC is and isn't ★

WFC takes local adjacency constraints — usually learned from an example — and produces larger output where every local neighbourhood is plausible. It's a constraint solver, and it's genuinely good at what it does.

**It has no concept of global structure.** It cannot know that:

- Every walkable tile should be reachable from the entrance
- There must be exactly one down-staircase
- The boss room should be far from the start
- A key must be obtainable before the door it opens
- The level should take roughly N turns to traverse

None of those are expressible as tile adjacency rules, because none of them are local properties.

The clearest result in the literature compares two domains. In a **maze** domain, where connectivity emerges naturally from local spatial relationships, WFC does well. In a **Zelda-style** domain, which "depends on global constraints such as exactly one player, key, and door," local patterns visibly fall short.

**A roguelike dungeon is the Zelda case**, not the maze case.

Active research is still attacking this — combining WFC with evolutionary search, with reinforcement learning, with graph structure. A paper from within the last week still frames guaranteeing playability as the open challenge. **This is not something you'll solve in your generator.**

### 2.2 The failure modes

| Symptom | Cause |
|---|---|
| **Unreachable exit** | WFC has no connectivity concept (§2.1) |
| Level generation occasionally hangs | Contradiction plus restart loop on a large grid (§4.3) |
| Generation time spikes unpredictably | Same — restart cost scales with grid size |
| **Same seed produces different dungeons** | Non-deterministic iteration or unstable RNG order (§6.3) |
| Same seed, different loot | Shared RNG stream shifted by a level-gen change (§6.2) |
| Reloading a save rerolls drops | RNG state not saved (§6.4) |
| Two actors act in different orders across runs | No deterministic tie-break in the scheduler (§7.2) |
| Ability data grows conditionals, then loops | The YAML-as-language trap (§8.2) |
| An ability kills its caster mid-resolution and crashes | No defined effect resolution order (§9.3) |
| Reactive abilities cause infinite cascades | No depth limit (§9.4) |
| One build dominates everything | No automated balance testing (§10) |

### 2.3 What makes this project good

Setting aside the WFC caveat, the combination is unusually well-suited:

- **Turn-based grid means no floats** in the simulation, which removes the entire cross-platform determinism problem that plagues real-time games
- **Determinism plus replay gives you** regression testing, bug reports that reproduce exactly, verifiable leaderboards, and the balance harness in §10
- **Data-driven abilities plus a simulation harness** means you can actually measure balance rather than guess

Those reinforce each other. It's a coherent design.

---

## 3. Level generation architecture

### 3.1 Two tiers ★

```
Tier 1 — MACRO (graph-based)
    rooms as nodes, corridors as edges
    connectivity guaranteed BY CONSTRUCTION
    quest structure: entrance, exit, key/door, boss placement
        ↓
Tier 2 — MICRO (WFC per region)
    fill each room's interior with locally-coherent tiles
    constrained by the doorways the macro tier placed
        ↓
Tier 3 — VALIDATE AND REPAIR
    flood fill, reachability, quest ordering
    repair or reject-and-reseed
```

The macro tier is where playability comes from. Build the room graph first — a spanning tree guarantees connectivity, then add a few extra edges for loops — and place quest elements on that graph where the constraints are trivially checkable.

```python
def generate_macro(rng, params):
    rooms = place_rooms(rng, params)            # e.g. BSP, or Poisson-disc + Delaunay

    # A spanning tree over the rooms guarantees every room is reachable.
    graph = minimum_spanning_tree(rooms)

    # Extra edges create loops, which make a dungeon feel less like a tree.
    for _ in range(params.extra_edges):
        graph.add_edge(rng.choice(candidate_edges(rooms, graph)))

    # Quest placement operates on the GRAPH, where distance is meaningful.
    entrance = graph.leaf_farthest_from_center()
    exit_    = graph.node_farthest_from(entrance)
    boss     = exit_.neighbor_toward(entrance)

    # Key must be on the entrance side of the door — checkable on a graph,
    # not expressible as a tile adjacency rule.
    door = graph.edge_on_path(entrance, exit_, fraction=0.6)
    key  = rng.choice(graph.nodes_reachable_without(door, from_=entrance))

    return MacroLayout(rooms, graph, entrance, exit_, boss, key, door)
```

Every one of those quest constraints is a graph property and trivially verifiable. None of them is expressible in WFC.

### 3.2 Where WFC earns its place

Within a room, WFC is excellent. It gives you interiors that look hand-authored — coherent wall runs, plausible furniture clusters, rubble that falls where rubble falls — from a small set of example patterns.

```
Macro tier says:   "Room at (12,8), 9×7, doorways on N and E, theme=library"
WFC tier does:     fills the 9×7 interior with library tiles, respecting
                   the doorway positions as fixed constraints
```

That's WFC used for exactly what it's good at: local texture, constrained at the boundary by something that already guaranteed the global properties.

### 3.3 Alternatives worth knowing

| Method | Good for | Roguelike fit |
|---|---|---|
| **BSP** | Rectangular rooms, guaranteed connectivity | Classic, reliable, a bit regular-looking |
| **Cellular automata** | Organic caves | Needs a connectivity pass — islands are common |
| **Drunkard's walk** | Winding caves | Simple; poor control |
| **Graph grammar** | Quest-structured dungeons | **Strong fit** — rules like "replace a corridor with lock-key-room" |
| **Room templates + stitching** | Hand-authored quality at scale | Very strong, and underused |
| **WFC** | Local texture and detail | Tier 2 only |

**Hand-authored room templates stitched by a graph** is the approach that most reliably produces dungeons that feel designed, and it's worth prototyping alongside WFC. A hybrid — templates for special rooms, WFC for filler — is often the best of both.

---

## 4. WFC in practice

### 4.1 The algorithm

```
1. Every cell starts in superposition (all tiles possible)
2. Pick the cell with lowest entropy (fewest remaining options)
3. Collapse it to one tile, weighted by the tileset's frequencies
4. Propagate: remove now-impossible options from neighbours, transitively
5. Repeat until all cells are collapsed, or a cell has zero options
```

Step 5's failure case is the contradiction.

### 4.2 Tileset authoring is the real work

Most of the effort in a WFC generator is not the algorithm — it's authoring tiles and adjacency rules that produce good output without constant contradictions.

Two modes:

- **Simple tiled**: you declare each tile's edge sockets and which sockets connect. Explicit, controllable, tedious.
- **Overlapping**: learn N×N patterns from an example bitmap. Less authoring, less control, more contradictions.

**Simple tiled is the better choice for a roguelike**, because you need to guarantee that doorway tiles connect to floor tiles and that walls are closed — and you want to state that explicitly rather than hope the example taught it.

```yaml
tiles:
  - id: floor
    sockets: { n: open, e: open, s: open, w: open }
    weight: 10
  - id: wall_ns
    sockets: { n: wall, e: solid, s: wall, w: solid }
    weight: 3
  - id: pillar
    sockets: { n: open, e: open, s: open, w: open }
    weight: 1
    max_count: 4          # a global constraint the base algorithm lacks
```

That `max_count` field is a useful extension: cap how many of a tile can appear. It's a crude global constraint and it prevents the "entire room is pillars" outcome that pure local rules permit.

### 4.3 Contradictions ★

Canonical WFC is **non-backtracking and greedy**. On contradiction, the standard response is to throw everything away and restart with a new seed — which is wasteful, and the waste grows with grid size.

Three mitigations, and you want all three:

**Scope WFC to rooms, not levels.** A 9×7 room that fails costs a millisecond to retry. A 100×100 level that fails costs a lot more, and the probability of hitting a contradiction somewhere grows with area.

**Bounded backtracking.** Keep a stack of decision points; on contradiction, undo the last collapse and try the next-best option. Cap the budget, because worst-case backtracking is exponential.

```python
def collapse_region(rng, region, tileset, max_restarts=8, max_backtrack=64):
    for attempt in range(max_restarts):
        # Derive a sub-seed so retries are deterministic and don't
        # disturb any other RNG stream (§6.2).
        sub = rng.derive(f"region/{region.id}/attempt/{attempt}")
        result = wfc_solve(sub, region, tileset, backtrack_budget=max_backtrack)
        if result.ok:
            return result

    # Every attempt failed. Fall back rather than hang.
    return fallback_fill(region, tileset)      # §4.4
```

**A fallback that always succeeds.** After N failures, fill the region with a guaranteed-valid simple pattern — plain floor with walls at the boundary. It's less interesting and it is always playable, which matters more.

**Measure your contradiction rate per tileset.** It's a property of the tileset, not the algorithm, and a tileset with a 30% failure rate needs its adjacency rules fixed, not more retries. Track it in CI over thousands of generations.

### 4.4 Constrain from the boundary in

The macro tier already decided where doorways are. Pre-collapse those cells before running WFC:

```python
def prepare_region(region, macro):
    wave = Wave(region.width, region.height, tileset)
    for door in macro.doorways_for(region):
        wave.collapse_to(door.x, door.y, tile="doorway", orientation=door.facing)
    for x, y in region.boundary():
        if not region.is_doorway(x, y):
            wave.restrict(x, y, to=WALL_TILES)
    return wave
```

This both enforces the macro structure and **reduces contradictions**, because pre-collapsed cells shrink the search space. Constraining the boundary first is one of the cheapest reliability wins available.

---

## 5. Validation and repair ★

### 5.1 Validate every level, always

Even with a graph-guaranteed macro tier, validate. The WFC fill can block a doorway, a repair can disconnect something, and a bug in your own code will eventually produce an unplayable level.

```python
def validate(level, macro):
    issues = []

    # The one that matters most.
    reachable = flood_fill(level, from_=macro.entrance)
    if macro.exit not in reachable:
        issues.append(Issue.EXIT_UNREACHABLE)

    orphans = level.walkable_tiles() - reachable
    if orphans:
        issues.append(Issue.ORPHANED_REGIONS, count=len(orphans))

    # Quest ordering: the key must be reachable without passing the door.
    without_door = flood_fill(level, from_=macro.entrance, blocked={macro.door})
    if macro.key not in without_door:
        issues.append(Issue.KEY_BEHIND_ITS_OWN_DOOR)

    if level.count(TILE_DOWNSTAIRS) != 1:
        issues.append(Issue.WRONG_STAIRCASE_COUNT)

    if path_length(macro.entrance, macro.exit) < params.min_path:
        issues.append(Issue.LEVEL_TOO_SHORT)

    return issues
```

**The key-behind-its-own-door check is the one that will actually fire**, and it's a genuinely embarrassing bug to ship.

### 5.2 Repair before rejecting

Regenerating is expensive and loses good output. Most issues are locally fixable:

| Issue | Repair |
|---|---|
| Orphaned region | Carve a corridor to the nearest reachable tile |
| Small orphan (< N tiles) | Fill it in as solid rock |
| Doorway blocked by WFC fill | Force the tile back to floor |
| Too many staircases | Remove all but the intended one |
| Path too short | Add a detour, or reseed |

```python
def repair(level, macro, rng):
    reachable = flood_fill(level, from_=macro.entrance)
    for region in connected_components(level.walkable_tiles() - reachable):
        if len(region) < params.min_orphan_size:
            level.fill(region, TILE_ROCK)              # too small to bother
        else:
            carve_corridor(level, nearest_pair(region, reachable), rng)
        reachable = flood_fill(level, from_=macro.entrance)   # recompute
    return level
```

Recomputing reachability after each carve matters — carving to one orphan can connect several, and skipping the recompute carves redundant corridors.

**If repair fails, reseed with a derived seed and try again**, with a hard attempt cap and a fallback to a simple guaranteed-valid layout. A roguelike that occasionally fails to start a floor is worse than one that occasionally generates a boring floor.

### 5.3 Validate in CI at volume

```python
def test_generation_always_valid():
    failures = []
    for seed in range(100_000):
        level, macro = generate(seed)
        issues = validate(level, macro)
        if issues:
            failures.append((seed, issues))
    assert not failures, f"{len(failures)} invalid levels; first: {failures[0]}"
```

A hundred thousand seeds is minutes of compute and it finds the 1-in-10,000 case that would otherwise be a player's bug report. **Record the failing seed** — determinism means it reproduces exactly, which turns a generation bug from a mystery into a debugging session.

Track distributions too, not just validity: room count, path length, dead-end count, item count. A change that makes every level valid and identical is also a bug.

---

## 6. Determinism ★

### 6.1 What "fully deterministic seeds" has to mean

Same seed must reproduce: the dungeon, item placement and identity, enemy placement and types, enemy AI decisions, every random roll in combat, and shop inventories. If any one of those drifts, "seed sharing" doesn't work and neither does replay.

**Turn-based grid games have a large advantage here: you can avoid floating point entirely.** Integer positions, integer damage, integer-or-rational probabilities. That sidesteps the cross-platform float determinism problem completely — a luxury real-time games don't have.

Make it a rule and enforce it with a lint pass over the simulation code.

### 6.2 Hierarchical RNG streams ★

With one global RNG, every consumer shares a sequence. Add a single `rng.next()` call to level generation and **every downstream draw shifts** — loot, enemies, everything. Every existing seed now produces a different game.

That makes the seed worthless across versions and makes iteration painful.

```python
class Rng:
    def __init__(self, state: int):
        self.state = state

    def derive(self, path: str) -> "Rng":
        """Independent sub-stream. Adding draws in one subsystem cannot
        shift any other subsystem's sequence."""
        h = self.state
        for ch in path:
            h = splitmix64(h ^ ord(ch))
        return Rng(h)

# Independent, and stable under change:
rng_macro  = master.derive("level/3/macro")
rng_wfc    = master.derive("level/3/wfc")
rng_loot   = master.derive("level/3/loot")
rng_enemy  = master.derive("level/3/enemies")
rng_combat = master.derive("combat")
```

Nest the paths as finely as the structure allows — `level/3/wfc/room/7` means adding a retry to room 7 can't disturb room 8.

**Version the generator.** A seed is only meaningful against a specific generator version, so the shareable identifier is `seed + version`, and old versions should stay resolvable if you want players to replay old runs.

### 6.3 The other determinism killers

| Killer | Fix |
|---|---|
| **Dict/set iteration order** | Sort keys, or use ordered structures, anywhere the sim iterates |
| **Object identity or address in sorting** | Stable integer IDs only |
| Unstable sorts | Total ordering with an ID tiebreak |
| Wall-clock time | Turn counter only |
| Parallelism in the sim | Keep the simulation single-threaded |
| Unseeded library RNG | Ban the global `random` module in sim code |
| Float arithmetic | §6.1 — avoid entirely |

The dict-iteration one catches people constantly, especially in Python and in any language where a hash set's order depends on insertion history or memory addresses.

### 6.4 RNG state belongs in the save file ★

If a save restores the world but not the RNG state, a player can save before opening a chest, reload, and reroll the contents. That's save-scumming, and whether you consider it a bug depends on your design — but it should be a *decision*, not an accident.

```python
@dataclass
class SaveGame:
    version: int
    seed: int
    turn: int
    world: WorldState
    rng_states: dict[str, int]      # every active stream
    input_log: list[Action]         # for replay verification (§6.5)
```

Traditional roguelikes go further and delete the save on load, so there's only ever one. That's a design stance; make it deliberately.

### 6.5 Replay is nearly free, and worth a lot

Seed plus input log fully reproduces a run. That single fact gives you:

- **A regression suite** — replay recorded runs after a change and see what diverges (§16.2)
- **Bug reports that reproduce exactly** — "here's my seed and inputs"
- **Verifiable leaderboards** — replay the run server-side
- **The balance harness** (§10)

```python
@dataclass
class Replay:
    seed: int
    game_version: str
    actions: list[Action]           # one per player turn
    checkpoints: dict[int, int]     # turn → world state hash
```

The periodic checkpoints are what make divergence *findable*. Without them you know the replay broke; with them you binary-search to the exact turn where it did.

---

## 7. The turn scheduler

### 7.1 Energy-based scheduling

Fixed round-robin can't express speed differences. The standard roguelike solution gives each actor an energy pool:

```python
ENERGY_PER_TURN = 100

def advance(actors):
    while True:
        ready = [a for a in actors if a.energy >= ENERGY_PER_TURN]
        if ready:
            actor = min(ready, key=lambda a: (-a.energy, a.id))   # §7.2
            action = actor.take_turn()
            actor.energy -= action.cost
            return action
        for a in actors:
            a.energy += a.speed          # speed 100 = normal, 200 = double
```

A priority queue keyed by next-action time is the efficient form once you have many actors, but the energy loop is easier to reason about and fast enough for typical roguelike counts.

### 7.2 Deterministic tie-breaking ★

Two actors ready on the same tick must act in the same order every time, or the same seed diverges.

```python
actor = min(ready, key=lambda a: (-a.energy, a.id))
```

The `a.id` tiebreak is load-bearing. Without it, the order depends on list ordering, which depends on spawn order, which depends on iteration order somewhere else — and a subtle change three systems away silently reorders combat.

**Actor IDs must be assigned deterministically** too: from a counter advanced in a defined order, never from object identity or memory address.

### 7.3 The input problem

The turn loop must pause mid-execution waiting for player input, which is awkward control flow.

| Approach | Notes |
|---|---|
| **Blocking loop** | Simplest; fine for a desktop game with no animation |
| **State machine** | `WAITING_FOR_INPUT` / `RESOLVING` / `ANIMATING` |
| **Coroutines / generators** | `action = yield RequestInput()` — reads naturally |
| **Command queue** | The loop consumes actions; the UI produces them. **Best for replay** — the queue *is* the input log. |

The command queue is the one that composes with everything else: replay becomes feeding the recorded queue, and the AI-vs-AI harness in §10 becomes feeding a generated one.

### 7.4 Animation must not affect the simulation

Animation runs at frame rate; simulation runs on turns. Keep them completely separate: the simulation resolves instantly and emits an event log; presentation plays the events back over time.

```python
result = simulation.execute(action)     # instant, deterministic
presentation.enqueue(result.events)     # plays out over frames
```

If the simulation ever waits for an animation, timing has entered your determinism model and you've lost replay.

---

## 8. The ability system

### 8.1 What data-driven has to deliver

- Designers add abilities without recompiling
- Abilities compose from reusable pieces
- Validated at load, not at the moment a player triggers them
- Deterministic
- Legible enough to reason about balance

### 8.2 The YAML-as-language trap ★

The escalation is predictable and happens to everyone:

```yaml
# Week 1
fireball: { damage: 10 }

# Week 4
fireball: { damage: 10, scaling: { intelligence: 0.5 } }

# Week 8
fireball:
  damage: 10
  condition: { if: "target.has_status('wet')", then: 0, else: 10 }

# Week 12 — you have invented a programming language, badly
fireball:
  effects:
    - for_each: "targets_in_radius(2)"
      do:
        - if: "rng() < 0.3"
          then: [ apply_status: burning ]
```

At that point you have a language with no type checker, no debugger, no error messages, and no tooling.

**The fork:**

| | **Composable effect nodes** | **Embedded scripting (Lua)** |
|---|---|---|
| Expressiveness | Bounded — new node types need code | Unbounded |
| Validation | Static, at load | Runtime only |
| Determinism | Guaranteed by construction | Requires discipline |
| Analyzability | Queryable — "which abilities apply burning?" | No |
| Failure mode | "Unknown node type" at load | Crash mid-combat |
| Balance tooling | Easy — the data is structured | Hard |

**Composable nodes are the right default.** Each node type is engine code; the data only composes them. That's expressive through composition without being Turing-complete in data.

```yaml
abilities:
  fireball:
    name: "Fireball"
    cost: { mana: 8 }
    cooldown: 3
    targeting: { type: area, shape: circle, radius: 2, range: 6, los: true }
    effects:
      - type: for_each_target
        children:
          - type: damage
            amount: 10
            scaling: { intelligence: 0.5 }
            damage_type: fire
          - type: branch
            condition: { type: has_status, status: wet }
            if_true:  [ { type: remove_status, status: wet } ]
            if_false: [ { type: apply_status, status: burning, duration: 3 } ]
      - type: modify_terrain
        tiles_in_radius: 2
        from: grass
        to: scorched
```

Every `type` maps to a class in code. Adding a genuinely new *kind* of effect means writing a node; combining existing kinds means editing data.

### 8.3 Validate at load

```python
def validate_ability(data, registry):
    errors = []
    for effect in walk_effects(data):
        node = registry.get(effect["type"])
        if node is None:
            errors.append(f"unknown effect type '{effect['type']}'")
            continue
        errors += node.validate_params(effect)      # types, ranges, references
    if references_status(data) not in registry.statuses:
        errors.append("references an undefined status")
    return errors
```

**Fail the build, not the player's combat.** Load every ability at startup (and in CI) and refuse to run with an invalid one. A typo'd status name should be a build failure, not a silent no-op that a player eventually notices.

### 8.4 Structured data is queryable

A real benefit of the node model over scripting: you can ask questions about your own content.

```python
"Which abilities apply burning?"          → walk the trees, filter
"Which abilities scale with intelligence?" → same
"What's the damage range across all fire abilities at level 5?"
"Which statuses are applied but never removed by anything?"
```

That last query finds real design bugs. With Lua, none of these are answerable without running the game.

---

## 9. Effect resolution

### 9.1 Order must be explicit

"Apply 10 damage" hides several questions: before or after armour? Does an on-hit trigger fire before or after the damage lands? What if the target dies partway through a multi-effect ability?

Define a pipeline and document it:

```
1. VALIDATE       cost payable, target legal, not on cooldown
2. COMMIT         pay costs, start cooldown
3. RESOLVE TARGETS  snapshot the target list NOW  ← §9.2
4. PRE-EFFECT     gather modifiers (buffs, resistances, on_before_hit)
5. COMPUTE        final values from base + modifiers
6. APPLY          mutate state, emit events
7. POST-EFFECT    on_hit, on_damaged, on_kill reactions
8. CASCADE        resolve reactions, depth-limited  ← §9.4
9. CLEANUP        process deaths, expire statuses   ← §9.3
```

### 9.2 Snapshot targets before applying

A chain lightning that kills its first target shouldn't lose its remaining targets because the list was recomputed mid-resolution. Resolve the target list once, at step 3, and hold it — checking validity per target as you go.

### 9.3 Deferred death ★

An entity dying partway through resolution is the classic source of crashes and inconsistency. If death is processed immediately, subsequent effects operate on a destroyed object.

**Mark dead, process at cleanup:**

```python
def apply_damage(target, amount, ctx):
    target.hp -= amount
    ctx.emit(DamageEvent(target, amount))
    if target.hp <= 0 and not target.pending_death:
        target.pending_death = True         # mark only
        ctx.emit(DeathEvent(target))        # triggers on_kill reactions
        # The entity stays in the world until step 9.

def cleanup(ctx):
    for e in ctx.entities_with_pending_death():
        drop_loot(e, ctx.rng.derive(f"loot/{e.id}"))
        ctx.world.remove(e)
```

Effects after the death still resolve against a coherent object, on-kill triggers fire in a defined place, and nothing dangles.

### 9.4 Cascade depth limits ★

Reactive abilities create cascades: A's on-hit damages B, B retaliates, damaging A, whose on-hit fires again. Two reactive abilities can loop indefinitely.

```python
MAX_CASCADE_DEPTH = 8

def resolve_reaction(reaction, ctx):
    if ctx.depth >= MAX_CASCADE_DEPTH:
        ctx.log_warning("cascade depth limit hit", reaction=reaction.id)
        return                                   # stop, don't crash
    ctx.depth += 1
    execute_effects(reaction.effects, ctx)
    ctx.depth -= 1
```

Log it when it fires. A depth limit reached in normal play means a content bug — two abilities that shouldn't chain — and the log is how you find out.

### 9.5 Events are the seam

The simulation emits events; everything else consumes them. Presentation animates them, the log describes them, achievements watch them, and the replay verifier hashes them.

```python
@dataclass
class Event:
    turn: int
    type: EventType
    actor: EntityId
    target: EntityId | None
    data: dict
```

Keeping the simulation emitting rather than calling means adding a new consumer never touches simulation code — and it keeps §7.4's separation intact.

---

## 10. Automated balance testing ★

### 10.1 Combinatorial balance can't be hand-tested

With 60 abilities, 80 items, and 20 enemy types, the combination space is astronomically larger than you can play. You cannot find the dominant build by playing, because you'd have to think of it first.

**Deterministic simulation plus a command queue means you can run the game headless, thousands of times.** This is the payoff that justifies everything in §6.

### 10.2 The harness

```python
def run_simulation(seed, strategy, max_turns=20_000):
    game = Game(seed)
    while not game.over and game.turn < max_turns:
        game.execute(strategy.choose_action(game.observable_state()))
    return RunResult(
        seed=seed,
        won=game.won,
        depth=game.max_depth,
        turns=game.turn,
        cause_of_death=game.death_cause,
        abilities_used=game.ability_use_counts,
        items_held=game.final_inventory,
        damage_by_source=game.damage_attribution,
    )
```

Strategies range from trivial to informative:

| Strategy | Tells you |
|---|---|
| **Random** | Is the game survivable at all by accident? A floor on difficulty. |
| **Greedy** | Attack the nearest, use the highest-damage ability. A baseline. |
| **Scripted builds** | "Always take fire abilities" — measures build viability |
| **Simple search** | Shallow lookahead. Approximates a competent player. |

### 10.3 What the numbers tell you

Run ten thousand and look at the distributions:

```
Win rate by strategy:     random 0.2% · greedy 12% · fire-build 31% · ice-build 4%
                          ↑ a 27-point spread means fire dominates

Death causes:             starvation 3% · floor-4 boss 34% · poison 21%
                          ↑ one encounter is a wall

Ability usage:            Fireball 34% of all uses · Ice Shard 0.2%
                          ↑ Ice Shard is dead content

Damage attribution:       Fireball 41% of all player damage dealt

Run length:               median 4,200 turns · p95 11,000
                          ↑ the tail is probably grinding
```

**An ability used 0.2% of the time is dead content.** Either it's weak, or its niche never comes up. Either way it's a design bug you'd otherwise find only from a forum post.

### 10.4 Run it in CI

A nightly job over ten thousand runs, with the aggregate statistics tracked over time. A balance change that moves the fire-build win rate from 31% to 58% shows up the next morning rather than at launch.

At a couple of seconds per headless run and a handful of cores, ten thousand runs is well under an hour.

---

## 11. Core roguelike systems

### 11.1 Field of view

**Recursive shadowcasting** is the standard: correct, fast, and symmetric variants exist.

The choice worth making deliberately is **symmetry**: if A can see B, can B see A? Asymmetric FOV lets a player see an enemy that can't see them, which players read as a bug even when it's a defensible design choice. Symmetric shadowcasting avoids the complaint.

### 11.2 Pathfinding: Dijkstra maps

A* is fine for one actor going one place. For many actors heading toward the same goals, the roguelike-specific technique is better:

```python
def dijkstra_map(level, goals):
    """Distance field from a goal set. Actors roll downhill; fleeing
    actors roll uphill. One computation serves every actor."""
    dist = {tile: INF for tile in level.walkable}
    queue = deque()
    for g in goals:
        dist[g] = 0
        queue.append(g)
    while queue:
        t = queue.popleft()
        for n in level.neighbors(t):
            if dist[n] > dist[t] + 1:
                dist[n] = dist[t] + 1
                queue.append(n)
    return dist
```

Compute one map per goal category — the player, the exit, items — and every actor reads from them. **Fleeing is the same map read backwards** (multiply by a negative coefficient and roll downhill), which is elegant and free.

Recompute only when the goal moves or terrain changes.

### 11.3 Saving

Serialize the full world plus RNG states (§6.4). With the whole simulation in integers and plain data, this is straightforward — another payoff from the no-floats rule.

Version the format and write a migration path. A roguelike run lasts hours; breaking saves on every patch is hostile.

### 11.4 Content data layout

```
data/
  abilities/*.yaml
  items/*.yaml
  monsters/*.yaml
  statuses/*.yaml
  tilesets/*.yaml           # WFC adjacency rules
  room_templates/*.yaml
  generation/*.yaml         # macro parameters per depth
```

**Validate the whole content tree in CI.** Every reference resolved, every ability parsed, every tileset checked for contradiction rate (§4.3). A typo in a monster's ability list should fail the build.

---

## 12. Tech stack and setup

| Layer | Choice | Why |
|---|---|---|
| **Language** | **Rust** or **C#**, or Python for prototyping | See below |
| **Rendering** | Terminal (`bracket-lib`, `libtcod`) or a light 2D engine | Turn-based grids need very little |
| **Data** | YAML or TOML | Human-editable, diffable |
| **Validation** | Schema plus custom checks | §8.3 |
| **Testing** | Native test framework plus the sim harness | §10 |

On language: **Python is excellent for prototyping this** — the generator, the ability system, and the balance harness all iterate fast — and its dict ordering is insertion-stable, which helps determinism. It may be too slow for a ten-thousand-run nightly, though that parallelizes.

**Rust** suits the finished version: integer math, no GC pauses, and `bracket-lib` (from the well-known Rust roguelike tutorial) is purpose-built. **C#** with a light engine is a good middle.

Two setup notes:

**Build the headless harness before the UI.** Everything in §10 depends on the game running without a renderer, and designing for that from the start is much easier than retrofitting it.

**Set up seed-reproducibility tests in week one.** Generate 1,000 levels, hash them, check the hashes in. A determinism break caught the day it's introduced is an hour; caught later it's an archaeology project across every system.

---

## 13. Repository layout

```
Procedural-Dungeon-Roguelike/
├── README.md
├── docs/
│   ├── design.md               ← this document
│   ├── determinism.md          ← ★ the rules, for contributors
│   ├── effect-resolution.md    ← ★ the §9.1 pipeline
│   └── balance-reports/        ← nightly output over time
├── src/
│   ├── rng/
│   │   └── streams.rs          ← ★ hierarchical derivation (§6.2)
│   ├── gen/
│   │   ├── macro.rs            ← ★ graph layout, quest placement (§3.1)
│   │   ├── wfc.rs              ← per-region fill (§4)
│   │   ├── tileset.rs
│   │   ├── validate.rs         ← ★ (§5.1)
│   │   └── repair.rs           ← ★ (§5.2)
│   ├── sim/
│   │   ├── scheduler.rs        ← ★ energy + deterministic tiebreak (§7)
│   │   ├── world.rs
│   │   ├── events.rs
│   │   └── fov.rs
│   ├── abilities/
│   │   ├── registry.rs
│   │   ├── nodes/              ← ★ one file per effect node type (§8.2)
│   │   ├── resolution.rs       ← ★ the pipeline (§9.1)
│   │   └── validate.rs
│   ├── replay/
│   │   ├── record.rs
│   │   └── verify.rs           ← ★ checkpoint hashes (§6.5)
│   ├── harness/
│   │   ├── headless.rs         ← ★ (§10.2)
│   │   └── strategies/
│   └── ui/
├── data/
└── tests/
    ├── determinism/            ← ★ seed → hash fixtures
    ├── generation/             ← ★ 100k-seed validity sweep (§5.3)
    └── content/                ← every ability parses and validates
```

---

## 14. Milestone ladder

### M0 — Determinism contract and architecture ★
**Est. 3–4 days**

Write `docs/determinism.md`: no floats in the sim, hierarchical streams, no unordered iteration, deterministic actor IDs. Decide the two-tier generation split (§3.1). Decide composable nodes versus scripting (§8.2).

**Done when:** the rules are written and the WFC scope is decided.

---

### M1 — RNG and determinism infrastructure ★
**Est. 1 week**

Hierarchical stream derivation, seed/version identity, world state hashing, seed-reproducibility tests.

**Build this first.** Every other system has to be written against it, and retrofitting determinism means auditing everything.

---

### M2 — Turn scheduler and the core loop
**Est. 1 week**

Energy scheduling with deterministic tie-breaking, the command queue, the event system, presentation separation.

**Done when:** a player and several monsters take turns in a hand-built level, and the same input sequence produces identical event logs every time.

---

### M3 — Macro generation and validation ★
**Est. 1.5 weeks**

Room graph, spanning tree connectivity, quest placement, validation, repair, the CI sweep.

**Done when:** 100,000 seeds produce 100,000 valid, connected, quest-consistent levels.

---

### M4 — WFC region fill
**Est. 1.5 weeks**

Simple-tiled WFC, boundary pre-collapse, bounded backtracking, fallback fill, contradiction-rate measurement.

**Done when:** rooms look hand-authored, and the contradiction rate per tileset is a tracked number.

---

### M5 — The ability system ★
**Est. 2 weeks**

Node registry, the composable data format, load-time validation, the resolution pipeline, deferred death, cascade limits.

**Done when:** an invalid ability fails the build, and a chain of reactive abilities terminates cleanly.

---

### M6 — Replay and the balance harness ★
**Est. 1.5 weeks**

Recording, checkpoint hashes, verification, headless runs, strategies, aggregate reporting.

**Before writing content.** You want the measurement apparatus before there's anything to measure, or you'll balance by intuition and find out later.

---

### M7 — Core systems
**Est. 2 weeks**

FOV, Dijkstra maps, monster AI, inventory, saving with RNG state.

---

### M8 — Content
**Est. 6–10 weeks**

Abilities, items, monsters, tilesets, room templates, depth progression — with the nightly balance run informing every iteration.

---

### M9 — UI and polish
**Est. 4 weeks**

Rendering, animation playback, message log, inventory UI, seed entry and sharing.

---

## 15. Reference implementations

### 15.1 WFC with bounded backtracking

```python
def wfc_solve(rng, wave, tileset, backtrack_budget):
    stack = []
    backtracks = 0

    while not wave.fully_collapsed():
        cell = wave.lowest_entropy_cell()          # ties broken by index
        if cell is None:
            break

        options = wave.options(cell)
        if not options:
            if not stack or backtracks >= backtrack_budget:
                return Result(ok=False, reason="contradiction")
            wave, cell, tried = stack.pop()        # undo the last decision
            backtracks += 1
            remaining = wave.options(cell) - tried
            if not remaining:
                continue                           # this branch is exhausted too
            choice = weighted_choice(rng, remaining, tileset.weights)
            stack.append((wave.snapshot(), cell, tried | {choice}))
            wave.collapse(cell, choice)
        else:
            choice = weighted_choice(rng, options, tileset.weights)
            stack.append((wave.snapshot(), cell, {choice}))
            wave.collapse(cell, choice)

        if not wave.propagate(cell):
            continue                               # contradiction — loop handles it

    return Result(ok=True, tiles=wave.result())
```

Two details: entropy ties must break deterministically (by cell index, not by iteration order), and the `tried` set per stack frame is what prevents re-choosing the option that already failed.

Snapshotting the wave is the memory cost of backtracking — which is another reason to scope WFC to small regions.

### 15.2 An effect node

```python
@register_effect("damage")
class DamageEffect(EffectNode):
    amount: int
    damage_type: str
    scaling: dict[str, float] = field(default_factory=dict)

    def validate(self, registry) -> list[str]:
        errs = []
        if self.damage_type not in registry.damage_types:
            errs.append(f"unknown damage type '{self.damage_type}'")
        for stat in self.scaling:
            if stat not in registry.stats:
                errs.append(f"unknown stat '{stat}' in scaling")
        return errs

    def execute(self, ctx: EffectContext) -> None:
        total = self.amount
        for stat, coef in sorted(self.scaling.items()):     # sorted = deterministic
            total += int(ctx.caster.stat(stat) * coef)      # integer math (§6.1)

        total = ctx.apply_modifiers(total, self.damage_type)
        total = ctx.target.apply_resistance(total, self.damage_type)

        apply_damage(ctx.target, max(0, total), ctx)
```

`sorted(self.scaling.items())` looks fussy and isn't — iterating a dict in insertion order works until someone edits the YAML and reorders the keys, at which point integer truncation at each step can produce a different total.

### 15.3 State hashing for replay verification

```python
def hash_world(world) -> int:
    h = FNV_OFFSET
    h = fnv(h, world.turn)
    # Sorted by ID — iteration order must not affect the hash.
    for e in sorted(world.entities, key=lambda e: e.id):
        h = fnv(h, e.id); h = fnv(h, e.x); h = fnv(h, e.y)
        h = fnv(h, e.hp); h = fnv(h, e.energy)
        for s in sorted(e.statuses, key=lambda s: s.id):
            h = fnv(h, s.id); h = fnv(h, s.remaining)
    for name in sorted(world.rng_states):
        h = fnv(h, world.rng_states[name])
    return h
```

Including the RNG states is what makes the hash catch a divergence at the moment it happens rather than several turns later when it first affects an outcome.

---

## 16. Testing

### 16.1 Generation sweeps ★

```python
def test_100k_seeds_valid():
    bad = []
    for seed in range(100_000):
        level, macro = generate(seed)
        if issues := validate(level, macro):
            bad.append((seed, issues))
    assert not bad, f"{len(bad)} invalid; reproduce with seed {bad[0][0]}"

def test_generation_distributions():
    stats = [measure(generate(s)[0]) for s in range(1000)]
    assert 8 <= mean(s.room_count for s in stats) <= 16
    assert stdev(s.path_length for s in stats) > 5      # not all identical
    assert max(s.gen_time_ms for s in stats) < 500      # no pathological seeds
```

The variance assertion matters. A generator that always produces valid, identical levels passes validity and fails at being a generator.

### 16.2 Determinism

```python
def test_same_seed_same_world():
    a = hash_world(Game(12345).generate_all_levels())
    b = hash_world(Game(12345).generate_all_levels())
    assert a == b

def test_replay_reproduces():
    replay = load("tests/replays/deep_run.replay")
    game = Game(replay.seed)
    for turn, action in enumerate(replay.actions):
        game.execute(action)
        if turn in replay.checkpoints:
            assert hash_world(game.world) == replay.checkpoints[turn], \
                f"diverged at turn {turn}"
```

Keep a corpus of recorded runs and replay them in CI. When one diverges, the checkpoint tells you the turn, and determinism means you can step to it.

### 16.3 Content validation

Every ability, item, monster, and tileset parses and validates in CI. Every reference resolves. Contradiction rate per tileset stays under threshold.

### 16.4 Effect resolution

```python
def test_death_during_multi_effect():
    # A three-effect ability whose first effect is lethal must not crash,
    # and the later effects must still resolve coherently.

def test_reactive_cascade_terminates():
    # Two monsters with mutual retaliation must stop at the depth limit.

def test_target_list_snapshot():
    # Chain lightning killing target 1 must still hit targets 2 and 3.
```

These three are the ones that will actually break, and each corresponds directly to a design decision in §9.

### 16.5 Balance regression

Nightly ten-thousand-run sweep, aggregate statistics tracked over time, alerting on large movements in win rate or ability usage.

---

## 17. Stretch goals

| Feature | Effort | Value |
|---|---|---|
| **Daily challenge** | Small | One seed for everyone, with a leaderboard. Replay verification already exists (§6.5). |
| **Seed sharing in-game** | Small | Enter a seed, play that dungeon. Nearly free. |
| **Ghost replays** | Small | Watch another player's run; the replay format already supports it |
| **Modding** | Medium | The data format is already the mod format; you need loading and sandboxing |
| **Graph grammar generation** | Medium | Rule-based quest structure — "replace corridor with lock-key-room" |
| **Hand-authored room template library** | Medium | Often the biggest quality win per hour (§3.3) |
| **Search-based agents for balance** | Medium | Better proxies for skilled play than greedy (§10.2) |
| **Procedural item generation** | Medium | Composable affixes; reuses the effect node model |
| **Deterministic multiplayer** | Large | Determinism makes lockstep possible; see the rollback netcode analysis |

The daily challenge is the best value on that list. Everything it needs — deterministic seeds, replay recording, server-side verification — already exists by M6, and it's one of the strongest retention features a roguelike can have.

---

## 18. References

### Procedural generation

- **Maxim Gumin**, WaveFunctionCollapse — the canonical implementation
- **Paul Merrell**, "Model Synthesis" — the earlier work WFC derives from, and a comparison of the two
- **Karth & Smith**, "WaveFunctionCollapse is Constraint Solving in the Wild" — the clearest framing of what WFC actually is
- **"Evolutionary Wave Function Collapse"** — the maze-versus-Zelda result in §2.1, and why global constraints defeat local patterns
- **Brian Bucklew**, "Tile-based map generation using Wave Function Collapse in *Caves of Qud*" — WFC in a shipped roguelike
- **MarkovJunior** (Gumin) — pattern matching plus constraint propagation; a strong alternative worth evaluating
- **Procedural Content Generation in Games** (Shaker, Togelius, Nelson) — the textbook
- Recent work combining WFC with reinforcement learning and evolutionary search for playability

### Roguelike development

- **RoguelikeDev** community and the r/roguelikedev tutorial series
- **Herbert Wolverson**, *Hands-on Rust* and the Rust Roguelike Tutorial — `bracket-lib`
- **RogueBasin** — the reference wiki; particularly the FOV and pathfinding articles
- **"The Incredible Power of Dijkstra Maps"** (Brogue's author) — §11.2
- **Brogue**, **Caves of Qud**, **DCSS** — design references, and all three have public discussion of their generation

### Systems

- Splitmix64 and PCG — PRNGs suitable for hierarchical derivation
- **Slay the Spire** and **Into the Breach** postmortems — data-driven effects and turn-based determinism in shipped games
- Writing on automated playtesting and simulation-based balance from teams that do it

---

## Appendix A — Decision record

| Decision | Rationale |
|---|---|
| **WFC is a decorator, not an architect** | It cannot guarantee solvability; a roguelike needs global properties (reachable exit, one staircase, key before door) that aren't expressible as tile adjacency |
| Two-tier generation: graph macro, WFC micro | Connectivity comes from a spanning tree by construction; quest constraints are checkable graph properties |
| Prototype hand-authored room templates alongside | Often the largest quality-per-hour win, and it composes with WFC for filler |
| **Scope WFC to rooms, not levels** | Canonical WFC is non-backtracking; a contradiction in a 9×7 room costs a millisecond, one in a 100×100 level costs far more |
| Pre-collapse boundaries and doorways | Enforces macro structure *and* shrinks the search space, reducing contradictions |
| Bounded backtracking plus a guaranteed fallback fill | A boring floor beats a floor that fails to generate |
| Track contradiction rate per tileset in CI | It's a property of the tileset, not the algorithm — a 30% rate means the rules need fixing, not more retries |
| **Validate and repair every level, always** | Even a graph-guaranteed macro tier can be broken by the fill, by a repair, or by a bug |
| Recompute reachability after each carve | One carve can connect several orphans; skipping it carves redundant corridors |
| 100,000-seed CI sweep | Minutes of compute; finds the 1-in-10,000 unplayable level. Determinism makes the failing seed reproducible. |
| Assert on distributions, not just validity | A generator producing identical valid levels passes validity and fails at generating |
| **No floats in the simulation** | Turn-based grid games can avoid them entirely, which removes cross-platform determinism as a problem |
| **Hierarchical RNG streams, derived by path** | One stream means adding a call to level gen shifts every loot drop for every existing seed |
| Seed + generator version as the shareable identity | A seed is only meaningful against a specific generator |
| **RNG state in the save file** | Otherwise reloading rerolls drops — save-scumming should be a decision, not an accident |
| Replay = seed + input log | Gives regression tests, exact bug reports, verified leaderboards, and the balance harness for nearly free |
| Periodic state-hash checkpoints in replays | Without them you know the replay broke; with them you binary-search to the turn |
| **Deterministic tie-break by actor ID in the scheduler** | Without it, order depends on spawn order depends on iteration order, and a distant change silently reorders combat |
| Command queue for input | The queue *is* the input log, so replay and the AI harness both fall out of it |
| Simulation resolves instantly; presentation replays events | If the sim ever waits for an animation, timing has entered the determinism model |
| **Composable typed effect nodes, not scripting in YAML** | The escalation to conditionals and loops is predictable; nodes give expressiveness through composition with static validation and analyzability |
| Validate all content at load and in CI | A typo'd status name should fail the build, not silently no-op in a player's combat |
| Sorted iteration inside effect nodes | Dict order works until someone reorders the YAML, and integer truncation per step then changes the result |
| **Explicit effect resolution pipeline, documented** | "Apply 10 damage" hides ordering questions that otherwise get answered inconsistently per ability |
| Snapshot the target list before applying | Chain lightning killing target 1 must still hit targets 2 and 3 |
| **Deferred death: mark, then process at cleanup** | Immediate removal means later effects operate on a destroyed object |
| Cascade depth limit, logged when hit | Two mutually-reactive abilities loop forever; hitting the limit signals a content bug |
| Events as the seam between sim and everything else | New consumers never touch simulation code |
| **Balance harness before content** | You cannot find a dominant build by playing, because you'd have to think of it first |
| An ability used 0.2% of the time is dead content | The kind of finding only aggregate simulation produces |
| Nightly balance sweep tracked over time | A change moving win rate from 31% to 58% shows up the next morning, not at launch |
| Build headless before the UI | Everything in §10 depends on it, and retrofitting is much harder |
| Symmetric FOV | Asymmetric lets a player see an unseeing enemy, which reads as a bug |
| Dijkstra maps over per-actor A* | One computation serves every actor, and fleeing is the same map read backwards |

---

## Appendix B — Quick reference card

```
WFC — what it can't do
  ★ "cannot guarantee solvability, does not scale to larger levels"
  no concept of: reachability · exactly one exit · key before door
                 boss placement · path length
  maze domain (connectivity is local) → WFC works
  Zelda domain (one player, key, door) → WFC fails
  ★ a roguelike is the Zelda case

ARCHITECTURE
  MACRO  graph + spanning tree ⇒ connectivity BY CONSTRUCTION
         quest placement on the graph (checkable properties)
  MICRO  WFC per ROOM, boundary pre-collapsed at doorways
  ALWAYS validate + repair, then reseed as a last resort

WFC PRACTICE
  canonical WFC is greedy + NON-BACKTRACKING → contradiction = restart
  scope to rooms: a 9×7 failure is 1 ms, a 100×100 failure is not
  pre-collapse boundaries: enforces structure AND shrinks the search
  bounded backtracking + guaranteed fallback fill
  ★ track contradiction rate PER TILESET — 30% means fix the rules
  entropy ties break by cell INDEX, not iteration order

VALIDATION
  flood fill from entrance → exit reachable?
  orphan regions → carve or fill (recompute reachability each carve)
  ★ key reachable WITHOUT passing its own door
  exactly one down-staircase · path length above minimum
  100k-seed sweep in CI · assert distributions, not just validity

DETERMINISM
  ★ turn-based grid ⇒ NO FLOATS ANYWHERE ⇒ cross-platform for free
  ★ hierarchical streams: derive("level/3/wfc/room/7")
    one shared stream ⇒ a new call in level gen shifts every loot drop
  identity = seed + GENERATOR VERSION
  ★ RNG state in the SAVE, or reload = reroll
  killers: dict iteration · object identity · unstable sort · wall clock
           parallelism · unseeded library RNG
  replay = seed + input log + periodic state-hash checkpoints

SCHEDULER
  energy: +speed per tick, act at 100, pay the action's cost
  ★ tie-break by (-energy, actor_id) — id is load-bearing
  actor IDs from a deterministic counter, never object identity
  command queue for input ⇒ the queue IS the replay log
  sim resolves INSTANTLY; presentation replays events over frames

ABILITIES — avoid inventing a language in YAML
  the escalation: number → scaling → conditions → loops → regret
  ★ composable TYPED EFFECT NODES: each type is code, data composes them
    validated at LOAD (unknown type = build failure)
    queryable: "which abilities apply burning?"
  resolution pipeline:
    validate → commit → SNAPSHOT TARGETS → pre-effect → compute
    → apply → post-effect → cascade (DEPTH LIMITED) → cleanup
  ★ deferred death: mark pending, remove at cleanup
  ★ cascade depth limit ~8, and LOG when it fires (content bug)

BALANCE — you cannot hand-test a combinatorial system
  ★ determinism + headless + command queue ⇒ 10,000 runs in under an hour
  strategies: random (floor) · greedy (baseline) · scripted builds · search
  read: win rate by build · death causes · ability usage · damage attribution
  an ability used 0.2% of the time is DEAD CONTENT
  nightly in CI, tracked over time
```
