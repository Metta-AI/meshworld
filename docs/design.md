# MeshWorld Design

MeshWorld is an iterative improvement of BitWorld. It keeps the parts that
worked well: integer simulation, compact protocol messages, lockstep replays,
and simple clients. It extends that model with a generic 2D and 3D renderer
that loads client-side assets by id.

## Philosophy

MeshWorld treats the server simulation as the source of truth. The sim sends
small integer facts: positions, tile heights, render ids, animation ids, input
results, and state changes. The client turns those facts into smooth visuals,
but it does not decide gameplay.

The design favors boring deterministic data over engine magic. A game should be
able to replay from a seed, an initial state, and an input stream. Visual assets
can be large and rich, but the protocol should stay small and stable.

The client is generic. A game describes its world, UI, camera, text, assets, and
input targets through data. That lets WASM and native clients play many games
without custom game logic compiled into the client.

MeshWorld should also provide a broad shared library stack for games and
policies. Games should not need to reimplement common loading, pathfinding,
asset conversion, replay, result writing, or protocol helpers.

## Integer Determinism

MeshWorld uses integer simulation and integer protocol values so lockstep can
work on different hardware, operating systems, browsers, and native builds.
Floating-point math can produce tiny differences between CPUs, GPUs, compilers,
runtime libraries, and optimization settings. Those differences are small for
rendering, but they can compound into different simulation results.

Lockstep and replay depend on each client reaching the same answer from the same
seed, initial state, and input stream. Integer math makes that contract much
easier to test, hash, replay, and debug. Floats may be used inside the client
for rendering and interpolation, but they should not become gameplay truth or
protocol state.

Simulation code must avoid platform-dependent behavior. It should not depend on
hash table iteration order, pointer addresses, pointer hashes, randomized hash
seeds, nondeterministic random number generators, Unix time, wall-clock time, or
thread scheduling. Any randomness used by the sim must come from an explicit
seed and a deterministic RNG. Any ordering that affects sim state must be
explicitly sorted or stored in stable arrays.

Simulation code should also avoid runtime allocation during ticks. Games should
favor fixed-size containers with explicit capacities, such as 16 players, 100
bullets, 256 particles that affect gameplay, or 4096 active tiles. Capacity
limits should be part of game config or constants, not emergent side effects of
dynamic containers. The tick loop should reuse preallocated arrays, free lists,
ring buffers, and stable integer handles.

Allocation is acceptable during loading, setup, replay decoding, asset
preparation, and tooling. It should not be required for ordinary simulation
steps such as movement, combat, spawning, pathing, collision, or scoring.

AI chat strings are the main exception. Policies and games may allocate strings
for chat prompts, model responses, player messages, and transcript logging.
Chat is input. Chat strings must be bounded, recorded in replays, included in
input hashes, and kept separate from hot numeric sim loops where possible.

## Goals

- Run many games with generic WASM and native clients.
- Keep all gameplay truth in the server simulation.
- Keep protocol values integer-only.
- Support 2D games, 3D games, UI, text, shapes, and visualization.
- Make existing BitWorld games cheap to port before adding rich 3D features.
- Preserve deterministic replay through versioned state and hashes.
- Write a `results.json` file for every completed game run.
- Provide shared libraries that games and policies can use directly.

## Non-Goals

- Do not put game rules in the client.
- Do not send render meshes or images through the game protocol.
- Do not infer gameplay collision from visual meshes.
- Do not require custom client code for each game.

## Core Rules

- The sim owns gameplay truth.
- The client owns rendering, interpolation, loading, and local picking.
- Meshes, images, fonts, sounds, and effects are assets by id.
- Sim geometry, height maps, tile flags, and blockers are protocol data.
- Client interpolation is visual only.
- Replays must not depend on mutable bucket contents.
- Replay recording is required.
- `results.json` output is required.
- A MeshWorld game must fit the Coworld spec, not define a parallel runtime
  contract.
- Sim state must not depend on platform-specific hash, pointer, random, time,
  or thread behavior.
- Sim ticks should use fixed-capacity storage and avoid memory allocation.
- AI chat strings are the main allocation exception and must be bounded and
  always replay-recorded and input-hashed.

## Shared Libraries

MeshWorld is a protocol and client family, but it is also a library stack for
game authors and policies.

Shared libraries should cover common game-development needs:

- Aseprite sprite and animation loading.
- Tiled map loading.
- glTF loading and asset inspection.
- Integer pathfinding.
- Height map and tile helpers.
- Shape, text, sound, particle, and UI builders.
- Asset manifest generation and validation.
- Replay recording and replay playback.
- `results.json` writing and validation.
- Protocol encoding, decoding, and hashing.
- Policy helpers for reading game state and emitting valid commands.
- Polling and streaming AI adapters for popular model providers.
- Shared AI credential and rate-limit helpers for policy containers.

The same helpers should be usable by game servers, local tools, policies,
tournament runners, and tests. The goal is to make the correct deterministic
path easy, not to leave every game with its own ad hoc asset and replay code.

AI helpers should make it easy for policies to connect to common hosted models
without each policy reimplementing provider-specific polling, retries, response
parsing, tool calls, or rate-limit handling. Provider credentials should come
from environment variables supplied by the Coworld runtime.

## Low-Level Stack

MeshWorld should build on the existing Treeform Nim library stack where
possible. These libraries are the preferred low-level foundation:

- `windy` for windows, native input, and platform events.
- `silky` for cross-platform rendering surfaces and client support.
- `pixie` for 2D image processing, raster work, and text/image helpers.
- `vmath` for vector and matrix math.
- `gltf` for glTF parsing, models, materials, nodes, and animations.
- `slappy` for low-level audio.
- `fluffy` as the primary profiler and trace timeline viewer.

The WASM and native clients should share as much code as possible above this
layer. Platform differences should be isolated behind small compile-time
backends, not duplicated game logic.

Profiling should be part of the normal development loop. MeshWorld games,
clients, libraries, and policies should emit traces that Fluffy can inspect,
so performance work starts from concrete timelines instead of guesswork.

## Coworld Spec Compatibility

MeshWorld games should fit the Coworld spec directly. A MeshWorld game should
be a valid Coworld game with MeshWorld asset, render, and replay fields added
where the Coworld manifest and config allow them. It should not require a
parallel manifest format, launcher contract, player contract, or tournament
contract.

Game manifests and player manifests should use the Coworld shape:

```text
coworld_manifest.json
  game
  player
  commissioner
  reporter
  grader
  variants
  certification

coplayer_manifest.json
  name
  image
  run
  env
  games
```

Game settings should keep the same JSON-schema-driven format:

```text
game.config_schema
game.results_schema
variants[].game_config
certification.game_config
```

Fixed roster settings should keep the same field names and meanings:

```text
tokens
players
slots
seed
minPlayers
maxGames
closedRoster
```

The runtime environment variables should stay compatible:

```text
COGAME_SAVE_REPLAY_URI
COGAME_RESULTS_URI
COGAME_CONFIG_URI
COGAME_LOAD_REPLAY_URI
COGAME_HOST
COGAME_PORT
COGAMES_ENGINE_WS_URL
```

Player websocket URLs should keep the existing query shape:

```text
ws://host:port/player?name=<name>&slot=<slot>&token=<token>
```

MeshWorld may add fields for assets, cameras, render state, and 3D features.
Those additions must be schema-friendly Coworld extensions and must not break
existing Coworld launchers, players, policies, reporters, graders, or
tournament tooling.

## Units

The default tile size is 256 sim units.

```text
TileBits = 8
TileSize = 256
TileMask = 255
```

Positions use integer world units:

```text
tileX = worldX div 256
tileY = worldY div 256
localX = worldX mod 256
localY = worldY mod 256
```

Height uses the same unit scale:

```text
height = 0      # ground
height = 128    # half tile
height = 256    # one tile
height = -256   # one tile below
```

## Simulation Geometry

The sim sends integer height and tile data. This data is authoritative for
movement, pathing, line of sight, placement, collision, and replay.

```text
HeightChunk
  chunkX
  chunkY
  layer
  width
  height
  tileSize
  baseHeight
  heights
  flags
  hash
  applyTick
```

Dynamic terrain changes should be sent as deltas where possible:

```text
HeightDelta
  tick
  chunkId
  tileIndex
  oldHeight
  newHeight
  oldFlags
  newFlags
```

## Assets

Assets live outside the protocol in a versioned manifest. WASM clients should
fetch assets through HTTPS URLs backed by S3 or a CDN. Native clients may use
the same URLs, a local cache, or packaged assets, but the manifest hash remains
the source of truth.

During local game development, the same websocket or HTTP game server may serve
the asset manifest and asset files. This keeps iteration simple. For deployed
games, assets should move to immutable S3 or CDN-backed URLs so the game server
only handles protocol traffic.

```text
AssetManifest
  manifestId
  hash
  models
  images
  cursors
  fonts
  sounds
  effects
```

The server sends asset ids, never asset bytes. Every replay records the asset
manifest hash used by that session.

## Render Nodes

Everything visible is a render node. Meshes, images, UI elements, text, and
shapes all use the same parent-child transform model.

```text
RenderNode
  nodeId
  parentId
  layerId
  kind
  assetId
  x
  y
  z
  rotation
  scaleX
  scaleY
  scaleZ
  offsetX
  offsetY
  offsetZ
  hitTest
  hoverCursorId
  visible
```

`parentId = 0` means no parent. Parent links allow weapons, labels, health
bars, bounds, effects, and UI children to move with another node.

Cycles are invalid.

## Entity Anchors and Offsets

The sim location is the gameplay anchor. Model offsets are visual alignment.

```text
Entity
  entityId
  simX
  simY
  simZ
  rotation
  rootRenderNodeId
```

Imported models may have bad origins, wrong facing, or different native scale.
Those issues should be fixed with render offsets, rotation offsets, and scale,
not by changing sim truth.

## Animation

The sim sends animation state, speed, and start tick.

```text
AnimationState
  nodeId
  animationId
  mode
  startTick
  speed
```

Animation speed is fixed point:

```text
1000 = 1.0x
500 = 0.5x
2000 = 2.0x
```

Gameplay timing should still be explicit in sim ticks. Animation playback does
not decide damage, movement, targeting, or cooldowns.

## 2D Support

A 2D image is a mesh in its native size. The client can generate a quad mesh
for each image asset. Images may also define sprite frames and sprite
animations for atlas-based 2D animation.

```text
RenderMode
  Mesh3d
  SpriteWorld
  SpriteBillboard
  SpriteScreen
```

This lets 2D games use the same render node, layer, parent, animation, and
input systems as 3D games.

Sprite animations use the same server-owned animation state as model
animations. For an image node, `animationId` selects a sprite animation instead
of a glTF clip. The client chooses the visible frame from `startTick`, `speed`,
and the image asset's frame data.

```text
SpriteFrame
  frameId
  x
  y
  width
  height
  offsetX
  offsetY

SpriteAnimation
  animationId
  frames
  frameTicks
  loop
```

## UI

UI uses layers and screen-space render nodes.

```text
UiNode
  nodeId
  parentId
  layerId
  assetId
  x
  y
  width
  height
  anchor
  pivot
  clip
  hitTest
  hoverCursorId
  zOrder
```

UI can send input events back to the server, but the server decides what they
mean.

## Text

Text is a first-class node type.

```text
TextNode
  nodeId
  text
  fontId
  size
  color
  align
  valign
  maxWidth
  maxHeight
  wrap
  mode
```

Text modes:

```text
ScreenText
WorldBillboardText
WorldFlatText
World3dText
```

Fonts are assets. Text layout is visual unless the server also defines hit
boxes or UI targets.

## Sounds

Sounds are first-class scene state. Sound files are assets, while playback is
protocol state controlled by the server.

```text
SoundNode
  soundNodeId
  parentId
  soundId
  mode
  active
  loop
  startTick
  stopTick
  volume
  pitch
  x
  y
  z
  innerRadius
  outerRadius
```

Sound modes:

```text
WorldSound
ScreenSound
CameraSound
```

World sounds use integer world coordinates and can be parented to moving game
objects. Screen sounds are for UI. Camera sounds move with a view, such as a
first-person sound or picture-in-picture overlay.

Volume and pitch use fixed point:

```text
1000 = 1.0
0 = silent
```

The server decides when a sound starts, stops, loops, and follows a parent. The
client may decode, stream, mix, spatialize, and fade locally, but sound playback
state should be replayable from ticks and ids.

## Particles

Particles are server-controlled visual effects. Effect definitions are assets,
while emitters are protocol state. Particle effects may use static images or
sprite animations from image assets.

```text
ParticleEmitter
  emitterId
  parentId
  effectId
  active
  loop
  startTick
  stopTick
  seed
  x
  y
  z
  rotation
  scale
  color
  rate
  speed
  animationId
```

The server decides which effect plays, where it plays, when it starts, whether
it loops, which sprite animation it uses, and when it stops. The client may
simulate and draw individual particles locally from the emitter state, asset
definition, and seed.

Particles are visual unless the server gives them gameplay meaning through
separate sim state. For example, a fire area should have explicit tile or shape
state for gameplay, even if a particle emitter makes it look like fire.

## Cameras

Games define cameras through protocol state.

```text
Camera
  cameraId
  active
  mode
  screenX
  screenY
  screenWidth
  screenHeight
  x
  y
  z
  rotation
  zoom
  fov
  targetNodeId
  layerMask
```

The same client should support top-down 2D, isometric, perspective 3D, replay,
spectator cameras, minimaps, and picture-in-picture views. Multiple cameras may
be active at the same time. Each active camera renders into its own screen rect.

## Input and Picking

Mouse and pointer input sends integers only.

```text
PointerEvent
  tick
  pointerId
  action
  button
  screenX
  screenY
  hitKind
  targetId
  triangleId
  worldX
  worldY
  worldZ
  baryU
  baryV
  baryW
```

Triangle barycentric values use integers where:

```text
baryU + baryV + baryW = 65535
```

The client may report exact mesh hits, but gameplay should validate against
sim state such as tiles, blockers, visibility, and range.

## Cursors

Cursors are client-side pointer visuals controlled by server data. Cursor
images are assets with explicit hot spots.

```text
CursorAsset
  cursorId
  imageId
  hotspotX
  hotspotY
```

Hit-testable nodes may define a hover cursor.

```text
hoverCursorId = 0  # Use the current default cursor.
```

When the pointer hovers a node, UI element, or shape with `hoverCursorId`, the
client may switch to that cursor immediately. This is generic hit-test behavior,
not game logic.

The server can also set the current cursor explicitly for modes such as
dragging, targeting, building placement, or spell casting. If the server sets a
cursor, that cursor stays active until changed or cleared.

## Shapes

Shapes are first-class render nodes. They can be used for normal gameplay
objects, UI, editor tools, debugging, and visualization.

```text
Shape
  shapeId
  parentId
  layerId
  kind
  x
  y
  z
  x2
  y2
  z2
  radius
  color
  durationTicks
  hitTest
  hoverCursorId
  flags
```

Useful shapes include lines, rectangles, circles, boxes, cubes, paths, axes,
tile highlights, selection rings, ranges, and labels.

If a shape affects gameplay, the server owns that meaning. The client draws the
shape, but it does not decide collision, damage, targeting, or pathing from the
rendered primitive.

## Replay

Replay is a hard requirement. Every game run must be able to write enough data
to reconstruct the same sim and render stream.

```text
Replay
  protocolVersion
  simVersion
  assetManifestHash
  seed
  initialChunks
  inputStream
  chatInputStream
  simDeltas
  renderDeltas
  tickHashes
```

Chat input belongs to replay input, even when it comes from an AI policy. Chat
messages, prompts, and model responses that are visible to or consumed by the
game must be recorded and included in `inputHash`.

Per-tick hashes should be available for desync debugging:

```text
TickHash
  tick
  simHash
  inputHash
  renderHash
```

`renderHash` can be optional.

## Results

`results.json` is a hard requirement for completed game runs. It is the compact
machine-readable summary used by launchers, tournaments, tests, dashboards, and
batch tooling.

```text
Results
  gameId
  sessionId
  protocolVersion
  simVersion
  assetManifestHash
  seed
  startedAt
  endedAt
  status
  players
  scores
  winner
  replayPath
  replayHash
  tickCount
  finalSimHash
  error
```

`status` should distinguish normal completion, timeout, crash, cancellation, and
validation failure. `error` should be empty for successful runs.

## First Milestone

The first milestone should be intentionally small.

```text
One WASM client and one native client using the same protocol.
One flat 2D game ported from BitWorld.
Sprites as image quads.
Zero height map.
One camera.
Integer pointer input.
Replay can render the same session.
```

After that, add height maps, 3D models, animation, offsets, richer UI, text,
shapes, and chunk streaming.
