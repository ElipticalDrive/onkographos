# Sparse Octree Terrain Representation

Also called: Sparse Octree Reconstruction Model, Sparse Octree Terrain

## Overview
Sparse Octree Terrain Representation replaces dense heightmap diffusion with a sparse octree structure that stores only occupied voxels, aligning with the Voxy schema to avoid wasting compute on empty cells [S1, unofficial]. Unlike previous methods that diffuse all 256 values of a full 16x16 heightmap regardless of terrain presence, this method works only over the occupied 8x8 patch surface, storing only non-empty octree nodes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A chunk coordinate pair $(chunkX, chunkZ)$, an $8 \times 8$ heightmap ($heightmap_{8 \times 8}$), biome data ($biomes_{64}$), and a Level of Detail integer ($lod$) [S1, unofficial].
*   **Output:** A `TerrainPatch` object containing a set of occupied voxels and a binary occupancy grid [S1, unofficial].

**Data Structures:**
1.  **Voxel:** An immutable value class representing a single occupied voxel at $(x, y, z)$ in octree-local space [S1, unofficial]. It implements `equals` and `hashCode` to be safe as a `Set` key [S1, unofficial].
2.  **TerrainPatch:** Contains two fields computed from the heightmap:
    *   `Set<Voxel> occupiedVoxels`: An unmodifiable set of terrain positions [S1, unofficial].
    *   `int[][][] octreeStructure`: An $8 \times 8 \times 8$ binary occupancy grid indexed by $[x][y][z]$, where the Y-axis quantizes the height range into $OCTREE\_SIZE$ levels [S1, unofficial].

**Parameters:**
*   $OCTREE\_SIZE$: A constant integer with value 8, defining the dimension of the binary occupancy grid and the number of Y-axis quantization levels [S1, unofficial].

**Steps:**
1.  **Validation:** Validate that the input $heightmap_{8 \times 8}$ has exactly $8 \times 8$ dimensions and reject input of wrong size [S1, unofficial].
2.  **Smoothing:** Apply $OCTREE\_SIZE - lod$ smoothing passes to the heightmap [S1, unofficial].
    *   If $lod = 0$, apply 8 smoothing passes for the most detail [S1, unofficial].
    *   If $lod \ge 8$, apply 0 smoothing passes as a no-op [S1, unofficial].
3.  **Quantization and Occupancy:** Quantize the height range of the smoothed heightmap into $OCTREE\_SIZE$ levels along the Y-axis [S1, unofficial]. Populate the `octreeStructure` $8 \times 8 \times 8$ binary grid based on the quantized heights [S1, unofficial]. Populate the `occupiedVoxels` set with the coordinates of all non-zero entries in the grid [S1, unofficial].
4.  **Invariant Enforcement:** Ensure the lowest Y-level in the `octreeStructure` is always occupied, even in the edge case of a flat heightmap [S1, unofficial].
5.  **Return:** Return the constructed `TerrainPatch` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import List, Set, Tuple

OCTREE_SIZE = 8

@dataclass(frozen=True)
class Voxel:
    x: int
    y: int
    z: int

@dataclass
class TerrainPatch:
    occupied_voxels: Set[Voxel]
    octree_structure: List[List[List[int]]]

def apply_smoothing(heightmap: List[List[float]], passes: int) -> List[List[float]]:
    # Standard 3x3 box blur pass [G]
    smoothed = [[0.0] * len(heightmap[0]) for _ in range(len(heightmap))]
    for _ in range(passes):
        new_map = [[0.0] * len(heightmap[0]) for _ in range(len(heightmap))]
        for x in range(len(heightmap)):
            for z in range(len(heightmap[0])):
                total = 0.0
                count = 0
                for dx in range(-1, 2):
                    for dz in range(-1, 2):
                        nx, nz = x + dx, z + dz
                        if 0 <= nx < len(heightmap) and 0 <= nz < len(heightmap[0]):
                            total += heightmap[nx][nz]
                            count += 1
                new_map[x][z] = total / count
        smoothed = new_map
    return smoothed

def quantize_height(height_val: float, octree_size: int) -> int:
    # Standard clamp to discrete Y-levels [G]
    return max(0, min(octree_size - 1, int(height_val)))

def build_sparse_octree(chunk_x: int, chunk_z: int, heightmap_8x8: List[List[float]], biomes_64: object, lod: int) -> TerrainPatch:
    # Step 1: Validation
    if len(heightmap_8x8) != OCTREE_SIZE or any(len(row) != OCTREE_SIZE for row in heightmap_8x8):
        raise ValueError("Heightmap must be 8x8")

    # Step 2: Smoothing
    num_passes = OCTREE_SIZE - lod
    if num_passes > 0:
        smoothed_heightmap = apply_smoothing(heightmap_8x8, num_passes)
    else:
        smoothed_heightmap = heightmap_8x8

    # Step 3: Quantization and Occupancy
    octree_structure = [[[0 for z in range(OCTREE_SIZE)] for y in range(OCTREE_SIZE)] for x in range(OCTREE_SIZE)]
    occupied_voxels: Set[Voxel] = set()

    for x in range(OCTREE_SIZE):
        for z in range(OCTREE_SIZE):
            height_val = smoothed_heightmap[x][z]
            y_level = quantize_height(height_val, OCTREE_SIZE)
            octree_structure[x][y_level][z] = 1
            occupied_voxels.add(Voxel(x, y_level, z))

    # Step 4: Invariant Enforcement
    has_occupancy_at_y0 = any(octree_structure[x][0][z] == 1 for x in range(OCTREE_SIZE) for z in range(OCTREE_SIZE))
    if not has_occupancy_at_y0:
        for x in range(OCTREE_SIZE):
            for z in range(OCTREE_SIZE):
                octree_structure[x][0][z] = 1
                occupied_voxels.add(Voxel(x, 0, z))

    # Step 5: Return
    return TerrainPatch(occupied_voxels, octree_structure)
```

**Edge Cases and Pitfalls:**
*   **Lowest-Y-level Invariant:** The lowest Y-level in the octree structure must always be occupied, even in the edge case of a flat heightmap [S1, unofficial].
*   **Input Size:** The input heightmap must be exactly $8 \times 8$ and wrong-size input is rejected [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The build process is $O(OCTREE\_SIZE^2)$ for the heightmap processing and smoothing, where $OCTREE\_SIZE = 8$ [S1, unofficial]. This is significantly faster than the previous dense $16 \times 16$ diffusion which processed 256 values [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of occupied voxels and the fixed $8 \times 8 \times 8$ grid [S1, unofficial]. The `occupiedVoxels` set stores only non-empty nodes, providing compression proportional to sparsity [S1, unofficial].
*   **Strengths:** Avoids wasting compute on empty cells, aligns with Voxy schema, and remains fully backward-compatible [S1, unofficial].
*   **Limitations:** Limited to $8 \times 8$ patches and uses a fixed resolution defined by $OCTREE\_SIZE$ [S1, unofficial].
*   **When to use:** When terrain sparsity is high and compute efficiency is critical, or when integrating with Voxy storage schema [S1, unofficial].

## Background
This method was introduced in a pull request to the LODiffusion project to replace the existing LOD pipeline that forced dense 16x16 heightmap diffusion for every chunk [S1, unofficial]. It aligns the data contract with Voxy's octree storage schema [S1, unofficial].

## Sources
[S1] feat: sparse octree terrain representation aligned with Voxy schema, https://github.com/rhythmatician/LODiffusion/pull/17, fetched 2026-10-26 (unofficial)
