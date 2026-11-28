# Random Spots Noise
Also called: Cellular Noise (Spot variant), Spots Pattern
## Overview
Random Spots Noise is a procedural noise function that generates random spots within a 3D grid cell, supporting multiple shapes (step, linear, quadratic) and jitter [S1, unofficial]. It solves the problem of generating organic, spot-like structures for volume rendering by evaluating a 3x3x3 neighborhood of grid cells and applying a shape function to each spot's distance [S1, unofficial]. Unlike its sibling [Worley Noise](1726-worley-noise.md), which explicitly computes geometric distances to scattered feature points to create distinct cell boundaries, Random Spots Noise generates spots within grid cells using deterministic seeds, creating isolated spot intensities [S1, unofficial]. It builds on [Spot Noise](2959-spot-noise.md) by defining a specific sparse convolution kernel where impulses are placed deterministically within grid cells.
## How it works
**Inputs:**
- `pos`: Input 3D position (float3).
- `scale`: Frequency multiplier (float).
- `seed`: Integer seed (int).
- `size`: Spot radius/threshold (float).
- `minNum`: Minimum spots per cell (int).
- `maxNum`: Maximum spots per cell (int).
- `jitter`: Random offset scale (float).
- `shape`: Profile shape enum (`SHAPE_STEP`, `SHAPE_LINEAR`, `SHAPE_QUADRATIC`).

**Outputs:**
- `val`: The computed noise value, typically in the range $[-1, 1]$ or $[0, 1]$ depending on the shape function.

**Data Structures:**
- No complex data structures are required. The method relies on a deterministic pseudo-random number generator (PRNG) seeded by cell coordinates and a global seed to generate spot positions and counts.

**Steps:**
1. If `size` < `EPSILON`, return `0.0f`.
2. Compute integer grid coordinates:
   $$ ix = \lfloor pos.x \cdot scale \rfloor $$
   $$ iy = \lfloor pos.y \cdot scale \rfloor $$
   $$ iz = \lfloor pos.z \cdot scale \rfloor $$
3. Compute fractional coordinates:
   $$ u = pos.x - ix $$
   $$ v = pos.y - iy $$
   $$ w = pos.z - iz $$
4. Initialize `val` to `-1.0f`.
5. Loop over `x`, `y`, `z` from `-1` to `1` (inclusive) to traverse the 3x3x3 neighborhood of grid cells.
6. For each neighbor cell $(ix+x, iy+y, iz+z)$:
   a. Calculate the number of spots in the current neighbor cell using a deterministic PRNG seeded by the cell coordinates and global seed. The number of spots is typically clamped between `minNum` and `maxNum`.
   b. For each spot `i` in `numSpots`:
      i. Generate a random position for the spot within the local cell coordinates using the PRNG. Apply jitter if `jitter` > 0.
      ii. Calculate the distance from the query point to the spot in local coordinates:
          $$ distU = u - spotU_i $$
          $$ distV = v - spotV_i $$
          $$ distW = w - spotW_i $$
      iii. Compute the squared distance:
          $$ distSq = distU^2 + distV^2 + distW^2 $$
      iv. Update `val` based on the `shape`:
          - `SHAPE_STEP`: If `distSq` < `size`, set `val` to `1.0f`. Else, `val` remains `-1.0f` (or is updated to the maximum of current `val` and `1.0f` if `distSq` < `size`).
          - `SHAPE_LINEAR`: Use Manhattan distance $D = |distU| + |distV| + |distW|$. Apply linear falloff: $val = \max(val, 1.0 - D / size)$.
          - `SHAPE_QUADRATIC`: Use Euclidean distance squared. Apply quadratic falloff: $val = \max(val, 1.0 - distSq / size^2)$.
7. Return `val`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def random_spots_noise(pos, scale, seed, size, min_num, max_num, jitter, shape):
    if size < 1e-6:
        return 0.0

    ix = math.floor(pos[0] * scale)
    iy = math.floor(pos[1] * scale)
    iz = math.floor(pos[2] * scale)

    u = pos[0] - ix
    v = pos[1] - iy
    w = pos[2] - iz

    val = -1.0

    for x in range(-1, 2):
        for y in range(-1, 2):
            for z in range(-1, 2):
                cell_x = ix + x
                cell_y = iy + y
                cell_z = iz + z

                # Deterministic PRNG seeded by cell coordinates and global seed
                rng = PRNG(seed=seed, cell_x=cell_x, cell_y=cell_y, cell_z=cell_z)
                
                # Determine number of spots in this cell
                num_spots = rng.next_int(min_num, max_num)
                
                for i in range(num_spots):
                    # Generate spot position within local cell
                    spot_u = rng.next_float()
                    spot_v = rng.next_float()
                    spot_w = rng.next_float()
                    
                    # Apply jitter
                    if jitter > 0:
                        spot_u = (spot_u - 0.5) * jitter + 0.5
                        spot_v = (spot_v - 0.5) * jitter + 0.5
                        spot_w = (spot_w - 0.5) * jitter + 0.5
                    
                    # Calculate distance
                    dist_u = u - spot_u
                    dist_v = v - spot_v
                    dist_w = w - spot_w
                    dist_sq = dist_u**2 + dist_v**2 + dist_w**2
                    
                    # Update value based on shape
                    if shape == SHAPE_STEP:
                        if dist_sq < size:
                            val = max(val, 1.0)
                    elif shape == SHAPE_LINEAR:
                        manhattan_dist = abs(dist_u) + abs(dist_v) + abs(dist_w)
                        val = max(val, 1.0 - manhattan_dist / size)
                    elif shape == SHAPE_QUADRATIC:
                        val = max(val, 1.0 - dist_sq / (size**2))
                        
    return val
```

**Parameters:**
- `size`: The radius/scale of the spots. If `size` < `EPSILON`, the function returns `0.0f`. Typical values depend on the desired spot density.
- `shape`: Enum determining the falloff of the spot value. Options: `SHAPE_STEP`, `SHAPE_LINEAR`, `SHAPE_QUADRATIC`.
- `jitter`: Random offset scale. Controls the randomness of spot positions within the cell.
- `minNum`, `maxNum`: Control the density of spots per cell.

**Edge Cases and Pitfalls:**
- The function requires traversing the entire 3x3x3 neighborhood to correctly handle spots near cell edges [S1, unofficial].
- Precision issues may arise if `size` is very small or very large relative to the grid scale.
- Degenerate input (e.g., `size` = 0) is handled by returning `0.0f`.

## Subtopics

### [Worley Cellular Noise](18073-worley-cellular-noise.md)
Worley Cellular Noise is a procedural noise function that calculates the distance to the nearest feature point in a 3D grid, returning a value in $[-1, 1]$. It solves the problem of generating organic, cell-like structures for volume rendering by leveraging the spatial partitioning properties of Voronoi diagrams. Unlike its sibling [Random Spots Noise](18072-random-spots-noise.md), which generates spots within grid cells using deterministic seeds, Worley noise explicitly computes geometric distances to scattered feature points, creating distinct cell boundaries rather than isolated spot intensities.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it always evaluates a fixed 3x3x3 neighborhood (27 cells) and a bounded number of spots per cell.
- **Memory Complexity:** $O(1)$, as no data structures are stored.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of spots per cell, which is bounded by `maxNum`.
- **Strengths:** Simple to implement, fast evaluation, supports various spot shapes.
- **Limitations:** Limited to spot-like patterns, may not capture complex organic structures as well as other noise functions.
- **When to use:** When spot-like patterns are desired, and computational efficiency is important.
- **When not to use:** When complex organic structures or cell-like boundaries are required.

## Background
- **Origin:** Not given in the sources.
- **History and Development:** Not given in the sources.
- **Usage:** Used in procedural texture generation and volume rendering for creating spot-like patterns.

## Sources
[S1] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
