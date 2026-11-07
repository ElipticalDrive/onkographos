# Compiled Per-Ray Triangle Grid Walk

Also called: Numba grid walk, compiled triangle-grid walk

## Overview
A ray traversal method that walks a triangle grid using a compiled loop (Numba) with one iteration per ray, replacing the previous array-based approach that used whole-array numpy passes over every ray still in flight [S1, unofficial]. Unlike the array walk, this method forms no (ray, triangle) pair arrays, so `blocks` takes no `work_limit` parameter [S1, unofficial]. It builds on [Amanatides–Woo grid traversal](5003-amanatides-woo-grid-traversal.md) by compiling the per-ray traversal logic into a tight loop, eliminating the overhead of managing global ray state arrays between voxel steps [S1, unofficial].

## How it works
**Inputs:**
- Ray origins and directions.
- Triangle grid definition (vertices, faces).
- Grid bounds and resolution.

**Outputs:**
- For each ray: the first triangle hit (or null if no hit), or the traversal end point.
- Visited voxel sequence (implicit in the walk).

**Data Structures:**
- No global (ray, triangle) pair arrays are formed [S1, unofficial].
- Per-ray state: current voxel coordinates, step sizes, and traversal parameters (axis permutation, shear) [S1, unofficial].
- Triangle grid data: stored in a format accessible to the Numba JIT (e.g., flat arrays for vertices and indices) [S1, unofficial].

**Steps:**
1. **Initialization:** Compile the traversal loop using `numba.njit(parallel=True)` [S1, unofficial].
2. **Ray Iteration:** Iterate over rays using `prange` such that each ray is processed in its own iteration [S1, unofficial].
3. **Per-Ray Setup:** For each ray, form the axis permutation and shear parameters once per ray, not per triangle [S1, unofficial]. This involves determining the primary axis of traversal and computing the step sizes and initial t-values for crossing voxel boundaries, analogous to the Amanatides–Woo algorithm [S1, unofficial].
4. **Voxel Walk:** Walk the ray through the voxels of the triangle grid to its first hit or its end, visiting the same voxels in the same order as the array walk [S1, unofficial].
5. **Triangle Test:** For each voxel visited, test for intersection with triangles within that voxel. Use the same watertight test (Woop, Benthin & Wald), including the averaged, exactly antisymmetric edge function, implemented as `grid_walk._cuts` [S1, unofficial].
6. **Tie Rule and Step Cap:** Apply the same tie rule and step cap as the original array walk [S1, unofficial].
7. **Termination:** Stop when a hit is found or the ray exits the grid bounds [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numba
import numpy as np

# Assume grid data is passed as flat numpy arrays:
# vertices: (N, 3) float32
# faces: (M, 3) int32
# grid_bounds: (6) float32 (min_x, min_y, min_z, max_x, max_y, max_z)
# grid_resolution: (3) int32 (nx, ny, nz)

@numba.njit(parallel=True)
def compiled_per_ray_walk(ray_origins, ray_directions, vertices, faces, 
                          grid_bounds, grid_resolution):
    """
    Walks a triangle grid for each ray using a compiled loop.
    
    Args:
        ray_origins: (R, 3) float32 array of ray origins.
        ray_directions: (R, 3) float32 array of ray directions.
        vertices: (N, 3) float32 array of triangle vertices.
        faces: (M, 3) int32 array of triangle face indices.
        grid_bounds: (6) float32 array defining the grid AABB.
        grid_resolution: (3) int32 array defining the grid dimensions.
        
    Returns:
        hits: (R,) int32 array of hit triangle indices (-1 if no hit).
        hit_t: (R,) float32 array of hit distances.
    """
    R = ray_origins.shape[0]
    hits = np.full(R, -1, dtype=np.int32)
    hit_t = np.full(R, np.inf, dtype=np.float32)
    
    # Precompute grid step sizes and inverse directions
    # This is done per-ray in the loop to match the source description
    # "form the axis permutation and shear parameters once per ray"
    
    for r in numba.prange(R):
        origin = ray_origins[r]
        direction = ray_directions[r]
        
        # Check if ray is inside the grid bounds initially
        # If not, handle appropriately (e.g., no hit or start from entry point)
        # [G] Implementation detail: Determine entry voxel and initial t-values
        
        # Initialize Amanatides-Woo-like traversal state
        # current_voxel: (3) int32
        # step: (3) int32
        # t_max: (3) float32 (distance to next voxel boundary)
        # t_delta: (3) float32 (distance between voxel boundaries)
        
        # Determine primary axis and permutation
        # [G] Standard Amanatides-Woo logic: find max component of direction,
        # determine traversal order of axes
        
        # Compute initial voxel coordinates and t_max values
        # [G] Calculate which voxel the ray starts in and the distance to the
        # first boundary crossing in each axis
        
        # Walk loop
        while True:
            # Check if current voxel is within grid bounds
            if not is_inside_grid(current_voxel, grid_resolution):
                break
            
            # Test triangles in the current voxel
            # [G] Retrieve triangles associated with the current voxel
            # For each triangle, perform watertight intersection test
            # using grid_walk._cuts logic
            
            # Watertight intersection test (Woop, Benthin & Wald)
            # This involves checking if the ray intersects the triangle
            # using the averaged, exactly antisymmetric edge function
            # [G] Implement the specific edge function and intersection logic
            # as defined in grid_walk._cuts
            
            # If a hit is found and is closer than the current best hit_t[r]
            # update hits[r] and hit_t[r]
            
            # Advance to next voxel
            # Determine which axis to step based on t_max
            # Update current_voxel, t_max, and t_delta accordingly
            
            # Apply tie rule and step cap
            # [G] Handle cases where t_max values are equal (tie rule)
            # [G] Ensure step cap is respected to prevent infinite loops
            
            # If t_max exceeds ray length or some other termination condition
            if hit_t[r] < np.inf:
                break # Hit found, no need to continue walking for this ray
        
    return hits, hit_t

def is_inside_grid(voxel, resolution):
    """Check if voxel coordinates are within grid bounds."""
    return (0 <= voxel[0] < resolution[0] and
            0 <= voxel[1] < resolution[1] and
            0 <= voxel[2] < resolution[2])
```

**Parameters:**
- Numba version: `numba>=0.61` (required for Python 3.13 wheels) [S1, unofficial].
- Thread control: `NUMBA_NUM_THREADS=1` is set to prevent xdist workers from starting a machine-sized pool [S1, unofficial].

**Edge Cases and Pitfalls:**
- The intersection test exists twice: `grid_walk._cuts` for the Numba loop and `_watertight_hit` + `_counts_as_hit` for the traced dense path, because a Numba loop cannot call a traced function. They must be kept equal by contract tests [S1, unofficial].
- The first call of a process compiles the loop, taking approximately 2 seconds [S1, unofficial].
- The inclusive far end: a segment ending in a triangle's plane is blocked [S1, unofficial].
- The exclusive near margin: a hit exactly at the margin doesn't count (axis-aligned) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(R * V * T) in the worst case, where R is the number of rays, V is the average number of voxels traversed per ray, and T is the average number of triangles per voxel. However, the constant factor is significantly reduced due to compiled per-ray execution [S1, unofficial].
- **Memory Complexity:** O(R + N + M), where R is the number of rays, N is the number of vertices, and M is the number of triangles. No additional (ray, triangle) pair arrays are formed [S1, unofficial].
- **Measured Performance:**
    - On a 4-core Linux container CPU, RayCastOcclusion on a 4,992-facet annular reactor: 54.47 s (main) vs 2.49 s (PR), a 21.9x speed-up [S1, unofficial].
    - TriangleBody wall (51,328 triangles, 18.4M pairs): 211.9 s (main) vs 15.4 s (PR), a 13.8x speed-up [S1, unofficial].
    - TriangleBody wall with default ShaftCulling(): 43.8 s (main) vs 2.3 s (PR), a 19.0x speed-up [S1, unofficial].
    - Walk alone measured against array walk on 200,000 rays: 16.9-37.3x speed-up across two scenes and three resolutions [S1, unofficial].
- **Strengths:** Significant speed-up over array-based walks, especially for large numbers of rays and complex grids. Eliminates the overhead of managing global ray state arrays.
- **Limitations:** Initial compilation time (~2 seconds). Requires Numba as a dependency. The intersection test must be duplicated for Numba and traced paths, increasing maintenance burden [S1, unofficial].
- **When to Use:** When performance is critical and the number of rays is large. Suitable for production ray tracers where per-ray execution is preferred.
- **When Not to Use:** When memory is extremely constrained (though memory usage is similar to array walk, the lack of pair arrays might be beneficial). When Numba is not available or acceptable as a dependency.

## Background
- **Origin:** Introduced by ctdegroot in Pull Request #571 to the DeGrootResearchGroup/aquaflux repository, dated Sep 26, 2026 [S1, unofficial].
- **History:** The method was developed to address the performance bottleneck of the previous array-based walk, which spent significant time on bookkeeping per ray per voxel step. The inspiration came from observing how production ray tracers like Embree achieve speed through compiled per-ray traversal [S1, unofficial].
- **Usage:** Used in the aquaflux repository for radiation simulations, specifically for computing ray masks and triangle body interactions [S1, unofficial].

## Sources
[S1] Radiation: walk the triangle grid in one compiled loop per ray (Numba), 14-22x on the mask builds, https://github.com/DeGrootResearchGroup/aquaflux/pull/571, fetched 2026-10-25 (unofficial)
