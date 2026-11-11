# Point Engine for Scalar Fields
Also called: Simplified Marching Cubes for points, rgridb point generation

## Overview
A method to estimate foot points on an isosurface of a scalar field using a simplified version of Marching Cubes that finds only vertices without computing triangles, avoiding the expense of iterative refinement [S1, unofficial]. It builds on [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md) by extracting initial vertex positions and gradients, which are then stored in a flat list suitable for point-based rendering or further processing like dynamic particle distribution [S1, unofficial].

## How it works
**Inputs and outputs:**
- A scalar field defined on a voxel grid [S1, unofficial].
- An isovalue threshold `iso` [S1, unofficial].
- The dimensions of the voxel grid `grid_dims` [S1, unofficial].
- A single list of points, where each point consists of a 3D position and an interpolated gradient vector [S1, unofficial].

**Data structures:**
- `Point`: A structure containing `position` (3D vector, typically `float32` x3) and `gradient` (3D vector, typically `float32` x3) [S1, unofficial].
- `PointList`: A contiguous buffer or dynamic array of `Point` structures [S1, unofficial].
- Bit packing is not specified in the sources [S1, unofficial].

**Numbered steps:**
1. Iterate over every voxel in the volume grid. Let the current voxel's position be $p$ [S1, unofficial].
2. Define the positions of three neighboring voxels: $q = p + (1, 0, 0)$, $r = p + (0, 1, 0)$, and $s = p + (0, 0, 1)$ [S1, unofficial].
3. For each neighbor $n \in \{q, r, s\}$:
   a. Check if the edge connecting $p$ and $n$ is inside the volume boundaries. If not, skip this edge [S1, unofficial].
   b. Sample the scalar field values $V_p$ at $p$ and $V_n$ at $n$ [S1, unofficial].
   c. Determine if the edge intersects the isosurface by checking if $(V_p - \text{iso})$ and $(V_n - \text{iso})$ have different signs (i.e., their product is negative) [S1, unofficial].
   d. If an intersection exists:
      i. Calculate the intersection point $x_{int}$ using linear interpolation (the standard Marching Cubes scheme for edge intersection):
         $$ t = \frac{\text{iso} - V_p}{V_n - V_p} $$
         $$ x_{int} = p + t \cdot (n - p) $$
      ii. Interpolate the gradient at $x_{int}$ [S1, unofficial].
      iii. Create a `Point` with `position` = $x_{int}$ and `gradient` = interpolated gradient [S1, unofficial].
      iv. Append the `Point` to the `PointList` [S1, unofficial].
4. Return the `PointList` [S1, unofficial].

**Formulas:**
- $p$: Current voxel position (3D vector) [S1, unofficial].
- $q, r, s$: Neighbor voxel positions along positive x, y, z axes [S1, unofficial].
- $n$: Generic neighbor position from $\{q, r, s\}$ [S1, unofficial].
- $V_p, V_n$: Scalar field values at $p$ and $n$ [S1, unofficial].
- $\text{iso}$: Isosurface threshold value [S1, unofficial].
- $t$: Interpolation parameter along the edge, $t \in [0, 1]$ [S1, unofficial].
- $x_{int}$: Calculated intersection point on the isosurface [S1, unofficial].

**Parameters:**
- `iso`: The isovalue threshold. Typical values depend on the specific scalar field normalization [S1, unofficial].

**Edge cases and pitfalls:**
- **Boundary Checks:** The algorithm must explicitly check if neighbors $q, r, s$ are within the grid bounds before accessing their scalar values [S1, unofficial].
- **Shared Vertices:** Unlike full Marching Cubes, this method visits only three edges per cell (positive x, y, z directions). This avoids double-counting shared vertices between adjacent cells, so no bookkeeping or deduplication is required [S1, unofficial].
- **Precision:** Linear interpolation assumes the isosurface is locally planar within the voxel. For highly curved surfaces, this may introduce small positional errors compared to iterative refinement [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_engine_for_scalar_fields(grid, iso, grid_dims):
    points = []
    
    def get_scalar(pos):
        if 0 <= pos[0] < grid_dims[0] and 0 <= pos[1] < grid_dims[1] and 0 <= pos[2] < grid_dims[2]:
            return grid[pos[0]][pos[1]][pos[2]]
        return None

    def get_gradient(pos):
        # [G] Standard central difference approximation for voxel center gradients
        gx = (get_scalar((pos[0]+1, pos[1], pos[2])) - get_scalar((pos[0]-1, pos[1], pos[2]))) / 2.0
        gy = (get_scalar((pos[0], pos[1]+1, pos[2])) - get_scalar((pos[0], pos[1]-1, pos[2]))) / 2.0
        gz = (get_scalar((pos[0], pos[1], pos[2]+1)) - get_scalar((pos[0], pos[1], pos[2]-1))) / 2.0
        return (gx, gy, gz)

    for x in range(grid_dims[0]):
        for y in range(grid_dims[1]):
            for z in range(grid_dims[2]):
                p = (x, y, z)
                v_p = get_scalar(p)
                if v_p is None:
                    continue
                
                neighbors = [
                    ((1, 0, 0), (x+1, y, z)),
                    ((0, 1, 0), (x, y+1, z)),
                    ((0, 0, 1), (x, y, z+1))
                ]
                
                for offset, n in neighbors:
                    v_n = get_scalar(n)
                    if v_n is None:
                        continue
                    
                    if (v_p - iso) * (v_n - iso) < 0:
                        t = (iso - v_p) / (v_n - v_p)
                        pos_int = (p[0] + t * offset[0], p[1] + t * offset[1], p[2] + t * offset[2])
                        
                        grad_p = get_gradient(p)
                        grad_n = get_gradient(n)
                        grad_int = (grad_p[0] + t * (grad_n[0] - grad_p[0]),
                                    grad_p[1] + t * (grad_n[1] - grad_p[1]),
                                    grad_p[2] + t * (grad_n[2] - grad_p[2]))
                        
                        points.append((pos_int, grad_int))
                        
    return points
```

## Subtopics

### [Dynamic Particle Distribution](9587-dynamic-particle-distribution.md)
Dynamic Particle Distribution is a particle-based sampling method for isosurface mesh generation that uses inter-particle forces to drive particles to local energy minima, controlling density via a sizing field $h$ and correcting for Euclidean distance approximation artifacts using a contraction factor $g$. It builds on [Point Engine for Scalar Fields](7529-point-engine-for-scalar-fields.md) by taking the initial vertex positions and refining them into a uniformly distributed point cloud suitable for high-quality triangulation, rather than outputting a mesh directly. Unlike Marching Cubes variants which produce fixed topological structures, this method adapts particle density to the local curvature and feature size of the implicit surface.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel is processed once, and a constant number of operations (3 edges) are performed per voxel [S1, unofficial].
- **Memory Complexity:** $O(M)$, where $M$ is the number of surface intersections. In the worst case (checkerboard pattern), $M \approx 3N$ [S1, unofficial].
- **Strengths:** Extremely fast; no table lookups; no vertex sharing bookkeeping; produces points with normals (gradients) directly [S1, unofficial].
- **Limitations:** Does not produce a mesh; points may be unevenly distributed; no topological connectivity information [S1, unofficial].
- **When to use:** When point-based rendering is sufficient, or as a preprocessing step for particle-based meshing methods [S1, unofficial].

## Background
Originated in the context of hardware-accelerated point rendering of isosurfaces, specifically for the rgridb renderer. The method was proposed as a computationally cheaper alternative to iterative foot-point estimation or full Marching Cubes triangle generation, leveraging the simplicity of visiting only three edges per cell [S1, unofficial].

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
