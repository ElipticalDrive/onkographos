# Worley Cellular Noise

Also called: Cellular Noise, Voronoi Noise

## Overview
Worley Cellular Noise is a procedural noise function that calculates the distance to the nearest feature point in a 3D grid, returning a value in $[-1, 1]$ [S1, unofficial]. It solves the problem of generating organic, cell-like structures for volume rendering by leveraging the spatial partitioning properties of Voronoi diagrams [S1, unofficial]. Unlike its sibling [Random Spots Noise](18072-random-spots-noise.md), which generates spots within grid cells using deterministic seeds, Worley noise explicitly computes geometric distances to scattered feature points, creating distinct cell boundaries rather than isolated spot intensities [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: Query point $(x, y, z)$, feature size `size`.
- Output: Scalar value $v \in [-1, 1]$.

**Data Structures**
- No persistent data structures are required; the function is stateless and computes values on the fly [S1, unofficial].

**Parameters**
- `size`: Feature size. If `size` < EPSILON, the function returns $0.0$ [S1, unofficial].

**Algorithm Steps**
1.  **Grid Decomposition**: Compute the integer grid coordinates $(i_x, i_y, i_z)$ and the fractional coordinates $(u, v, w)$ of the query point. The integer part identifies the current cell, while the fractional part represents the offset within that cell [S1, unofficial].
2.  **Initialization**: Initialize `minDist` to a large value, specifically $1000000.0$ [S1, unofficial].
3.  **Neighborhood Search**: Iterate through the 3x3x3 neighborhood of grid cells surrounding the query point. The loop indices $dx, dy, dz$ range from $-1$ to $1$ [S1, unofficial].
4.  **Feature Point Generation**: For each neighbor cell $(i_x+dx, i_y+dy, i_z+dz)$, determine the number of feature points contained within that cell using a deterministic random seed derived from the cell coordinates [S1, unofficial].
5.  **Distance Calculation**: For each feature point in the current neighbor cell:
    - Calculate the local coordinates of the feature point relative to the query point.
    - Compute the squared Euclidean distance from the query point to the feature point [S1, unofficial].
6.  **Minimum Distance Update**: If the calculated squared distance is smaller than `minDist`, update `minDist` with this new smaller distance [S1, unofficial].
7.  **Normalization**: After checking all neighbors, normalize the result to the range $[-1, 1]$ using the formula:
    $$ v = \text{saturate}(\sqrt{\text{minDist}}) \times 2.0 - 1.0 $$
    where $\text{saturate}(x)$ clamps $x$ to $[0, 1]$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def worley_cellular_noise(x, y, z, size):
    # Parameter check
    EPSILON = 1e-6
    if size < EPSILON:
        return 0.0

    # 1. Grid decomposition
    ix = math.floor(x)
    iy = math.floor(y)
    iz = math.floor(z)
    u = x - ix
    v = y - iy
    w = z - iz

    # 2. Initialization
    min_dist_sq = 1000000.0

    # 3. Neighborhood search (-1 to 1)
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            for dz in range(-1, 2):
                # 4. Deterministic seed for feature points in cell (ix+dx, iy+dy, iz+dz)
                # Note: Specific hash function for deterministic random seed is not provided in sources [G]
                seed = hash_cell(ix + dx, iy + dy, iz + dz)
                
                # Determine number of points in this cell [G]
                num_points = get_num_points(seed)
                
                for p in range(num_points):
                    # Generate feature point position in local cell coordinates [G]
                    # Note: Specific random number generation from seed is not provided in sources [G]
                    fx, fy, fz = get_feature_point(seed, p)
                    
                    # Adjust for neighbor offset
                    fx += (ix + dx)
                    fy += (iy + dy)
                    fz += (iz + dz)
                    
                    # 5. Squared distance calculation
                    dist_sq = (x - fx)**2 + (y - fy)**2 + (z - fz)**2
                    
                    # 6. Update minimum
                    if dist_sq < min_dist_sq:
                        min_dist_sq = dist_sq

    # 7. Normalization
    min_dist = math.sqrt(min_dist_sq)
    # Saturate to [0, 1]
    if min_dist > 1.0:
        min_dist = 1.0
    if min_dist < 0.0:
        min_dist = 0.0
        
    result = min_dist * 2.0 - 1.0
    return result

def hash_cell(ix, iy, iz):
    # Placeholder for deterministic hashing logic [G]
    return ix * 73856093 ^ iy * 19349663 ^ iz * 83492791

def get_num_points(seed):
    # Placeholder for determining point count [G]
    return 1

def get_feature_point(seed, index):
    # Placeholder for generating point coordinates [G]
    return 0.5, 0.5, 0.5
```

**Edge Cases and Pitfalls**
- **Precision**: The use of squared distances avoids square roots during the search, improving performance, but the final normalization requires a square root [S1, unofficial].
- **Boundaries**: The 3x3x3 neighborhood ensures that the nearest feature point is found even if it lies in an adjacent cell [S1, unofficial].
- **Degenerate Input**: If `size` is less than EPSILON, the function returns $0.0$ to avoid division by zero or undefined behavior [S1, unofficial].

## Subtopics

### [Periodic Cellular Structure Replication](19927-periodic-cellular-structure-replication.md)
A method for generating infinite cellular microstructures by applying a periodic space mapping function to a base unit cell. It differs from standard Worley noise by enabling seamless tiling of infinite domains without explicit geometric duplication, suitable for modeling regular lattices.

## Operating characteristics
- **Time Complexity**: $O(1)$ per query, as it always checks a fixed 3x3x3 neighborhood (27 cells) and a small, fixed number of feature points per cell [S1, unofficial].
- **Memory Complexity**: $O(1)$, as no data structures are stored [S1, unofficial].
- **Performance**: The method is computationally efficient due to the fixed neighborhood size and lack of global data structures [S1, unofficial].
- **Strengths**: Fast evaluation, stateless, generates organic cell-like patterns [S1, unofficial].
- **Limitations**: The output range is limited to $[-1, 1]$, and the specific distribution of feature points depends on the deterministic hashing implementation [S1, unofficial].

## Background
Worley Cellular Noise is a procedural noise function widely used in computer graphics for generating textures and volumes that resemble cellular structures [S1, unofficial]. The method is implemented in various open-source projects, such as `voxelized3d`, which provides a practical reference for its implementation [S1, unofficial].

## Sources
[S1] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
