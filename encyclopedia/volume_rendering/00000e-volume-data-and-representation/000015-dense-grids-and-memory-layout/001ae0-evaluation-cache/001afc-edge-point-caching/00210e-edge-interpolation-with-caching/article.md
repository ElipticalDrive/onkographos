# Edge Interpolation with Caching

Also called: Per-Edge Interpolation Cache

## Overview
Edge Interpolation with Caching optimizes Marching Cubes surface generation by storing the computed intersection point and associated gradient normal for each unique edge shared between adjacent cubes, preventing redundant linear interpolation calculations for triangle corners that share the same edge [S2, unofficial]. It builds on [Edge Point Caching](891-edge-point-caching.md) by extending the cache to include geometric normals derived from the scalar field gradient, ensuring consistent shading across shared edges [S1, S2, unofficial].

## How it works
**Inputs:**
- Scalar field values at grid vertices.
- Isosurface threshold value ($I$).
- Boolean flag `bSmooth` indicating whether to use linear interpolation based on scalar values or simple geometric midpoints [S1, unofficial].
- A cache data structure mapping unique edge identifiers to computed results [S1, S2, unofficial].

**Outputs:**
- Intersection position on the edge.
- Gradient normal at the intersection point (if `bSmooth` is true) [S2, unofficial].

**Data Structures:**
The cache uses a key-value mapping. The key is an `FEdgeKey` structure defined as:
- `startPos`: Start position of the edge (e.g., `FIntVector` or 3D integer coordinates) [S1, unofficial].
- `endPos`: End position of the edge (e.g., `FIntVector` or 3D integer coordinates) [S1, unofficial].
- `position`: The computed intersection point (e.g., `FVector` or 3D float coordinates) [S1, unofficial].
- `normal`: The computed gradient normal at the intersection point (implied by [S2, unofficial], stored alongside position) [S2, unofficial].

**Steps:**
1. **Identify Unique Edges:** For the current voxel cell, identify all edges that intersect the isosurface [S2, unofficial].
2. **Generate Key:** For each intersecting edge, create an `FEdgeKey` using the integer coordinates of the start and end vertices [S1, unofficial].
3. **Cache Lookup:** Check if the `FEdgeKey` exists in the cache.
   - If present, retrieve the cached `position` and `normal` [S2, unofficial].
   - If absent, proceed to Step 4.
4. **Compute Intersection:**
   - Let $S$ be the scalar value at the start position, and $E$ be the scalar value at the end position [S1, unofficial].
   - Let $P_s$ be the start position vector, and $P_e$ be the end position vector [S1, unofficial].
   - Calculate the difference $\Delta = I - S$ [S1, unofficial].
   - If $|\Delta| < 10^{-6}$, the intersection is at the start position. Set result $P = P_s$ [S1, unofficial].
   - Else, calculate the interpolation factor $t = \frac{\Delta}{E - S}$ [S1, unofficial].
   - Handle boundary conditions for $t$:
     - If $t < 10^{-6}$, set $P = P_s$ [S1, unofficial].
     - If $(1 - t) < 10^{-6}$, set $P = P_e$ [S1, unofficial].
     - Otherwise, calculate $P = P_s + (P_e - P_s) \times t$ [S1, unofficial].
   - If `bSmooth` is false, $P$ is simply the midpoint $(P_s + P_e) \times 0.5$ [S1, unofficial].
   - Compute the gradient normal $N$ at $P$ using the scalar field gradients at the edge endpoints (method for normal computation not explicitly detailed in sources, but implied by "gradient normal" in [S2, unofficial]) [S2, unofficial].
5. **Store and Return:** Store the computed $P$ and $N$ in the cache under the `FEdgeKey` and return them [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_edge_intersection(edge_start_pos, edge_end_pos, scalar_start, scalar_end, isovalue, b_smooth, cache):
    # Create key from integer coordinates of vertices
    key = FEdgeKey(start=edge_start_pos, end=edge_end_pos)
    
    # Check cache
    if key in cache:
        return cache[key].position, cache[key].normal
    
    # Compute intersection
    if not b_smooth:
        # Simple midpoint
        t = 0.5
        pos = (edge_start_pos + edge_end_pos) * 0.5
        # Normal computation for non-smooth case not specified in sources [G]
        normal = compute_normal(edge_start_pos, edge_end_pos, scalar_start, scalar_end, isovalue)
    else:
        delta = isovalue - scalar_start
        if abs(delta) < 1e-6:
            pos = edge_start_pos
        else:
            denom = scalar_end - scalar_start
            if abs(denom) < 1e-12:
                # Avoid division by zero; treat as midpoint or start
                t = 0.5
                pos = (edge_start_pos + edge_end_pos) * 0.5
            else:
                t = delta / denom
                if t < 1e-6:
                    pos = edge_start_pos
                elif (1 - t) < 1e-6:
                    pos = edge_end_pos
                else:
                    pos = edge_start_pos + (edge_end_pos - edge_start_pos) * t
        
        # Compute normal based on scalar field gradient [S2, unofficial]
        normal = compute_gradient_normal(edge_start_pos, edge_end_pos, scalar_start, scalar_end, isovalue)
    
    # Store in cache
    cache[key] = CacheEntry(position=pos, normal=normal)
    
    return pos, normal
```

**Parameters:**
- `b_smooth`: Boolean flag. `true` for linear interpolation based on scalar values, `false` for geometric midpoints [S1, unofficial].
- `1e-6`: Epsilon threshold for numerical stability checks [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero if scalar values at both ends of an edge are identical. Handled by checking denominator magnitude [G].
- Numerical precision issues when $t$ is very close to 0 or 1. Handled by explicit checks against $10^{-6}$ [S1, unofficial].
- Out-of-range lattice samples treated as density 0 to cap the isosurface against walls [S2, unofficial].

## Subtopics

### Per-Edge Interpolation Cache
A specific implementation variant where the cache stores both the interpolated position and the gradient normal for each unique cut edge, ensuring that all triangle corners sharing that edge reuse the same geometric data, which improves shading consistency and reduces computation [S2, unofficial]. It is chosen when smooth shading is required and memory allows storing normals per edge [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces interpolation and normal computation from $O(E \times C)$ to $O(E)$ per cell, where $E$ is the number of edges and $C$ is the number of corners sharing edges, assuming cache hits [S2, unofficial].
- **Memory Complexity:** Requires additional memory to store the cache entries (position and normal) for each unique edge across the volume [S2, unofficial].
- **Strengths:** Avoids redundant calculations for shared edges; ensures consistent normals for smooth shading [S2, unofficial].
- **Limitations:** Requires cache management overhead; memory usage increases with volume size and resolution [S2, unofficial].
- **When to Use:** When smooth shading is desired and computational cost of interpolation/normal calculation is significant [S2, unofficial].
- **When Not to Use:** When memory is constrained or if flat shading is acceptable (midpoint calculation is cheaper) [S1, S2, unofficial].

## Background
Originated in implementations of Marching Cubes for voxel-based rendering, specifically in projects like VoxelHelper and WebGPU-based renderers [S1, S2, unofficial]. The technique is used to optimize mesh generation by leveraging the redundancy of edge intersections in dense grids [S1, S2, unofficial].

## Sources
[S1] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S2] Renderer: marching-cubes isosurface mesh (r=mesh, WebGPU), https://github.com/nikolai3d/waterfall/pull/17, fetched 2026-10-27 (unofficial)
