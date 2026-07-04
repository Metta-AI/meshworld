# MeshWorld Protocol Sketch

This document is a draft shape for the protocol. It describes messages and
state, not a final binary encoding.

## Design Rules

- All protocol values are integers.
- Coordinates use 256 sim units per tile by default.
- Asset ids are local to one game and one asset manifest.
- The server sends render ids and sim data, not render asset bytes.
- The client can interpolate visuals, but sim state is tick-based.
- Gameplay commands should be validated against sim state.
- All clients support the full protocol feature set.
- Games omit messages for features they do not use.
- Floats are allowed only as local client render details, never wire state.
- Sim state must not depend on hash table order, pointer addresses, pointer
  hashes, nondeterministic random numbers, Unix time, wall-clock time, or thread
  scheduling.
- Sims should favor fixed-capacity containers and avoid memory allocation during
  ticks.

## Game Descriptor

The game descriptor lets any MeshWorld client run many games.

```text
GameDescriptor
  gameId
  name
  protocolVersion
  serverUrl
  assetManifestUrl
  assetManifestHash
  tickRate
  tileSize
  renderMode
```

MeshWorld game descriptors should be derived from valid Coworld manifests.
MeshWorld should not require a second game manifest format. The required
manifest shape remains:

```text
coworld_manifest.json
  game.name
  game.version
  game.runnable.image
  game.runnable.run
  game.config_schema
  game.results_schema
  player[]
  variants[].game_config
  certification
```

Standalone players should keep the `coplayer_manifest.json` format:

```text
coplayer_manifest.json
  name
  image
  run
  env
  games
```

Runtime game config JSON should keep BitWorld field names where applicable:

```text
tokens
players
slots
seed
minPlayers
maxGames
closedRoster
```

`slots[]` entries should continue to use schema-advertised fields such as
`name`, `token`, and `role`.

Example fixed-roster game config:

```json
{
  "tokens": ["token-a", "token-b"],
  "players": [
    {
      "name": "red"
    },
    {
      "name": "blue"
    }
  ],
  "slots": [
    {
      "name": "red",
      "token": "token-a",
      "role": "attacker"
    },
    {
      "name": "blue",
      "token": "token-b",
      "role": "defender"
    }
  ],
  "seed": 12345,
  "minPlayers": 2,
  "maxGames": 1,
  "closedRoster": true
}
```

Game containers should understand the BitWorld runtime environment variables:

```text
COGAME_SAVE_REPLAY_URI
COGAME_RESULTS_URI
COGAME_CONFIG_URI
COGAME_LOAD_REPLAY_URI
COGAME_HOST
COGAME_PORT
```

Player and policy containers should understand:

```text
COGAMES_ENGINE_WS_URL
```

The player websocket URL keeps the existing format:

```text
ws://host:port/player?name=<name>&slot=<slot>&token=<token>
```

## Asset Manifest

The asset manifest maps integer ids to fetchable assets.

During development, `assetManifestUrl` and asset URLs may point at the local
game HTTP or websocket server. For deployment, they should point at immutable
S3 or CDN-backed URLs.

```text
AssetManifest
  gameId
  manifestHash
  models
  images
  cursors
  fonts
  sounds
  effects
```

Model entries can define default scale, offsets, and animation ids.

```text
ModelAsset
  modelId
  url
  sha256
  defaultScale
  defaultOffsetX
  defaultOffsetY
  defaultOffsetZ
  defaultRotationOffset
  animations
```

Image entries define native size. A 2D image renders as a generated quad mesh.

```text
ImageAsset
  imageId
  url
  sha256
  nativeWidth
  nativeHeight
  frames
  animations
```

Sprite frames and animations are used by 2D sprites and particles.

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

Cursor entries define pointer images and hot spots. `imageId` refers to an
image asset in the same manifest.

```text
CursorAsset
  cursorId
  imageId
  hotspotX
  hotspotY
```

Font entries define assets used by text nodes. The text string itself is state,
not an asset.

```text
FontAsset
  fontId
  url
  sha256
  defaultSize
```

Sound entries define audio assets. Playback state is sent separately.

```text
SoundAsset
  soundId
  url
  sha256
  channels
  sampleRate
```

Effect entries define particle and visual effect assets. Emitter state is sent
separately.

```text
EffectAsset
  effectId
  url
  sha256
  defaultDurationTicks
  imageId
  animationId
```

Example asset manifest JSON:

```json
{
  "gameId": "arena",
  "manifestHash": "sha256:manifest0001",
  "models": [
    {
      "modelId": 1,
      "url": "https://assets.example.com/arena/models/knight.glb",
      "sha256": "sha256:model0001",
      "defaultScale": 1000,
      "defaultOffsetX": 0,
      "defaultOffsetY": 0,
      "defaultOffsetZ": -24,
      "defaultRotationOffset": 0,
      "animations": [
        {
          "animationId": 1,
          "name": "idle",
          "defaultSpeed": 1000
        },
        {
          "animationId": 2,
          "name": "walk",
          "defaultSpeed": 1000
        },
        {
          "animationId": 3,
          "name": "attack",
          "defaultSpeed": 1200
        }
      ]
    }
  ],
  "images": [
    {
      "imageId": 10,
      "url": "https://assets.example.com/arena/images/unit.png",
      "sha256": "sha256:image0010",
      "nativeWidth": 64,
      "nativeHeight": 64,
      "frames": [
        {
          "frameId": 1,
          "x": 0,
          "y": 0,
          "width": 64,
          "height": 64,
          "offsetX": 0,
          "offsetY": 0
        },
        {
          "frameId": 2,
          "x": 64,
          "y": 0,
          "width": 64,
          "height": 64,
          "offsetX": 0,
          "offsetY": 0
        }
      ],
      "animations": [
        {
          "animationId": 101,
          "name": "idle",
          "frames": [1, 2],
          "frameTicks": 8,
          "loop": true
        }
      ]
    },
    {
      "imageId": 11,
      "url": "https://assets.example.com/arena/images/button.png",
      "sha256": "sha256:image0011",
      "nativeWidth": 128,
      "nativeHeight": 32
    },
    {
      "imageId": 12,
      "url": "https://assets.example.com/arena/images/cursor_attack.png",
      "sha256": "sha256:image0012",
      "nativeWidth": 32,
      "nativeHeight": 32
    }
  ],
  "cursors": [
    {
      "cursorId": 50,
      "imageId": 12,
      "hotspotX": 4,
      "hotspotY": 4
    }
  ],
  "fonts": [
    {
      "fontId": 20,
      "url": "https://assets.example.com/arena/fonts/ui.ttf",
      "sha256": "sha256:font0020",
      "defaultSize": 16
    }
  ],
  "sounds": [
    {
      "soundId": 30,
      "url": "https://assets.example.com/arena/sounds/hit.ogg",
      "sha256": "sha256:sound0030",
      "channels": 2,
      "sampleRate": 48000
    }
  ],
  "effects": [
    {
      "effectId": 40,
      "url": "https://assets.example.com/arena/effects/spark.json",
      "sha256": "sha256:effect0040",
      "defaultDurationTicks": 24,
      "imageId": 10,
      "animationId": 101
    }
  ]
}
```

## Session Start

The server sends the session setup before state.

```text
SessionStart
  sessionId
  gameId
  protocolVersion
  simVersion
  assetManifestHash
  tickRate
  tileSize
  seed
```

The client must reject mismatched protocol or manifest hashes unless an
explicit compatibility rule exists.

## Chunks

Height chunks are authoritative sim data.

```text
SetHeightChunk
  tick
  chunkId
  chunkX
  chunkY
  layer
  width
  height
  baseHeight
  encodedHeights
  encodedFlags
  hash
```

Chunk deltas patch existing chunks.

```text
SetHeightDelta
  tick
  chunkId
  tileIndex
  oldHeight
  newHeight
  oldFlags
  newFlags
```

Large worlds can stream chunks.

```text
LoadChunk
  tick
  chunkId
  hash

UnloadChunk
  tick
  chunkId
```

## Render Nodes

Render nodes describe visible objects.

```text
UpsertRenderNode
  tick
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

RemoveRenderNode
  tick
  nodeId
```

Scale uses fixed point:

```text
1000 = 1.0
```

Rotation uses 360 integer values unless a game opts into a finer rotation
scale.

## Animation

```text
SetAnimation
  tick
  nodeId
  animationId
  mode
  startTick
  speed
```

Animation modes:

```text
Loop
Once
Hold
Stop
```

For model assets, `animationId` selects a model animation. For image assets,
`animationId` selects a sprite animation. The same message drives both 3D model
animation and 2D sprite animation.

## UI

```text
UpsertUiNode
  tick
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
  visible
```

UI nodes are render nodes with screen-space layout rules.

## Text

```text
UpsertTextNode
  tick
  nodeId
  parentId
  layerId
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

Text nodes may be screen-space, world-space, billboards, or flat world labels.

## Sounds

```text
UpsertSoundNode
  tick
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

RemoveSoundNode
  tick
  soundNodeId
```

Sound modes:

```text
WorldSound
ScreenSound
CameraSound
```

Volume and pitch use fixed point where `1000 = 1.0`.

## Cursors

```text
SetCursor
  tick
  pointerId
  cursorId
```

`cursorId = 0` clears the explicit cursor and returns to hover/default cursor
behavior.

## Particles

```text
UpsertParticleEmitter
  tick
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

RemoveParticleEmitter
  tick
  emitterId
```

Particle fields are integers. `scale`, `rate`, and `speed` use fixed point
where `1000 = 1.0`. `animationId` may select a sprite animation from the effect
asset's image.

## Shapes

```text
UpsertShape
  tick
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

RemoveShape
  tick
  shapeId
```

`durationTicks = 0` means the shape lives until removed.

## Cameras

```text
UpsertCamera
  tick
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

## Input

Pointer input is integer-only.

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

The server should resolve pointer events into deterministic commands.

```text
ResolvedCommand
  tick
  commandKind
  targetEntityId
  tileX
  tileY
  tileZ
  uiNodeId
```

## Replay

Replay output is required for every game run.

```text
ReplayHeader
  protocolVersion
  simVersion
  gameId
  assetManifestHash
  seed
  tickRate
  tileSize

ReplayBody
  initialState
  inputStream
  simDeltas
  renderDeltas
  soundDeltas
  particleDeltas
  tickHashes
```

The replay must store all sim chunk data or immutable references to exact
chunk hashes.

## Results

`results.json` output is required for every completed game run.

```text
ResultsJson
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

Example `results.json`:

```json
{
  "gameId": "arena",
  "sessionId": "run-000001",
  "protocolVersion": 1,
  "simVersion": 3,
  "assetManifestHash": "sha256:manifest0001",
  "seed": 12345,
  "startedAt": "2026-07-04T12:00:00Z",
  "endedAt": "2026-07-04T12:03:10Z",
  "status": "complete",
  "players": [
    {
      "playerId": 1,
      "name": "red"
    },
    {
      "playerId": 2,
      "name": "blue"
    }
  ],
  "scores": [
    {
      "playerId": 1,
      "score": 10
    },
    {
      "playerId": 2,
      "score": 7
    }
  ],
  "winner": 1,
  "replayPath": "replay.meshworld",
  "replayHash": "sha256:replay0001",
  "tickCount": 9120,
  "finalSimHash": "sha256:simfinal0001",
  "error": ""
}
```
