# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering by eliminating unnecessary evaluations of empty space or non-contributing geometry. It achieves this by adapting the spatial domain of the voxel array to the actual extent of the data or by culling Signed Distance Field (SDF) primitives using bounding volumes [S1, unofficial][S2]. Unlike sibling methods that focus on data compression or smoothing, this method specifically targets traversal efficiency by restricting the active rendering domain to tight bounds, thereby reducing false positives and memory bandwidth usage [S1, unofficial].

## How it works
The method operates in two distinct modes: spatial domain adaptation for voxel grids and primitive culling for SDF raymarching. Both rely on the principle that computations should only occur where geometry exists.

### Inputs and Outputs
*   **Inputs:**
    *   For Voxel Adaptation: A dense 3D voxel grid defined by resolution $(N_x, N_y, N_z)$ and a spatial domain, along with the binary or density values of the voxels [S1, unofficial].
    *   For SDF Culling: A set of geometric primitives, each with an associated SDF evaluation function and a bounding volume (typically an Axis-Aligned Bounding Box, AABB) [S2].
*   **Outputs:**
    *   A reduced voxel array dimensions $(N'_x, N'_y, N'_z)$ and an offset $(O_x, O_y, O_z)$ defining the new origin [S1, unofficial].
    *   A culling mask or early-out condition for the raymarching loop that skips SDF evaluations for primitives outside the current ray's bounding volume [S2].

### Data Structures
1.  **Adapted Voxel Grid:** A 3D array with dimensions equal to the tight bounding box of the occupied voxels. The layout is a standard contiguous memory block or texture, but with reduced size compared to the original fixed grid [S1, unofficial].
2.  **Bounding Volume List:** An array of AABBs, where each AABB is defined by two 3D vectors: `min` and `max` coordinates in world or local space [S2].

### Algorithm Steps

#### Mode 1: Voxel Array Dimension Adaptation
This mode eliminates empty space at the boundaries of a fixed-resolution grid.

1.  **Scan for Extents:** Iterate through the original voxel grid $(N_x, N_y, N_z)$ to find the minimum and maximum indices $(i_{min}, j_{min}, k_{min})$ and $(i_{max}, j_{max}, k_{max})$ that contain non-empty (or relevant) data [S1, unofficial].
2.  **Calculate New Dimensions:** Compute the tight dimensions:
    $$ N'_x = i_{max} - i_{min} + 1 $$
    $$ N'_y = j_{max} - j_{min} + 1 $$
    $$ N'_z = k_{max} - k_{min} + 1 $$
3.  **Compute Offset:** The new origin offset is $(O_x, O_y, O_z) = (i_{min}, j_{min}, k_{min})$ [S1, unofficial].
4.  **Reallocate/Map:** Create or map a new texture/array of size $(N'_x, N'_y, N'_z)$. When accessing the grid during rendering, map the ray's voxel coordinates $(u, v, w)$ to the new array using:
    $$ u' = u - O_x, \quad v' = v - O_y, \quad w' = w - O_z $$
    If $(u', v', w')$ is outside $[0, N'_x-1] \times [0, N'_y-1] \times [0, N'_z-1]$, the ray is outside the volume [S1, unofficial].

#### Mode 2: SDF Bounding Volume Culling
This mode skips expensive SDF evaluations for primitives that do not influence the minimal distance.

1.  **Precompute Bounds:** For each primitive $P_i$ with SDF function $SDF_i(\mathbf{x})$, compute its AABB $B_i = [\mathbf{b}_{i,min}, \mathbf{b}_{i,max}]$ [S2].
2.  **Ray-BV Intersection:** During raymarching, for each step or at the start of the trace, test the ray against the bounding volumes of all primitives [S2].
3.  **Culling Decision:** If the ray does not intersect $B_i$, skip the evaluation of $SDF_i(\mathbf{x})$ for that primitive [S2].
4.  **Evaluation:** Only evaluate $SDF_i(\mathbf{x})$ for primitives where the ray intersects $B_i$. The final distance is the minimum of the evaluated SDFs [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Mode 1: Voxel Array Adaptation
def compute_voxel_bounds(voxel_grid, Nx, Ny, Nz):
    """
    Finds the tight bounding box of non-empty voxels.
    Returns (min_idx, max_idx, new_dims, offset).
    """
    min_idx = [Nx, Ny, Nz]
    max_idx = [-1, -1, -1]
    
    # Scan grid to find extents [S1, unofficial]
    for x in range(Nx):
        for y in range(Ny):
            for z in range(Nz):
                if voxel_grid[x, y, z] is not empty:
                    min_idx[0] = min(min_idx[0], x)
                    max_idx[0] = max(max_idx[0], x)
                    min_idx[1] = min(min_idx[1], y)
                    max_idx[1] = max(max_idx[1], y)
                    min_idx[2] = min(min_idx[2], z)
                    max_idx[2] = max(max_idx[2], z)
    
    # Calculate new dimensions and offset [S1, unofficial]
    new_dims = [
        max_idx[0] - min_idx[0] + 1,
        max_idx[1] - min_idx[1] + 1,
        max_idx[2] - min_idx[2] + 1
    ]
    offset = min_idx
    
    return new_dims, offset

def raycast_adapted_voxel(ray_origin, ray_dir, voxel_grid, new_dims, offset, Nx_orig, Ny_orig, Nz_orig):
    """
    Raycasts against the adapted voxel grid.
    """
    # Map ray to voxel space of original grid
    # ... (standard ray-box intersection for original grid bounds) ...
    
    for t in ray_march_steps:
        pos = ray_origin + t * ray_dir
        voxel_coords = world_to_voxel(pos)
        
        # Apply offset to map to adapted grid coordinates [S1, unofficial]
        u = voxel_coords.x - offset.x
        v = voxel_coords.y - offset.y
        w = voxel_coords.z - offset.z
        
        # Check bounds of adapted grid
        if not (0 <= u < new_dims[0] and 
                0 <= v < new_dims[1] and 
                0 <= w < new_dims[2]):
            continue # Outside adapted volume
            
        density = voxel_grid[u, v, w]
        # ... accumulate density ...

# Mode 2: SDF Bounding Volume Culling [S2]
def raymarch_sdf_cull(ray_origin, ray_dir, primitives):
    """
    Raymarches using bounding volume culling for SDF primitives.
    """
    t = 0.0
    max_steps = 1000
    
    for step in range(max_steps):
        pos = ray_origin + t * ray_dir
        
        min_dist = infinity
        hit_primitive = None
        
        for prim in primitives:
            # Test ray against primitive's bounding volume [S2]
            if not ray_intersects_aabb(ray_origin, ray_dir, prim.bbox):
                continue # Skip expensive SDF evaluation
            
            # Evaluate SDF only if BV is hit [S2]
            dist = prim.sdf_function(pos)
            if dist < min_dist:
                min_dist = dist
                hit_primitive = prim
        
        # Advance ray by the minimum distance found
        t += min_dist
        
        if min_dist < epsilon:
            break # Hit surface
            
        if t > max_distance:
            break # Missed everything
            
    return hit_primitive, t
```

### Parameters and Edge Cases
*   **Threshold:** In Mode 1, the definition of "empty" depends on the data type (binary occupancy vs. density threshold). A threshold of 0 is typical for binary masks [S1, unofficial].
*   **Degenerate Input:** If the voxel grid is entirely empty, $N'_x$ etc. become 0 or negative. The implementation must handle this by returning an empty volume or a default size [G].
*   **Precision:** When mapping coordinates, floating-point errors can cause off-by-one errors at boundaries. Using integer arithmetic for index calculation where possible is recommended [G].

## Operating characteristics
*   **Time Complexity:**
    *   Voxel Adaptation: $O(N_x N_y N_z)$ for the initial scan, then $O(1)$ per ray step for bounds checking. The scan is a one-time cost [S1, unofficial].
    *   SDF Culling: $O(P)$ per ray step, where $P$ is the number of primitives, but with a significantly reduced constant factor due to skipped SDF evaluations [S2].
*   **Memory Complexity:**
    *   Voxel Adaptation: Reduces memory usage from $O(N_x N_y N_z)$ to $O(N'_x N'_y N'_z)$. For a 512³ grid with a height of 321, this reduces the volume from $134,217,728$ to $83,886,080$ voxels, saving ~37% memory [S1, unofficial].
*   **Performance:**
    *   In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
    *   Without optimization, computation cost is uniform across the frame; with optimization, expensive computations are localized to areas with complex geometry [S2].
*   **Strengths:**
    *   Eliminates false positives in traversal [S1, unofficial].
    *   Reduces memory bandwidth by loading only necessary data [S1, unofficial].
    *   Allows separation of game world and entities, enabling physics simulations using voxel models as bounding boxes [S1, unofficial].
*   **Limitations:**
    *   AABBs can cause visually harsh lighting artifacts because face normals are axis-aligned, leading to non-physically plausible shading [S1, unofficial].
    *   Requires precomputation of bounds, which adds latency to scene initialization [S2].

## Background
Bounding Volume Optimization originated in the context of raymarching and voxel-based rendering to address the high computational cost of evaluating Signed Distance Fields (SDFs) and traversing large voxel grids. The concept of adapting voxel array sizes to reduce redundancy was discussed in the context of voxel landscapes and rigid body physics simulations [S1, unofficial]. The specific application of bounding volumes to cull SDF primitives in CUDA-based raymarchers was detailed in research on distance field rendering [S2]. Modern GPUs, being MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD, benefit from this optimization by allowing different Streaming Multiprocessors (SMs) to distribute expensive threads after cheaper, culled parts finish early [S2].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
