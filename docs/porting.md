# Porting BitWorld Games

MeshWorld is an iterative improvement of BitWorld. The porting path should keep
the original philosophy clear: the game owns the rules, state changes happen on
integer ticks, protocol messages stay compact, inputs are replayable, and the
client stays generic.

A port should adapt existing game state into MeshWorld state without moving game
rules into the client.

## Porting Model

```text
Existing BitWorld game logic
  existing tick loop
  existing integer state
  existing input and replay model
        |
        v
MeshWorld adapter
  height chunks
  render nodes
  UI nodes
  text nodes
  shapes
        |
        v
Generic MeshWorld clients
```

## Minimal Port Files

A small game should start with only design and data files.

```text
assets.json
  Image, model, font, and animation ids.

world.json
  Tile size, camera, and layers.

render.nim
  Game-state to MeshWorld-state adapter.
```

The MeshWorld-specific file names can change once the implementation exists.
The BitWorld-compatible manifest file names should stay stable.

## Runtime Compatibility

Ports should keep the Coworld manifest, player, and game settings formats.
This is required so existing Coworld players, policies, launchers, reporters,
graders, and tournament tools can run MeshWorld games.

Keep:

- `coworld_manifest.json`.
- `coplayer_manifest.json`.
- `game.config_schema`.
- `game.results_schema`.
- `variants[].game_config`.
- `certification.game_config`.
- `tokens`, `players`, `slots`, `seed`, `minPlayers`, and `maxGames`.
- `COGAMES_ENGINE_WS_URL` for player and policy containers.
- `COGAME_SAVE_REPLAY_URI`, `COGAME_RESULTS_URI`, `COGAME_CONFIG_URI`,
  `COGAME_LOAD_REPLAY_URI`, `COGAME_HOST`, and `COGAME_PORT` for game
  containers.

MeshWorld-specific asset and render fields should extend the Coworld format,
not replace it.

## Porting Levels

### Level 1: Flat 2D

- Height is zero for every tile.
- Sprites are image quads.
- One orthographic camera.
- Mouse sends screen position and tile position.
- Replays render the same session.
- Each run writes `results.json`.

### Level 2: Tile and Height

- Send integer height chunks.
- Send tile flags and blockers.
- Validate clicks against tiles.
- Add chunk streaming if needed.

### Level 3: Models and Animation

- Replace some image quads with model assets.
- Send animation ids, speeds, scales, and offsets.
- Keep gameplay timing in sim ticks.

### Level 4: Rich Presentation

- Add parent render nodes.
- Add UI layers, text nodes, and shapes.
- Add multiple cameras and spectator views.

### Level 5: Full 3D

- Use richer terrain presentation.
- Use world text, model attachments, and 3D shape tools.
- Add audio and particles after the base port is stable.

## Adapter Duties

The adapter should translate game state to MeshWorld state.

```text
entity position -> render node transform
sprite id -> image asset id
board cell -> height or tile data
selection -> UI or shape
game label -> text node
click target -> game command
```

It should not duplicate rules already owned by the game.

## First Target

The first real port should prove the smallest useful path.

```text
One flat 2D game ported from BitWorld.
One generic WASM client and one generic native client.
Sprites as image quads.
Zero height map.
One camera.
Integer pointer input.
Replay support.
`results.json` output.
```

After that works, add one 3D model and one non-zero height chunk to prove the
upgrade path.

## Client Boundary

The WASM and native clients should be generic. They may:

- Load the game descriptor.
- Fetch assets.
- Render nodes.
- Interpolate transforms.
- Draw UI and text.
- Raycast or pick.
- Send integer input events.

The clients should not:

- Decide game rules.
- Validate attacks.
- Choose pathing results.
- Apply damage.
- Infer collision from render meshes.
- Require per-game code to play normal games.

## Shared Libraries

Ports should use MeshWorld-provided libraries instead of rebuilding common
systems per game.

Useful shared libraries include:

- Aseprite loading for sprite sheets and sprite animations.
- Tiled loading for tile maps and height data.
- glTF loading for models, nodes, bounds, and animation metadata.
- Integer pathfinding over tiles, blockers, and height maps.
- Asset manifest generation.
- Replay and `results.json` writing.
- Policy helpers for parsing state and emitting legal commands.
- Polling and streaming AI adapters for popular model providers.
- Shared AI credential and rate-limit helpers.

Ports should prefer the low-level Treeform stack used by MeshWorld:

- `windy`.
- `silky`.
- `pixie`.
- `vmath`.
- `gltf`.
- `slappy`.
- `fluffy` for profiling and trace timeline inspection.

Ports should emit traces that Fluffy can read. Profiling should cover the game
loop, protocol encode/decode, pathfinding, rendering, policies, replay writing,
and `results.json` writing.

## Debugging

Ports should use shapes early for gameplay and debugging.

Useful debug layers:

- Tile occupancy.
- Height map values.
- Pathfinding results.
- Line of sight.
- Selection rays.
- Clicked triangle and barycentric point.
- Entity anchors.
- Model offsets.
- Parent node trees.

Shape output used only for debugging should be optional and replay-channel
aware.
