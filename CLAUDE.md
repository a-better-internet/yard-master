# Yard Master 3D

A single-file three.js (r128) game. Everything — markup, styles, game code — lives in
`index.html`. Dependencies are two CDN script tags (three.js, Tailwind); there are no
build steps, no bundler, and no asset files. Textures are generated procedurally into
canvases at load, so keep it that way: no image, model or audio fetches.

## Running it

Open `index.html` in a browser. For automated checks, serve or open it over `file://`
with a headless Chromium — WebGL works under swiftshader, just slowly (expect ~10fps,
which is the renderer, not the game).

To lint, extract the script block and run `node --check` on it:

```sh
sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' > /tmp/game.js
node --check /tmp/game.js
```

## Gotchas that have bitten us

These are real bugs that shipped. Check for them before writing similar code.

### Gable roofs: Euler order decides which way the slopes fall

A gable roof is two slabs, each offset along its own local ±Z from the ridge and tilted
by ±pitch about its own X. If the building is also yawed, **the yaw has to be applied
outermost or the roof comes out inverted** — both slopes fall inward into a V instead of
meeting at a ridge.

`THREE.Euler`'s default order `'XYZ'` builds `Rx * Ry * Rz`, so a yaw in `ry` is applied
*before* the pitch in `rx`, and the pitch then tilts the slab about the **world** X axis
rather than its own. Use `'YXZ'` (`Ry * Rx * Rz`) so the yaw is outermost:

```js
euler.set(pitch, yaw, 0, 'YXZ');
```

`partBuilder().add()` already does this. If you build a roof any other way — a Group, a
raw Mesh, a Matrix4 — set the order yourself, and pass the building's yaw to *both* the
part's position and its rotation. Getting the offset yawed but not the tilt is exactly
what produced the inverted roofs.

Sanity check any new roof by looking at it from a low angle on a rotated instance, not
just an axis-aligned one. Axis-aligned buildings hide this bug completely.

### `vertexColors: true` needs an actual `color` attribute

Per-instance tinting via `InstancedMesh.setColorAt()` uses `instanceColor`, which is a
**separate** shader path from `vertexColors`. Setting `vertexColors: true` on a material
whose geometry has no `color` attribute makes the shader read zero and the mesh renders
solid black.

- Tinting per instance only → leave `vertexColors` off; `setColorAt` works on its own.
- Tinting per vertex (merged parts under one material) → the geometry needs a `color`
  attribute, and any merge helper has to carry it through. `concatGeos` silently dropped
  it once and turned a row of houses black.
- Both at once is fine: r128 multiplies `color * instanceColor`.

### Fences and walls are built level, not draped over the terrain

Sampling `getElevation()` per picket makes the fence undulate and look broken. Build to a
single world Y (the max ground height along the run, plus the fence height) and make the
pickets long enough to stay buried at the low corners. The ground is allowed to slope
underneath; the fence is not.

### Grass outside the fence reads as mowable

Only the yard grid inside the fence can be cut. Anything out there that looks like the
lawn implies it can be mowed and confuses players. Use bushes, ferns, scrub, rocks and
logs for ground cover instead — visually distinct from lawn tufts.

### The yard is a square, so exclusion zones must be square

Scattering scenery through a *circular* annulus whose inner radius is the fence's half
width puts props inside the yard near the diagonals — at 45° a point at r = 20.9 has
|x| = |z| = 14.8, well within a fence at ±20.4. Sample a square ring instead
(`Math.max(|x|, |z|) < half + clearance`) and add each prop's own spread to the
clearance, or bushes overhang the pickets.

### Minigame prop groups must default to hidden

`setGameMode()` sets `.visible` on every minigame group, but it only runs once the
player enters a mode. A group left visible at build time sits in the middle of the yard
on the title screen and through mowing — that's how the golf club, ball, flag and aim
line ended up parked at the origin. Set `visible = false` where the group is created.

### `font: <weight> <size>/<lh> inherit` is invalid and silently dropped

`inherit` is not an accepted family inside the `font` shorthand, so the whole
declaration is discarded and the text falls back to the browser's 16px default. Set
`font-family` once on the container and use `font-weight` / `font-size` / `line-height`
longhand on the children.

### There are two ground surfaces — use `surfaceY()`, not `getElevation()`

`getElevation(x, z)` is the terrain function, not the height of the thing you can
actually stand on. The lawn plate is drawn at `elevation - 0.1` with the grass tufts
sitting on top at `elevation`, and the surrounding field is at `elevation - OUTER_DROP`.
A prop placed outside the fence with a bare `getElevation()` hovers — that is what left
the garden path, the flagstones and the small scenery floating. `surfaceY(x, z)` returns
whichever surface is under the point; use it for anything that rests on the ground.

Props built inside a `Group` (the house, the shed) are placed relative to the group's
origin, so a part several metres away along a slope needs
`surfaceY(worldX, worldZ) - baseY` as its local offset, not a constant. The porch steps
and foundation planting both floated for exactly this reason.

### Unlit materials ignore the day/night cycle

`MeshBasicMaterial` on distant scenery looks fine at noon and then stays a bright band
through sunset and night while everything around it darkens. Anything meant to sit in
the world needs a lit material (`MeshStandardMaterial`), or its colour has to be driven
from the sky palette every frame.

### Headless checks: CSS transitions crawl under swiftshader

The software renderer starves the compositor, so a `transition: opacity .4s` reads about
0.03 after 600ms and screenshots look like the element never appeared. Don't chase it as
a bug — set `style.transition = 'none'` before asserting or screenshotting, or read the
inline value rather than the computed one.

## Architecture notes

- **Grass** is one `InstancedMesh` per lawn (yard + infinity), each instance a tuft of
  tapered blades. Wind is a shared uniform block (`grassUniforms`) evaluated per-vertex
  in a `onBeforeCompile` injection, so it costs no per-frame CPU. Mowed cells are the
  same instances squashed on Y and re-tinted; nothing is added or removed.
- **Per-cell randomness must be deterministic.** Use `deterministicHash(x, z)`, never
  `Math.random()`, for anything a cell might need to reproduce — regrowing grass that
  re-rolls its colour flickers.
- **Detailed props** go through `partBuilder()`, which buckets parts by material and
  merges each bucket into one geometry. The whole house is ~6 draw calls. Box UVs span
  0..1 per face, so scale them per part (`uvScale`) or shared tiling textures stretch
  differently on every piece.
- **Textures** are drawn into canvases by `makeTex(key, size, drawFn, repeatX, repeatY)`
  and cached by key. `makeNormalTex` derives a normal map from the same pixels via a
  Sobel pass. Add new surfaces as `drawSomething(ctx, size)` functions.
- **Draw-call and triangle budget**: classic mode currently runs ~150 calls and ~700k
  triangles. Background scenery should not cast shadows — the shadow camera only covers
  ~28 units around the player, so casting from distant props is pure cost.
- **Adaptive resolution** (`updateAdaptiveQuality`) scales the backbuffer when the frame
  rate drops. Prefer that over thinning the yard.
- **Colour** goes through `installToneMapping()`, which wraps ACES in a saturation lift
  (`COLOR_SATURATION`) via `CustomToneMapping`. ACES desaturates as it rolls off
  highlights, which greyed the whole palette. Reach for that constant to adjust overall
  vividness — it is a colour operation and leaves every light intensity untouched. It
  patches a three.js shader chunk, so it must run before the first material compiles,
  and it falls back to plain ACES if the chunk it expects isn't there.
