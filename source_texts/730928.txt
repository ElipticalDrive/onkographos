![A procedurally generated hex terrain](media/hero.png)

[![core](https://img.shields.io/github/v/tag/FraserFallows/hex-map?filter=v*&sort=semver&label=core)](https://github.com/FraserFallows/hex-map/releases)
[![pathfinding](https://img.shields.io/github/v/tag/FraserFallows/hex-map?filter=pathfinding-v*&sort=semver&label=pathfinding)](https://github.com/FraserFallows/hex-map/releases)

Procedurally generated, height-mapped hex terrain for Unity: a grid of hexagonal cells with
per-cell elevation and surface type, rendered as tiered 3D terrain with walls between
neighbouring height steps. Pre-1.0; the API may still change.

## Features

- **Deterministic generation** — an outline (hexagon, rectangle, parallelogram), size, seed, and
  height source produce a reproducible map.
- **Composable heightmaps** — build elevation from a graph of noise nodes (Perlin, Simplex and
  Worley through fractal, domain-warp, curve, terrace, remap, blur, invert and layered-blend),
  saved as reusable `Heightmap` assets. Image and flat height sources are also supported.
- **Procedural surface classification** — a `Surfacemap` scores slope, altitude, convexity and a
  noise field to tag each cell grass, dirt or rock, drawn by a palette shader with feathered
  cell outlines.
- **Live editor tuning** — adjust either recipe in the inspector and the map updates immediately.

## Packages

- **`com.hexterra.core`** — the generator, hex maths, noise system, and mesh/shader content. The
  package to install.
- **`com.hexterra.pathfinding`** — optional A* and reachability queries over a generated map.
  Depends on core.

## Requirements

Unity 2022.3 or newer, with the Universal Render Pipeline.

## Installation

In Package Manager, choose **Add package from git URL**.

**Core only:**

```
https://github.com/FraserFallows/hex-map.git?path=/Packages/com.hexterra.core
```

**With pathfinding** — add core first, then add:

```
https://github.com/FraserFallows/hex-map.git?path=/Packages/com.hexterra.pathfinding
```

Append a tag from [Releases](https://github.com/FraserFallows/hex-map/releases) to pin a
version, e.g. `#v0.3.0`. Or clone the repository and add either package folder through
**Add package from disk**.

## Quick start

1. Add a **HexMap** component to an empty GameObject.
2. Assign the hex top and wall prefabs and the surface and edge materials from
   `Packages/com.hexterra.core/Content/`.
3. Assign the recipe assets from `Packages/com.hexterra.core/Data/`: a `Heightmap` (e.g.
   `Hills`), a `Surfacemap` (`RockyHills_Grass01`), and a `SurfacePaletteSet`
   (`DefaultSurfacePalette`).
4. Set the shape, size and seed.
5. Press **Generate** in the inspector. `HexMap` also builds itself on `Start`.

## Pathfinding

With `com.hexterra.pathfinding` installed:

1. Add a **PathfindingVisualiser** to a GameObject — it brings a **Pathfinder** with it.
2. At the bottom of `HexMap` inspector, add the `Pathfinder` to the **Map Generated** event, select
   `RebuildGraph`, and regenerate.
3. With that GameObject selected, click a hex in the scene view to set the start, shift-click for
   the goal. The route and reachable range draw as gizmos.

`Pathfinder.TryFindPath` / `TryFindReachable` answer the same queries from code.

Hex coordinates and neighbour maths follow
[Red Blob Games: Hexagonal Grids](https://www.redblobgames.com/grids/hexagons/).

## License

MIT. See [LICENSE](LICENSE).
