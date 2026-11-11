# Voxel Chess Official Mods

The mods that come from the makers of [Voxel Chess](https://github.com/dmelin/VoxelChess), and the
guide to writing your own. Copy a mod's folder (`tiny_tank`, `orbs_of_power`) into the game's
`user://mods/` folder to use it.

- `tiny_tank/`: SciFi Chess
- `orbs_of_power/`: Orbs of Power

The game itself ships with no mods. The guide below is kept in step with the game's mod loader
(`scripts/piece_mods.gd`).

# Writing a piece mod

A mod is a single `mod.json` in its own folder. Voxel Chess loads every
`<folder>/mod.json` found in:

- `user://mods/` (your own mods; on Windows `%APPDATA%/Godot/app_userdata/Voxel Chess/mods`)

Turn mods on, off and reorder them from the **MODS** button in the menu. The board behind the menu
reloads at once to show the change. Click a mod's row to select it and see its `preview.png`; the **i** button next to a mod (shown when
it has a `description`) opens its description. Only problems are written under a mod: an invalid
pack's error, or a piece entry that was ignored.

`tiny_tank/` ("SciFi Chess", in this repo) is the complete example: all six pieces and the board.

## Structure

```
{
  "id": "unique_id",          // folder-independent, max 80 chars
  "name": "Shown in the menu",
  "description": "Optional text, max 600 chars, shown with the (i) button next to the mod",
  "pieces": {                 // 1-7 of: pawn knight bishop rook queen king board
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

An invalid pack is shown in the menu with an error and cannot be enabled.

## How mods combine

A piece is the **standard piece with the enabled mods applied to it one after the other**, in the
order of the list in the menu (a mod lower in the list is applied later).

- **Behaviour attributes** (`speed`, `pass_over_occupied_space`, and each key of `capture`) are
  optional. What a mod leaves out is inherited from the mod before it that said something about the
  piece, and if there is none, from the standard piece. A mod can change just one thing:

  ```json
  { "id": "fast_rooks", "name": "Fast rooks",
    "pieces": { "rook": { "speed": 9, "capture": { "sweep": "left_right" } } } }
  ```

- **The design is all or nothing.** `model` and `colors` must come together, and they replace the
  voxels of the piece completely. You cannot recolour three voxels of someone else's piece. A mod
  that gives no design keeps the design it inherits (a mod's, or the standard one).

- **A beam capture needs a beam source** (see below). If, once the mods are put together, a piece
  would capture with a beam but has no `beam_emitter` voxel, that mod's entry for the piece is
  ignored and the mods menu says so under the mod.

## `model`

A piece is built from solid boxes of voxels. Coordinates are integer voxel indices, both corners
inclusive.

- `width`: 1-15. The footprint is `width x width` (x and z, 0 to width-1). A board square is 15
  voxels wide.
- **Axes** (from the piece's own point of view): the piece looks toward low `z`, so `z = 0` is its
  front and `z = width-1` its back. `y` is up (max 39). `x` runs from the piece's **left to its
  right**: `x = 0` is its left flank and `x = width-1` its right. In other words, stand behind the
  piece looking the way it looks and `x` grows left to right; if you face the front of the piece
  instead, you see it mirrored and `x` grows from your right to your left. Keep eyes, visors,
  barrels at low `z`.
- `boxes`: up to 256, each `{"from": [x,y,z], "to": [x,y,z], "material": "name"}`. Later boxes
  overwrite earlier ones. At most 9000 voxels in total. Pieces are hollow inside, only the voxels
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
(slowest) to `9` (fastest). Leave it out to inherit it. The standard pieces:

| Piece | Speed |
|---|---|
| pawn | 7 |
| knight, bishop, queen, king | 5 |
| rook | 3 |

For a flying piece (see below) speed sets how fast it travels in the air; taking off, turning and
landing always take the same time.

## `pass_over_occupied_space`

How the piece gets past pieces standing in its way: `"fly"` or `"flow"`. The standard knight is
`fly`, every other standard piece `flow`.

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
| `crush` | The piece crushes the victim under it (the same landing). On the ground it runs into it like `collide`. |
| `floor_flip` | A trapdoor drops the victim through the floor. |
| `beam` | Every voxel marked `beam_emitter` fires a beam that sweeps across the victim. |

**A `beam` capture needs at least one voxel marked `beam_emitter` in the piece's design.** A pack
that gives both in the same piece without an emitter is invalid; a beam capture that meets a
design without one after the mods are combined is ignored (see above). Glowing voxels alone are
only decoration.

- `"sweep"` sets the direction of the beam as seen from the attacker: `top_bottom` (default),
  `bottom_top`, `left_right`, `right_left`. Emitters keep their layout, so two eyes become two
  parallel beams.
- `"beam_start"` says when the beam goes off:
  - `"end"` (default): the piece moves up to just short of the victim (a knight stops at the
    corner of its L), fires, then steps onto the square.
  - `"start"`: the piece turns towards the victim and fires from where it stands, before it moves.
    Then, if its direction of travel is not the one it faces (a knight's first leg), it turns to
    it, and makes its move. The standard queen uses `start`.
- A flier that has to fly takes off and stays in the air while it does its trick (`floor_flip` and
  `beam` too), and only then flies over the victim's square and lands.

## `board`

The board is part of the piece set. The `board` entry is **one board square, one layer of voxels**:

- `width` must be 15 and every box must be on `y = 0` (one layer).
- `colors.white` is the palette of the light squares, `colors.black` of the dark squares.
- The game repeats the section for all 64 squares, in a chessboard pattern, and builds each square
  from six sheets of it on top of each other: the top sheet as you drew it, the sheets under it
  the same voxels, darker. Only the top sheet glows, the sheets under it use the colour of the
  middle voxel instead of any glowing one.
- There is no border around the board. `speed`, `pass_over_occupied_space` and `capture` mean
  nothing for the board.

The SciFi Chess board is granite: a base material, two kinds of speckle voxels scattered over it,
and a glowing outer ring of voxels in green (white squares) and magenta (black squares).

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
        "width": 15,
        "boxes": [
          {"from": [0, 0, 0], "to": [14, 0, 14], "material": "stone"},
          {"from": [0, 0, 0], "to": [14, 0, 0], "material": "edge", "glow": true},
          {"from": [0, 0, 14], "to": [14, 0, 14], "material": "edge", "glow": true},
          {"from": [0, 0, 1], "to": [0, 0, 13], "material": "edge", "glow": true},
          {"from": [14, 0, 1], "to": [14, 0, 13], "material": "edge", "glow": true}
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
selected; a mod without one gets a blank frame saying "no preview". The one for `tiny_tank` is
drawn from its voxels, on its board squares, by `tools/mod_preview.py` (in the VoxelChess repo) (needs Pillow):

```bash
python3 tools/mod_preview.py ../VoxelChessOfficialMods/tiny_tank
```

## Trying a design

The piece workshop shows the real game meshes of the standard pieces and of the SciFi Chess mod,
each on a board square, and reloads when a `mod.json` or `voxels.gd` is saved:

```bash
GODOT=/Applications/Godot.app/Contents/MacOS/Godot node tools/piece-preview/serve.cjs
```

then open http://localhost:8061. Validate packs headlessly with:

```bash
godot --headless --path . --script tests/mods.gd
godot --headless --path . --script tests/pass_over.gd
```
