# Euclidean Distance Voxel SDF

Also called: 26-neighbor SDF, Exact voxel SDF

## Overview
The Euclidean Distance Voxel SDF method computes a proper signed distance field on a voxel grid by calculating the exact Euclidean distance from each voxel center to the nearest surface boundary, rather than relying on binary occupancy or truncated approximations [S1, unofficial]. It sets itself apart from simpler binary voxelization by providing continuous gradient information necessary for high-quality ray marching and normal estimation, though it incurs significantly higher computational cost per voxel [S1, unofficial].

## How it works
**Inputs:** A voxel grid where each voxel is marked as either occupied (surface or inside) or empty (outside).
**Outputs:** A scalar field where each voxel stores the signed Euclidean distance to the nearest surface point.

**Data Structures:**
*   `Grid`: A 3D array of voxels.
*   `Voxel`: Contains a boolean flag `is_occupied` (true if inside/on surface, false if empty) and a float `distance`.

**Steps:**
1.  Iterate through every voxel $v$ in the grid.
2.  Identify the 26 immediate neighbors of $v$ (forming a $3 \times 3 \times 3$ block centered at $v$).
3.  If the current voxel $v$ is **occupied** (inside the surface):
    *   Find the nearest neighbor $n$ among the 26 surrounding voxels that is **empty**.
    *   Calculate the Euclidean distance $d$ between the center of $v$ and the center of $n$.
    *   Set $v.distance = -d$.
    *   *Edge Case:* If all 26 neighbors are also occupied, the distance is not defined by immediate neighbors. The source suggests returning a default value (e.g., 1.0) or handling via a different topology, but strictly speaking, the distance to the nearest *empty* voxel is undefined within the local 26-neighborhood [S1, unofficial].
4.  If the current voxel $v$ is **empty** (outside the surface):
    *   Find the nearest neighbor $n$ among the 26 surrounding voxels that is **occupied**.
    *   Calculate the Euclidean distance $d$ between the center of $v$ and the center of $n$.
    *   Set $v.distance = d$.
    *   *Edge Case:* If all 26 neighbors are also empty, return a default distance value (e.g., 1.0) [S1, unofficial].

**Formula:**
For a voxel at grid coordinates $(x, y, z)$ with unit spacing, the Euclidean distance to a neighbor at $(x', y', z')$ is:
$$ d = \sqrt{(x-x')^2 + (y-y')^2 + (z-z')^2} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_euclidean_sdf(grid):
    # grid is a 3D array of booleans: True for occupied, False for empty
    # Dimensions: W x H x D
    W, H, D = grid.shape
    
    for x in range(W):
        for y in range(H):
            for z in range(D):
                current_occupied = grid[x, y, z]
                
                min_dist = float('inf')
                found_neighbor = False
                
                # Iterate over 26 neighbors
                for dx in [-1, 0, 1]:
                    for dy in [-1, 0, 1]:
                        for dz in [-1, 0, 1]:
                            # Skip the voxel itself
                            if dx == 0 and dy == 0 and dz == 0:
                                continue
                            
                            nx, ny, nz = x + dx, y + dy, z + dz
                            
                            # Boundary check
                            if 0 <= nx < W and 0 <= ny < H and 0 <= nz < D:
                                neighbor_occupied = grid[nx, ny, nz]
                                
                                # Check if this neighbor is the target type
                                # If current is occupied, we want empty neighbor
                                # If current is empty, we want occupied neighbor
                                target_is_empty = not current_occupied
                                
                                if neighbor_occupied == target_is_empty:
                                    # Calculate Euclidean distance
                                    # Assuming unit voxel size, distance is sqrt(dx^2 + dy^2 + dz^2)
                                    dist_sq = dx*dx + dy*dy + dz*dz
                                    dist = sqrt(dist_sq)
                                    
                                    if dist < min_dist:
                                        min_dist = dist
                                        found_neighbor = True
                
                # Assign signed distance
                if found_neighbor:
                    if current_occupied:
                        grid[x, y, z] = -min_dist # Negative for inside
                    else:
                        grid[x, y, z] = min_dist  # Positive for outside
                else:
                    # All 27 voxels (self + 26 neighbors) are of the same type
                    # Source suggests returning a default like 1.0 [S1, unofficial]
                    grid[x, y, z] = 1.0 if not current_occupied else -1.0
```

## Subtopics

### [One-shot Euclidean Signed Distance Field (ESDF)](23515-one-shot-euclidean-signed-distance-field-esdf.md)
Computes the field from scratch using a monotone-min Jacobi sweep, ensuring global consistency but with higher latency.

## Operating characteristics
*   **Complexity:** $O(N)$ where $N$ is the number of voxels, but with a high constant factor. Each voxel requires sampling up to 26 neighbors and computing square roots [S1, unofficial].
*   **Performance:** Described as "expensive" due to the per-voxel neighbor search and distance calculations [S1, unofficial].
*   **Strengths:** Provides accurate, continuous signed distances suitable for high-quality rendering and normal estimation.
*   **Limitations:** Computationally intensive; does not scale well to very high resolutions without optimization (e.g., look-up tables or GPU parallelization). Fails to provide meaningful distance for large homogeneous regions (all occupied or all empty) without fallback values.

## Background
This method addresses the limitation of binary voxelization where the "signed distance function" is neither signed nor a true distance [S1, unofficial]. It is often referenced in the context of ray-marching voxel grids to achieve proper normals and smooth shading, as discussed in community forums and blogs such as Inigo Quiles' blog and Computer Graphics Stack Exchange [S1, unofficial].

## Sources
[S1] Ray-Marching: Rendering a Voxel Grid with Proper Normals, https://computergraphics.stackexchange.com/questions/12143/ray-marching-rendering-a-voxel-grid-with-proper-normals, fetched 2026-10-25 (unofficial)
