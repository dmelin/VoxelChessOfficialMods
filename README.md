# Voxel Chess Official Mods

The mods that come from the makers of Voxel Chess, and the
guide to writing your own. Copy a mod's folder (`tiny_tank`, `orbs_of_power`) into the game's
`user://mods/` folder to use it.

- `tiny_tank/`: SciFi Chess
- `orbs_of_power/`: Orbs of Power
- `core_set/`: Classic Staunton, the pieces and board the game itself is made of. It is already in
  the game; this copy is here to read and to start your own mod from (see "The core set" below).

The game ships with the core set and no mods.

A mod changes how the pieces and the board **look, travel and capture**. It never changes the rules of
chess: which moves are legal is decided by the game, and checked by the game service for online and
saved games, whatever mods are on.

# Writing a piece mod

A mod is a single `mod.json` in its own folder. Voxel Chess loads every
`<folder>/mod.json` found in:

- `user://mods/` (your own mods; on Windows `%APPDATA%/Godot/app_userdata/Voxel Chess/mods`)

Turn mods on and off with the check box in front of each one in the **MODS** screen of the menu, and set
their order with **UP** and **DOWN**; turning a mod on does not move it. The board behind the menu
reloads at once to show a change (a "Loading mod assets" notice shows while it does). Click a mod's name to
select it and see its `preview.png`; the **i** button next to a mod (shown when it has a `description`) opens
its description. Only problems are written under a mod: an invalid pack's error, or a piece entry that was
ignored.

**EDIT** opens a mod in the game's Asset Builder: the voxels of each piece on one side and the mod's JSON on
the other. A mod that does not pass the checks opens too, as plain JSON with the reason it failed
(a syntax error with its line, or which piece and value is wrong), so you can fix it there and save.

`tiny_tank/` ("SciFi Chess", in this repo) is a complete example: all six pieces and the board.
`core_set/` is another, and the one every mod is applied on top of.

## Structure

```
{
  "id": "unique_id",          // folder-independent, max 80 chars
  "name": "Shown in the menu",
  "description": "Optional text, max 600 chars, shown with the (i) button next to the mod",
  "pieces": {                 // optional; up to 7 of: pawn knight bishop rook queen king board
    "<piece>": {
      "speed":   5,                        // optional, 1-9
      "pass_over_occupied_space": "flow",  // optional, "fly" or "flow"
      "capture": {...},                    // optional
      "model":   {...},                    // the voxels ...
      "colors":  {...}                     // ... and their colours: both or neither
    }
  }
}
```

A pack needs at least one piece entry or one clutter type. The optional top-level `clutter` object
adds board decorations; see [Board clutter](#board-clutter).

An invalid pack is shown in the menu with an error and cannot be enabled until it is fixed (**EDIT** opens it).

## How mods combine

A piece is the **core set's piece with the enabled mods applied to it one after the other**, in the
order of the list in the menu (a mod lower in the list is applied later).

- **Behaviour attributes** (`speed`, `pass_over_occupied_space`, and each key of `capture`) are
  optional. What a mod leaves out is inherited from the mod before it that said something about the
  piece, and if there is none, from the core set. A mod can change just one thing:

  ```json
  { "id": "fast_rooks", "name": "Fast rooks",
    "pieces": { "rook": { "speed": 9, "capture": { "sweep": "left_right" } } } }
  ```

- **The design is all or nothing.** `model` and `colors` must come together, and they replace the
  voxels of the piece completely. You cannot recolour three voxels of someone else's piece. A mod
  that gives no design keeps the design it inherits (a mod's, or the core set's).

- **A beam capture needs a beam source** (see below). If, once the mods are put together, a piece
  would capture with a beam but has no `beam_emitter` voxel, that mod's entry for the piece is
  ignored and the mods menu says so under the mod.

## `model`

A piece is built from solid boxes of voxels. Coordinates are integer voxel indices, both corners
inclusive.

- `width`: 1-25. The footprint is `width x width` (x and z, 0 to width-1). A board square is 25
  voxels wide, so a piece narrower than 25 stands smaller on its square, centred.
- **Axes** (from the piece's own point of view): the piece looks toward low `z`, so `z = 0` is its
  front and `z = width-1` its back. `y` is up (max 79). `x` runs from the piece's **left to its
  right**: `x = 0` is its left flank and `x = width-1` its right. In other words, stand behind the
  piece looking the way it looks and `x` grows left to right; if you face the front of the piece
  instead, you see it mirrored and `x` grows from your right to your left. Keep eyes, visors,
  barrels at low `z`.
- `boxes`: up to 4096, each `{"from": [x,y,z], "to": [x,y,z], "material": "name"}`. Later boxes
  overwrite earlier ones. At most 50000 voxels in total. Pieces are hollow inside, only the voxels
  near the surface are drawn.
- Optional per box: `"glow": true` (self-lit voxels, they stay bright in the shade) and
  `"beam_emitter": true` (a beam leaves from here, see capture). At most 32 emitter voxels per
  piece. `"beam_color": [r,g,b]` gives an emitter its own beam colour; without it the beam has the
  colour of the voxel's material for that side.
- A piece does not have to stand on the ground: leave the lowest voxels empty and it floats.

### `colors`

Every material needs an `[r,g,b]` (0-255) for both `white` and `black`, and both sides must list
the same materials (max 64). Voxels get a tiny random tint each, so flat colours do not look flat.

## `speed`

How fast the piece travels when it moves or goes in for a capture: a whole number from `1`
(slowest) to `9` (fastest). Leave it out to inherit it. The core set's pieces:

| Piece | Speed |
|---|---|
| pawn | 7 |
| knight, bishop, queen, king | 5 |
| rook | 3 |

For a flying piece (see below) speed sets how fast it travels in the air; taking off, turning and
landing always take the same time.

## `pass_over_occupied_space`

How the piece gets past pieces standing in its way: `"fly"` or `"flow"`. The core set's knight is
`fly`, every other piece `flow`.

- **`fly`**: the piece only takes off when it has to. If every square it passes over (the
  destination does not count) is free, it stays on the ground and moves normally. A knight has two
  routes, two forward then one sideways or one sideways then two forward; if either is free it
  takes that one (long side first), and it flies only when both are blocked. A flier lifts
  straight up, flies over everything and comes down on its destination. A knight flies its L, any
  other piece flies a straight line. A castling rook always flies, as the king is in its way.
- **`flow`**: the piece stays on the ground. A knight still travels its L and also prefers a free
  route. When every route is blocked it melts on the spot: every voxel sinks to the ground, and
  the piece streams along its path as a layer of mud, each voxel on its own, parting around the
  pieces in its way. It sets off when about half of the voxels have fallen, and starts to
  re-assemble when about half have arrived. If it has to stop in front of its target for a beam
  and the square it stops on is free, it re-forms there before it fires. If that square is
  occupied, it fires as mud: the beams leave from wherever the emitter voxels happen to be, and it
  then flows into the square and re-forms there while the victim crumbles.

## `capture`

Every key is optional and inherited if left out (see "How mods combine").

| `mode` | What happens |
|---|---|
| `collide` | The piece collides with the victim. `"style": "push"` shoves it away, `"impact"` (default) smashes it. A piece that is **in the air** (a `fly` piece with its route blocked) instead **lands on** the victim and crushes it. |
| `floor_flip` | A trapdoor hinged on one side swings up and over and tips the victim away; it topples and crumbles on the board. |
| `trap_door` | The same square, but it drops open **downwards**: the victim tilts with the door and crumbles at once, the rubble falling into the pit before the door swings shut. |
| `beam` | Every voxel marked `beam_emitter` fires a beam that sweeps across the victim. |

**A `beam` capture needs at least one voxel marked `beam_emitter` in the piece's design.** A pack
that gives both in the same piece without an emitter is invalid; a beam capture that meets a
design without one after the mods are combined is ignored (see above). Glowing voxels alone are
only decoration.

- `"sweep"` sets the direction of the beam as seen from the attacker: `top_bottom` (default),
  `bottom_top`, `left_right`, `right_left`, `center`, `center_explode`. Emitters keep their layout,
  so two eyes become two parallel beams. The first four sweep the beam across the victim and cut it
  in two. The last two do not sweep: every emitter fires straight at the middle of the victim.
  - `center`: the beam runs on through and out the far side, then the victim collapses into rubble.
  - `center_explode`: the beam stops at the middle, about halfway through, and the victim bursts
    outwards in a small explosion, its voxels flying in every direction.
- `"beam_start"` says when the beam goes off:
  - `"end"` (default): the piece moves up to just short of the victim (a knight stops at the
    corner of its L), fires, then steps onto the square.
  - `"start"`: the piece turns towards the victim and fires from where it stands, before it moves.
    Then, if its direction of travel is not the one it faces (a knight's first leg), it turns to
    it, and makes its move. The core set's queen uses `start`.
- A flier that has to fly takes off and stays in the air while it does its trick (`floor_flip`, `trap_door` and
  `beam` too), and only then flies over the victim's square and lands.

## `board`

The board is part of the piece set. The `board` entry is **one board square, one layer of voxels**:

- `width` must be 25 and every box must be on `y = 0` (one layer).
- `colors.white` is the palette of the light squares, `colors.black` of the dark squares.
- The game repeats the section for all 64 squares, in a chessboard pattern, and builds each square
  from six sheets of it on top of each other: the top sheet as you drew it, the sheets under it
  the same voxels, darker. Only the top sheet glows, the sheets under it use the colour of the
  middle voxel instead of any glowing one.
- There is no border around the board. `speed`, `pass_over_occupied_space` and `capture` mean
  nothing for the board.

The SciFi Chess board is granite: a base material, two kinds of speckle voxels scattered over it,
and a glowing outer ring of voxels in green (white squares) and magenta (black squares).

## Board clutter

Mods can add grass, stones and other small decorations to every board square with a top-level
`clutter` object, alongside `pieces`. A mod may contain only clutter: omit `pieces` entirely to
decorate an existing set without changing its pieces or board. This requires a game build with
clutter support.

Here is a complete `mod.json` for a small grass decoration:

```json
{
  "id": "little_grass",
  "name": "Little grass",
  "clutter": {
    "grass": {
      "count": { "min": 5, "max": 11 },
      "piece_reaction": "bends",
      "models": [
        {
          "model": {
            "width": 3,
            "boxes": [
              { "from": [1, 0, 1], "to": [1, 3, 1], "material": "leaf" }
            ]
          },
          "colors": {
            "white": { "leaf": [91, 135, 52] },
            "black": { "leaf": [91, 135, 52] }
          }
        }
      ]
    }
  }
}
```

Each key under `clutter` names a decoration type. Each type has:

- **`count`**: a whole number from 0 to 32, or an inclusive range such as
  `{"min": 5, "max": 11}`. The default is 1. Counts are chosen separately for each square;
  a minimum of 0 allows empty squares.
- **`models`**: 1–16 variants, each containing `model` and `colors` in the same format as pieces.
  Each spawn chooses a variant with equal probability. One variant is enough.
- **`piece_reaction`**: one of the following values.

| Reaction | Behavior |
|---|---|
| `ignored` (default) | Static decoration with no collision. |
| `bends` | Ground-level voxels stay planted; higher voxels bend away from nearby pieces and spring upright after they pass. Pieces flying above the decoration do not affect it. Uses batched meshes and a vertex shader, rather than individual physics bodies. |
| `push` | Each decoration is an intact loose rigid body that pieces and rubble can shove. It sleeps naturally; objects falling below the board are hidden and frozen. |

### Placement, colors and size

Decorations are scattered across all 64 squares, including occupied ones. Positions keep the
rotated footprint inside its square. Rotation is chosen from 16 orientations around the vertical
axis, in 22.5-degree steps.

Placement uses a deterministic seed per square and decoration type. Rebuilding the same mods or
resuming a game reproduces the layout and variant choices; it does not save the displaced positions
of loose `push` objects as chess state.

Coordinates are integer voxel indices, with inclusive box corners, just like piece models.
One voxel is 1/25 of a square. Width is 1–25; clutter is limited to 16 voxels high.
Supply both palettes as usual, but clutter uses the **`white` palette on every square**.
Using identical white and black palettes is a simple choice for neutral scenery.

The game's **Piece size** setting also scales clutter, around each object's own planted position.
It preserves the placement, rotations, counts and variant choices. Loose objects keep their current
positions when resized, and their collision shapes scale too.

### Limits and combining mods

- Up to **8 named clutter types per pack**.
- **1–16 model variants per type**, each with at most **512 voxels** and **16 voxels in height**.
- Counts range from **0–32**; the sum of all types' maximum counts in one pack cannot exceed
  **32 objects per square**.
- Enabled packs merge clutter by name in menu order. A later definition replaces the **whole
  earlier definition** with the same name. Different names coexist.
- Across all enabled packs, the renderer allocates at most **32 objects per square** in merged
  type order. Types reached after the budget is exhausted do not spawn.

Clutter is cosmetic: it does not change legal moves, piece selection or saved chess moves.
It rebuilds when changed mods are applied. Start with modest counts for `push` decorations,
since each spawned object adds a physics body.

### Editing

Add or edit `clutter` in the Asset Builder's JSON editor, then save and enable the mod.
JSON edits preserve clutter; there is no dedicated visual clutter modeling panel yet.
A clutter-only pack installs and enables in the same way as a piece mod.

## The core set

The game's own pieces and board are a pack like any other: **Classic Staunton**, in
[`core_set/mod.json`](core_set/mod.json). Ivory and ebony pieces, shaped like a tournament set, each
standing on the full 25 x 25 voxels of its square on a plain light and dark wood board. Every mod is
applied on top of it, so whatever your mod leaves out is what is described here. To start your own
mod, copy `core_set/` under a new name, change its `id` and `name`, and edit.

| Piece | Footprint | Height (voxels) | Look |
|---|---|---|---|
| pawn | 25 x 25 | 28 | round base, trim ring, a slim stem and a round head |
| rook | 25 x 25 | 38 | broad base, tall tower, a flared top with eight battlements |
| bishop | 25 x 25 | 48 | slim body and collar, a pointed mitre with a slit down its front |
| knight | 25 x 25 | 48 | a horse head facing forward, ears, eyes and a trim mane |
| queen | 25 x 25 | 53 | tall, ringed body and a crown of points |
| king | 25 x 25 | 62 | like the queen, taller, with a cross on top |
| board | 25 x 25 | 1 (one layer) | pale wood squares with a little grain |

Every piece is built the same way, bottom to top: a base of three overlapping boxes (two long
rectangles crossed, with a smaller one between), a ring of trim, then a body that gets slimmer the
higher it goes. Each stack of rings is drawn as **three boxes that overlap** (one wide in `x`, one
wide in `z`, one square in the middle); together they make a rounded cross-section out of plain
boxes, which is how a voxel piece gets to look round. Look at the pawn below: the base is lines 1-3,
the trim ring 4-6, the body 7-9, and so on up.

**Materials.** `wood` is the body, `trim` the rings and details; the knight also has `ink` for its
eyes and nostrils (dark on the white side, light on the black side, so they read on both).

| Material | White | Black |
|---|---|---|
| `wood` | `[240, 232, 212]` ivory | `[46, 42, 44]` ebony |
| `trim` | `[205, 186, 146]` | `[92, 80, 72]` |
| `ink` (knight) | `[40, 34, 32]` | `[215, 205, 185]` |

The board's materials are `sq` (the square: `[240, 217, 181]` light, `[181, 136, 99]` dark) and
`grain` (a slightly different shade for the six short lines of grain on each square).

**Behaviour.** The core set's pieces travel and capture as below (how fast a piece glides, whether it flies
or flows over the pieces in its way, and the effect of a capture; the moves themselves are always chess),
and a mod inherits all of it:

| Piece | `speed` | `pass_over_occupied_space` | `capture` |
|---|---|---|---|
| pawn | 7 | `flow` | `{"mode": "collide"}` |
| knight | 5 | `fly` | `{"mode": "collide"}` |
| bishop | 5 | `flow` | `{"mode": "beam", "sweep": "top_bottom"}` |
| rook | 3 | `flow` | `{"mode": "collide", "style": "push"}` |
| queen | 5 | `flow` | `{"mode": "beam", "sweep": "top_bottom", "beam_start": "start"}` |
| king | 5 | `flow` | `{"mode": "floor_flip"}` |

**Beam emitters.** Every core piece has at least one `beam_emitter` voxel, so a mod can give any of
them a beam capture without drawing a new design. They are single voxels, in the same material as
what surrounds them, so they are not visible as such. Positions are `[x, y, z]`:

| Piece | Emitters |
|---|---|
| pawn | `[7, 13, 3]`, on the front of the round head |
| rook | `[7, 9, 3]`, on the front of the tower |
| bishop | `[7, 17, 2]`, in the slit of the mitre |
| knight | `[7, 23, 2]`, on the front of the head |
| queen | eight, at the tops of the crown's points: `y = 30`, all around |
| king | `[7, 21, 2]`, on the front of the upper body |

So a mod that only says `"pawn": {"capture": {"mode": "beam"}}` makes the standard pawn fire a
beam from its head.

### The pawn, in full

A simple pawn on a 15-wide footprint (so it stands smaller than the core pawn, which is 25 wide with bevelled edges and too long to list here). A mod's `model` has the same shape. Boxes are listed from the
bottom up, three boxes per level (the overlapping cross-section described above).

```json
"pawn": {
  "model": {
    "width": 15,
    "boxes": [
      {"from": [1, 0, 4],  "to": [13, 1, 10], "material": "wood"},
      {"from": [4, 0, 1],  "to": [10, 1, 13], "material": "wood"},
      {"from": [3, 0, 3],  "to": [11, 1, 11], "material": "wood"},

      {"from": [2, 2, 5],  "to": [12, 2, 9],  "material": "trim"},
      {"from": [5, 2, 2],  "to": [9, 2, 12],  "material": "trim"},
      {"from": [4, 2, 4],  "to": [10, 2, 10], "material": "trim"},

      {"from": [3, 3, 5],  "to": [11, 5, 9],  "material": "wood"},
      {"from": [5, 3, 3],  "to": [9, 5, 11],  "material": "wood"},
      {"from": [4, 3, 4],  "to": [10, 5, 10], "material": "wood"},

      {"from": [4, 6, 6],  "to": [10, 9, 8],  "material": "wood"},
      {"from": [6, 6, 4],  "to": [8, 9, 10],  "material": "wood"},
      {"from": [5, 6, 5],  "to": [9, 9, 9],   "material": "wood"},

      {"from": [3, 10, 5], "to": [11, 10, 9], "material": "trim"},
      {"from": [5, 10, 3], "to": [9, 10, 11], "material": "trim"},
      {"from": [4, 10, 4], "to": [10, 10, 10],"material": "trim"},

      {"from": [4, 11, 6], "to": [10, 11, 8], "material": "wood"},
      {"from": [6, 11, 4], "to": [8, 11, 10], "material": "wood"},
      {"from": [5, 11, 5], "to": [9, 11, 9],  "material": "wood"},

      {"from": [3, 12, 5], "to": [11, 14, 9], "material": "wood"},
      {"from": [5, 12, 3], "to": [9, 14, 11], "material": "wood"},
      {"from": [4, 12, 4], "to": [10, 14, 10],"material": "wood"},

      {"from": [4, 15, 6], "to": [10, 15, 8], "material": "wood"},
      {"from": [6, 15, 4], "to": [8, 15, 10], "material": "wood"},
      {"from": [5, 15, 5], "to": [9, 15, 9],  "material": "wood"},

      {"from": [5, 16, 6], "to": [9, 16, 8],  "material": "wood"},
      {"from": [6, 16, 5], "to": [8, 16, 9],  "material": "wood"},
      {"from": [6, 16, 6], "to": [8, 16, 8],  "material": "wood"},

      {"from": [7, 13, 3], "to": [7, 13, 3],  "material": "wood", "beam_emitter": true}
    ]
  },
  "colors": {
    "white": {"trim": [205, 186, 146], "wood": [240, 232, 212]},
    "black": {"trim": [92, 80, 72],    "wood": [46, 42, 44]}
  }
}
```

(Blank lines are only for reading; JSON allows them between items.)

### A mod built on the core set

A mod does not have to redraw anything to change how the core set plays. This one keeps every
piece of Classic Staunton and only makes the knights and rooks fire beams and the pawns slow:

```json
{
  "id": "beam_knights",
  "name": "Beam knights",
  "pieces": {
    "knight": { "capture": { "mode": "beam", "beam_start": "start" } },
    "rook":   { "capture": { "mode": "beam", "sweep": "left_right" }, "speed": 5 },
    "pawn":   { "speed": 3 }
  }
}
```

It works because the core knight and rook have `beam_emitter` voxels (see the table above).

## A small example

A rook that is only faster and flies, a pawn with a complete design and a beam, and a plain board:

```json
{
  "id": "example",
  "name": "Example",
  "pieces": {
    "rook": { "speed": 9, "pass_over_occupied_space": "fly" },
    "pawn": {
      "speed": 6,
      "capture": {"mode": "beam", "sweep": "top_bottom", "beam_start": "start"},
      "model": {
        "width": 5,
        "boxes": [
          {"from": [1, 0, 1], "to": [3, 2, 3], "material": "body"},
          {"from": [2, 3, 2], "to": [2, 4, 2], "material": "body"},
          {"from": [2, 2, 0], "to": [2, 2, 0], "material": "lens", "glow": true, "beam_emitter": true}
        ]
      },
      "colors": {
        "white": {"body": [220, 220, 225], "lens": [60, 255, 120]},
        "black": {"body": [40, 50, 80],    "lens": [255, 50, 220]}
      }
    },
    "board": {
      "model": {
        "width": 25,
        "boxes": [
          {"from": [0, 0, 0], "to": [24, 0, 24], "material": "stone"},
          {"from": [0, 0, 0], "to": [24, 0, 0], "material": "edge", "glow": true},
          {"from": [0, 0, 24], "to": [24, 0, 24], "material": "edge", "glow": true},
          {"from": [0, 0, 1], "to": [0, 0, 23], "material": "edge", "glow": true},
          {"from": [24, 0, 1], "to": [24, 0, 23], "material": "edge", "glow": true}
        ]
      },
      "colors": {
        "white": {"stone": [214, 214, 218], "edge": [60, 255, 120]},
        "black": {"stone": [50, 52, 58],    "edge": [255, 50, 220]}
      }
    }
  }
}
```

(The comments in the structure sketch above are for explanation only; real `mod.json` files must
be plain JSON.)

## Preview

Put a `preview.png` (500x500 works well) next to `mod.json`. The mods menu shows it when the mod is
selected; a mod without one gets a blank frame saying "no preview". The previews in this repo show
the mod's pieces as isometric voxel sculptures on its board squares.

## Trying a design

Copy your mod's folder into `user://mods/`, open **MODS** in the game's menu and switch it on. The
board behind the menu reloads at once. After you change a `mod.json`, switch the mod off and on
again (or restart the game) to load the new file. A mod with a mistake in it is listed with the
error and cannot be switched on.
