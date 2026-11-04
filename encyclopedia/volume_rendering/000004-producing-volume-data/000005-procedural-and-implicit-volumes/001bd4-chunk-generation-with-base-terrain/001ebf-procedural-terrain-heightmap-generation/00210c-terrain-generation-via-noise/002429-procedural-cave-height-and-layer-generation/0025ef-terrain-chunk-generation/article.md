]

# Terrain Chunk Generation

Also called: FBM Terrain Synthesis, terrainChunk

## Overview
Generates a voxel chunk using two layers of fractal Brownian motion (fBm) noise: one for surface height and one for cave depth [S1, unofficial]. It assigns materials based on vertical distance from the surface and absolute Y position [S1, unofficial]. Builds on [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md) for procedural terrain implementation [G].

## How it works
**Inputs:**
- `seed`: number (default 1337). Deterministic seed for noise generation [S1, unofficial].
- `size`: number (default `CHUNK_SIZE`). Dimensions of the cubic chunk ($size \times size \times size$) [S1, unofficial].
- `materials`: number (default 4). Number of material layers. When set to 1, forces single-material output for benchmarking sensitivity [S1, unofficial].

**Outputs:**
- A `VoxelChunk` data structure filled with material IDs [S1, unofficial].

**Data Structures:**
- `VoxelChunk`: A 3D array or flat buffer of size $size^3$ storing integer material IDs [S1, unofficial].
- Noise Generators: Two independent fBm noise functions, `surface` and `cave` [S1, unofficial].

**Steps:**
1. Initialize `surface` noise generator with `seed` [S1, unofficial].
2. Initialize `cave` noise generator with `seed ^ 0x9e3779b9` (XOR with the golden ratio constant) [S1, unofficial].
3. Set cave noise to 3 octaves [S1, unofficial].
4. Iterate $z$ from 0 to $size-1$ [S1, unofficial].
5. Iterate $x$ from 0 to $size-1$ [S1, unofficial].
6. Compute surface height:
   $$ height = \lfloor (0.55 + surface(\frac{x}{size} \cdot 3.5, \frac{z}{size} \cdot 3.5) \cdot 0.32) \cdot size \rfloor $$
   where $surface$ is the fBm noise function [S1, unofficial].
7. Iterate $y$ from 0 to $\min(height, size)-1$ [S1, unofficial].
8. Check for cave occlusion:
   - Compute cave value: $c = cave(\frac{x}{size} \cdot 5, \frac{z}{size} \cdot 5 + y \cdot 0.21)$ [S1, unofficial].
   - If $y > 2$ AND $c > 0.45$, skip this voxel (it is inside a cave) [S1, unofficial].
9. Compute depth: $depth = height - y$ [S1, unofficial].
10. Assign material based on `materials` count and `depth`:
    - If `materials` == 1: material = 1 [S1, unofficial].
    - Else if $depth \le 1$: material = 1 [S1, unofficial].
    - Else if $depth \le 4$: material = 2 [S1, unofficial].
    - Else if $y < 4$: material = 4 [S1, unofficial].
    - Else: material = 3 [S1, unofficial].
11. Set voxel at $(x, y, z)$ to the assigned material [S1, unofficial].

**Pitfalls:**
- Cave noise uses a different seed and 3 octaves to drill into the chunk, creating inner surfaces [S1, unofficial].
- The $y$-offset in cave sampling ($y \cdot 0.21$) ensures caves are not perfectly vertical columns, allowing culling tests to distinguish between outer and inner faces [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terrain_chunk(seed=1337, size=64, materials=4):
    # Standard 2D fBm noise using hash-based base [G]
    def fbm(x, y, octaves, seed):
        val = 0.0
        amp = 1.0
        freq = 1.0
        for _ in range(octaves):
            val += amp * ((seed + int(x * 12.9898) + int(y * 78.233)) % 10000 / 10000.0 - 0.5)
            amp *= 0.5
            freq *= 2.0
        return val

    surface_noise = lambda x, y: fbm(x, y, 3, seed)
    cave_noise = lambda x, y: fbm(x, y, 3, seed ^ 0x9e3779b9)

    class VoxelChunk:
        def __init__(self, size):
            self.size = size
            self.data = [[[0] * size for _ in range(size)] for _ in range(size)]
        def set(self, x, y, z, val):
            self.data[x][y][z] = val

    chunk = VoxelChunk(size)

    for z in range(size):
        for x in range(size):
            nx, nz = x / size, z / size
            s_val = surface_noise(nx * 3.5, nz * 3.5)
            height = int((0.55 + s_val * 0.32) * size)
            if height >= size: height = size - 1
            if height < 0: height = 0

            for y in range(min(height, size)):
                if y > 2:
                    c_val = cave_noise(nx * 5, nz * 5 + y * 0.21)
                    if c_val > 0.45:
                        continue
                depth = height - y
                if materials == 1:
                    mat = 1
                elif depth <= 1:
                    mat = 1
                elif depth <= 4:
                    mat = 2
                elif y < 4:
                    mat = 4
                else:
                    mat = 3
                chunk.set(x, y, z, mat)
    return chunk
```

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
