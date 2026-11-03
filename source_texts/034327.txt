# ⚡ GreedMesh

**WebGL2 voxel engine with greedy meshing** — procedural terrain, first-person controls, block placing/breaking.

Built from scratch with raw WebGL2 (no frameworks). Demonstrates advanced real-time graphics techniques.

## Features

- **Greedy Meshing** — Merges adjacent same-type faces into maximal quads, reducing vertex count 10-20x vs. naive rendering
- **Procedural Terrain** — FBM (Fractal Brownian Motion) 3D Perlin noise for heightmaps, cave systems, and biome variation (grass, dirt, stone, sand, snow, water)
- **First-Person Controls** — Pointer-lock camera (WASD + mouse look), physics with collision, jumping, and gravity
- **Block Interaction** — DDA raycast voxel picking — left click to remove, right click to place
- **Chunk Management** — Dynamic chunk loading/unloading based on view distance
- **WebGL2** — GLSL 300 ES shaders, VAOs, raw vertex buffer management

## Play

**[https://nwfella.github.io/GreedMesh/](https://nwfella.github.io/GreedMesh/)**

| Key | Action |
|-----|--------|
| W/A/S/D | Move |
| Space | Jump |
| Shift | Descend |
| Mouse | Look around |
| Left click | Remove block |
| Right click | Place block |
| 1-6 | Select block type |

## Architecture

```
src/
├── main.js              — Game loop, chunk management, physics, raycasting
├── engine/
│   ├── camera.js        — First-person camera (yaw/pitch, view-projection matrices)
│   ├── constants.js     — Block types, colors, physics constants
│   ├── input.js         — Keyboard/mouse state management
│   ├── noise.js         — 3D Perlin noise + FBM for terrain generation
│   └── renderer.js      — WebGL2 shaders, VAOs, draw calls
├── world/
│   └── chunk.js         — Voxel data storage, terrain generation
└── meshing/
    └── greedy.js        — Greedy meshing algorithm
```

## The Algorithm

Greedy meshing works by scanning each axis-direction pair and building a 2D mask of exposed faces per slice, then finding maximal rectangles in that mask. Each rectangle becomes a single quad instead of many individual faces — dramatically reducing GPU draw calls and vertex count.

## License

MIT
