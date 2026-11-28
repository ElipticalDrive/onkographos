# Deterministic Fixture Worlds

Also called: Deterministic World Generation

## Overview
Deterministic Fixture Worlds are pure functions that generate specific, small-scale voxel structures based solely on a `VolumeId`, ensuring no reliance on random number generators (RNG), system clocks, or I/O [S1, unofficial]. They serve as regression tests and acceptance criteria for voxel engines, providing predictable geometric primitives like slabs, towers, and bridges [S1, unofficial]. Unlike general procedural generation which uses noise for terrain, fixture worlds are hand-crafted logical shapes defined by exact brick coordinates [S1, unofficial]. They build on [Procedural Voxel World Generation](3891-procedural-voxel-world-generation.md) and sit alongside [Biome and Voxel Layer Determination](4167-biome-and-voxel-layer-determination.md) and [Procedural Tree Generation](4826-procedural-tree-generation.md).

## How it works
The method consists of four specific generator functions and a verification digest [S1, unofficial]. All functions are pure and deterministic [S1, unofficial].

### Inputs and Outputs
- **Input**: A `VolumeId` for generation context and the `Volume` state for the digest [S1, unofficial].
- **Output**: A set of voxel blocks placed in the world grid, or a hash digest of the resulting world state [S1, unofficial].

### Data Structures
- **Brick**: A unit cube in the voxel grid with fields `coord` (integer triplet), `content hash` (block type identifier), and `revision` (integer) [S1, unofficial].
- **Volume**: A collection of resident bricks accessible via `Volume::resident_brick_coords()` in canonical order [S1, unofficial].
- **Hash**: A platform-stable BLAKE3 output [S1, unofficial].

### Steps
1. **Generate Flat Terrain**: Creates a $2 \times 2 \times 2$ brick bounded stone slab under a resident (non-absent) air layer, using uniform storage [S1, unofficial].
2. **Generate Hollow Tower**: Creates an 8-brick, 2-cell-thick stone shell straddling the origin (negative coordinates, cross-brick) [S1, unofficial]. Verification confirms brushes touch exactly eight bricks when centred on the corner where eight meet; a cell centre exactly on the radius is included [S1, unofficial].
3. **Generate Cross Brick Bridge**: Creates a $2 \times 2$-cell beam spanning four X bricks, matching the bridge-cross-brick acceptance fixture [S1, unofficial].
4. **Generate Sloped Terrain**: Creates a small excavatable staircase with a dirt surface over stone [S1, unofficial].
5. **Compute Digest**: Computes a platform-stable BLAKE3 hash over the `Volume` [S1, unofficial]. Iterates over resident bricks in canonical order $(z, y, x)$ [S1, unofficial]. Includes fields: `coord`, `content hash`, and `revision` [S1, unofficial]. This digest is a local regression check and is explicitly not the replicated `spall.topology.v1` hash [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import List, Tuple
import hashlib

@dataclass
class Brick:
    coord: Tuple[int, int, int]
    content_hash: str
    revision: int

@dataclass
class Volume:
    bricks: List[Brick]

    def resident_brick_coords(self) -> List[Brick]:
        return sorted(self.bricks, key=lambda b: (b.coord[2], b.coord[1], b.coord[0]))

def generate_flat_terrain(volume_id: str) -> List[Brick]:
    bricks = []
    for x in range(2):
        for y in range(2):
            for z in range(2):
                bricks.append(Brick(coord=(x, y, z), content_hash="stone", revision=1))
    return bricks

def generate_hollow_tower(volume_id: str) -> List[Brick]:
    bricks = []
    coords = [
        (-1, -1, -1), (-1, -1, 0), (-1, 0, -1), (-1, 0, 0),
        (0, -1, -1), (0, -1, 0), (0, 0, -1), (0, 0, 0)
    ]
    for c in coords:
        bricks.append(Brick(coord=c, content_hash="stone", revision=1))
    return bricks

def generate_cross_brick_bridge(volume_id: str) -> List[Brick]:
    bricks = []
    for x in range(4):
        for y in range(2):
            for z in range(2):
                bricks.append(Brick(coord=(x, y, z), content_hash="stone", revision=1))
    return bricks

def generate_sloped_terrain(volume_id: str) -> List[Brick]:
    bricks = []
    for step in range(3):
        x_start = step
        for x in range(x_start, x_start + 3):
            for y in range(2):
                bricks.append(Brick(coord=(x, y, step), content_hash="stone", revision=1))
                bricks.append(Brick(coord=(x, y, step + 1), content_hash="dirt", revision=1))
    return bricks

def digest_volume(volume: Volume) -> str:
    data_parts = []
    for brick in volume.resident_brick_coords():
        data_parts.append(f"{brick.coord[0]},{brick.coord[1]},{brick.coord[2]}:{brick.content_hash}:{brick.revision}")
    data_string = "|".join(data_parts)
    return hashlib.blake3(data_string.encode('utf-8')).hexdigest()
```

### Parameters
- **VolumeId**: String identifier for the volume context [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Ray results are explicitly not required to be bit-identical across machines; hit validation is server-side [S1, unofficial]. Integer brush plans stay fully deterministic [S1, unofficial].
- **Boundary**: A cell centre exactly on the radius is included in the hollow tower generation [S1, unofficial].
- **Digest Confusion**: The digest is a local regression check and is explicitly not the replicated `spall.topology.v1` hash [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ for generation, as the structures are fixed-size and small, and $O(N)$ for digest, where $N$ is the number of resident bricks [S1, unofficial].
- **Memory Complexity**: $O(1)$ for generation and $O(N)$ for digest [S1, unofficial].
- **Strengths**: Deterministic, platform-stable (for digest), easy to test, and no RNG dependencies [S1, unofficial].
- **Limitations**: Only suitable for small, fixed structures and not suitable for large-scale terrain generation [S1, unofficial].
- **When to use**: Regression testing, acceptance criteria, and debugging voxel engine geometry [S1, unofficial].
- **When not to use**: Large-scale world generation and dynamic content [S2, unofficial].

## Background
Deterministic Fixture Worlds were introduced in the `spall` voxel engine project [S1, unofficial]. They are part of a broader effort to ensure voxel engine correctness through pure functions and deterministic testing [S1, unofficial]. The concept is distinct from general procedural generation, which often relies on noise and hashing for terrain and object placement [S2, unofficial].

## Sources
[S1] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
[S2] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
