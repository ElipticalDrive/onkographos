# Point Engine for Scalar Fields
Also called: Simplified Marching Cubes for points, rgridb point generation

## Overview
A method to estimate foot points on an isosurface of a scalar field using a simplified version of Marching Cubes that finds only vertices without computing triangles, avoiding the expense of iterative refinement. It builds on [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md) by extracting initial vertex positions and gradients, which are then stored in a flat list suitable for point-based rendering or further processing like dynamic particle distribution.

## How it works
**Inputs:**
- A scalar field defined on a voxel grid.
- An isovalue threshold.
- The dimensions of the voxel grid.

**Outputs:**
- A single list of points, where each point consists of a 3D position and an interpolated gradient vector.

**Data Structures:**
- `Point`: A structure containing:
  - `position`: `vec3` (float32 x3)
  - `gradient`: `vec3` (float32 x3)
- `PointList`: A dynamic array or contiguous buffer of `Point` structures.

**Steps:**
1. Iterate over every voxel in the volume grid. Let the current voxel's position be $p$.
2. Define the positions of three neighboring voxels:
   - $q = p + (1, 0, 0)$
   - $r = p + (0, 1, 0)$
   - $s = p + (0, 0, 1)$
3. For each neighbor $n \in \{q, r, s\}$:
   a. Check if the edge connecting $p$ and $n$ is inside the volume boundaries. If not, skip this edge.
   b. Sample the scalar field values $V_p$ at $p$ and $V_n$ at $n$.
   c. Determine if the edge intersects the isosurface by checking if $(V_p - \text{iso})$ and $(V_n - \text{iso})$ have different signs (i.e., their product is negative).
   d. If an intersection exists:
      i. Calculate the intersection point $x_{int}$ using linear interpolation (the standard Marching Cubes scheme for edge intersection):
         $$ t = \frac{\text{iso} - V_p}{V_n - V_p} $$
         $$ x_{int} = p + t \cdot (n - p) $$
      ii. Interpolate the gradient at $x_{int}$. The source notes mention interpolating the gradient to that point but does not specify the exact gradient interpolation formula (e.g., trilinear interpolation of voxel gradients or finite difference approximation). [G] Standard practice in such engines is to compute gradients at voxel centers via finite differences and then trilinearly interpolate the gradient vector at $x_{int}$.
      iii. Create a `Point` with `position` = $x_{int}$ and `gradient` = interpolated gradient.
      iv. Append the `Point` to the `PointList`.
4. Return the `PointList`.

**Parameters:**
- `iso`: The isovalue threshold. Typical values depend on the specific scalar field normalization.

**Edge Cases and Pitfalls:**
- **Boundary Checks:** The algorithm must explicitly check if neighbors $q, r, s$ are within the grid bounds before accessing their scalar values.
- **Shared Vertices:** Unlike full Marching Cubes, this method visits only three edges per cell (positive x, y, z directions). This avoids double-counting shared vertices between adjacent cells, so no bookkeeping or deduplication is required.
- **Precision:** Linear interpolation assumes the isosurface is locally planar within the voxel. For highly curved surfaces, this may introduce small positional errors compared to iterative refinement.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_engine_for_scalar_fields(grid, iso, grid_dims):
    points = []
    
    # Helper to get scalar value with boundary check
    def get_scalar(pos):
        if 0 <= pos.x < grid_dims.x and 0 <= pos.y < grid_dims.y and 0 <= pos.z < grid_dims.z:
            return grid[pos.x][pos.y][pos.z]
        return None

    # Helper to compute gradient at a voxel center (finite difference)
    def get_gradient(pos):
        # [G] Standard central difference or forward/backward at boundaries
        dx = get_scalar(pos + (1,0,0)) - get_scalar(pos - (1,0,0))
        dy = get_scalar(pos + (0,1,0)) - get_scalar(pos - (0,1,0))
        dz = get_scalar(pos + (0,0,1)) - get_scalar(pos - (0,0,1))
        # Handle boundary conditions for gradient calculation if necessary
        return vec3(dx, dy, dz)

    # Helper to interpolate gradient at arbitrary point
    def interpolate_gradient(pos, grad_p, grad_n):
        # [G] Trilinear interpolation of gradients is typical, 
        # but source only says "interpolate the gradient". 
        # Simple linear interpolation of gradients along the edge is a minimal implementation.
        t = (iso - get_scalar(pos)) / (get_scalar(pos + (1,0,0)) - get_scalar(pos)) # Approximate t for edge
        # Actually, t is calculated per edge. Let's refine the loop structure.
        pass

    for x in range(grid_dims.x):
        for y in range(grid_dims.y):
            for z in range(grid_dims.z):
                p = vec3(x, y, z)
                v_p = get_scalar(p)
                if v_p is None: continue
                
                # Define neighbors
                neighbors = [
                    (vec3(1, 0, 0), vec3(x+1, y, z)),
                    (vec3(0, 1, 0), vec3(x, y+1, z)),
                    (vec3(0, 0, 1), vec3(x, y, z+1))
                ]
                
                for offset, n in neighbors:
                    v_n = get_scalar(n)
                    if v_n is None: continue
                    
                    # Check if edge crosses isosurface
                    if (v_p - iso) * (v_n - iso) < 0:
                        # Linear interpolation for position
                        t = (iso - v_p) / (v_n - v_p)
                        pos_int = p + t * offset
                        
                        # Interpolate gradient
                        # [G] Source does not specify gradient interpolation method.
                        # Assuming linear interpolation of gradients at p and n for simplicity
                        grad_p = get_gradient(p)
                        grad_n = get_gradient(n)
                        grad_int = grad_p + t * (grad_n - grad_p)
                        
                        points.append(Point(pos_int, grad_int))
                        
    return points
```

## Subtopics

### [Dynamic Particle Distribution](9587-dynamic-particle-distribution.md)
Dynamic Particle Distribution is a particle-based sampling method for isosurface mesh generation that uses inter-particle forces to drive particles to local energy minima, controlling density via a sizing field $h$ and correcting for Euclidean distance approximation artifacts using a contraction factor $g$. It builds on [Point Engine for Scalar Fields](7529-point-engine-for-scalar-fields.md) by taking the initial vertex positions and refining them into a uniformly distributed point cloud suitable for high-quality triangulation, rather than outputting a mesh directly. Unlike Marching Cubes variants which produce fixed topological structures, this method adapts particle density to the local curvature and feature size of the implicit surface.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel is processed once, and a constant number of operations (3 edges) are performed per voxel.
- **Memory Complexity:** $O(M)$, where $M$ is the number of surface intersections. In the worst case (checkerboard pattern), $M \approx 3N$.
- **Strengths:** Extremely fast; no table lookups; no vertex sharing bookkeeping; produces points with normals (gradients) directly.
- **Limitations:** Does not produce a mesh; points may be unevenly distributed; no topological connectivity information.
- **When to use:** When point-based rendering is sufficient, or as a preprocessing step for particle-based meshing methods.

## Background
Originated in the context of hardware-accelerated point rendering of isosurfaces, specifically for the rgridb renderer. The method was proposed as a computationally cheaper alternative to iterative foot-point estimation or full Marching Cubes triangle generation, leveraging the simplicity of visiting only three edges per cell.

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
