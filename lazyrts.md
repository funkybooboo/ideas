# lazyrts

> Real-time strategy. In a terminal. With ~5% of the buttons.

A deliberately tiny AOE2-flavored RTS, written in Zig, rendered as glyphs in your terminal. The "lazy" in the name is the design principle: every feature has to earn its place, and the default answer is *cut it*.

## Core loop

The same loop AOE2 has, just trimmed:

```
gather  ->  build  ->  train  ->  fight  ->  win
```

Win condition: destroy the enemy Town Center.

## Scope

| Thing       | AOE2                          | lazyrts                          |
|-------------|-------------------------------|----------------------------------|
| Resources   | food, wood, gold, stone       | **food, wood**                   |
| Buildings   | dozens                        | **TC, House, Barracks, Farm**    |
| Units       | dozens, with upgrades         | **Worker, Soldier**              |
| Ages        | Dark / Feudal / Castle / Imp  | none                             |
| Tech tree   | huge                          | none                             |
| Map         | random, big                   | fixed seed, ~80x40               |
| Players     | up to 8                       | 1v1, you vs scripted AI          |
| Fog of war  | yes                           | not in v1                        |
| Multiplayer | yes                           | no                               |

If you want a feature that isn't in the right column, the answer is *probably not*. Add only if it directly serves the core loop.

## Tech stack

- **Language:** Zig.
- **TUI library:** [vaxis](https://github.com/rockorager/libvaxis) -- handles raw mode, input events, double-buffered render. Avoids a week of termios yak-shaving.
- **No external runtime deps** beyond vaxis.

## Rendering

One glyph per tile.

| Glyph | Thing            |
|-------|------------------|
| ` `   | grass            |
| `T`   | tree             |
| `~`   | water            |
| `C`   | Town Center      |
| `H`   | House            |
| `B`   | Barracks         |
| `F`   | Farm             |
| `w`   | Worker           |
| `s`   | Soldier          |

Color via ANSI: own = blue, enemy = red, resources = green/yellow, terrain = dim.

## Sim model

Fixed-tick simulation, decoupled render.

- **Logic tick:** 10 Hz. Deterministic. Drives movement, gathering, combat, AI.
- **Render:** redraw on dirty, capped at 30 Hz.
- **Threads:** none. Single-threaded event loop via vaxis.

This is the standard RTS architecture for a reason -- it makes replay, AI scripting, and debugging tractable.

## Modules

```
src/
  main.zig         entry + event loop
  game.zig         GameState, tick()
  map.zig          tile grid, terrain gen, resource placement
  entity.zig       Worker / Soldier / Building, owner, hp, state
  command.zig      order queue: move, gather, build, attack
  pathfinding.zig  A* on tile grid
  combat.zig       adjacency damage per tick
  economy.zig      resource counters, drop-off, pop cap
  ai.zig           scripted opponent
  render.zig       vaxis buffer writes
  input.zig        cursor, selection, command dispatch
build.zig
build.zig.zon      vaxis dep
```

## UI layout

```
+--------------------------------------+-----------+
|                                      | Resources |
|           map viewport               |  food: 50 |
|        (scrolls with cursor)         |  wood: 30 |
|                                      |  pop: 4/10|
|                                      +-----------+
|                                      |  Selected |
|                                      |  Worker   |
|                                      |  HP 25/25 |
|                                      +-----------+
|                                      |  Minimap  |
+--------------------------------------+-----------+
status bar: hints / messages
```

## Controls

| Key            | Action                                   |
|----------------|------------------------------------------|
| Arrows / hjkl  | Move cursor                              |
| Space          | Select entity under cursor               |
| M              | Move selected to cursor                  |
| A              | Attack-move to cursor                    |
| G              | Gather (worker -> tree/farm under cursor)|
| B              | Build menu (then key per building)       |
| T              | Train (TC: worker; Barracks: soldier)    |
| Tab            | Cycle own units                          |
| Q              | Quit                                     |

## Milestones

Each milestone ships a runnable game.

1. **Empty world.** Map renders, cursor moves, Q quits.
2. **One worker.** Place TC, spawn worker, walk to commanded tile via A*.
3. **Economy.** Trees on map -> worker chops -> drops at TC -> wood counter increments.
4. **Buildings.** Build House (pop cap) and Farm (food).
5. **Military.** Build Barracks, train Soldier.
6. **Combat.** Two armies fight (manual control of both sides).
7. **AI.** Scripted opponent with a fixed build order.
8. **Win/lose.** Detect TC destruction, show result, restart prompt.

## Verification

- Each milestone runs end-to-end manually before moving on. If it isn't fun-ish to play at that milestone, fix that before adding the next layer.
- Unit tests on pure logic only: pathfinding correctness, resource arithmetic, combat math, AI state transitions.
- No automated render tests. Visual inspection is fine for a terminal app.

## Risks and tradeoffs

- **Pathfinding cost.** A* per unit per repath is fine at 80x40 with tens of units. Switch to flow fields only if perf actually bites.
- **Input latency.** Depends on terminal emulator; vaxis raw mode should keep it under ~50 ms.
- **AI brittleness.** Scripted AI is dumb but predictable, which is correct for "simple". Resist behavior trees and utility AI.
- **Real-time feel.** 10 Hz tick is the sweet spot. If it feels sluggish, raise to 15 Hz before changing architecture.

## Non-goals

Explicit list of things lazyrts will *not* do, so future-me does not relitigate them:

- Multiplayer / networking.
- Save / load.
- Custom maps or map editor.
- Tech trees, civilizations, hero units.
- Mouse support.
- Sound.
- Animation beyond glyph state changes.

## Name

Short. The "lazy" is honest -- the design wins by aggressive subtraction, not feature count.
