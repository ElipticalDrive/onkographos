# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization minimizes the computational cost of voxel-based rendering by restricting evaluation to the spatial extents of active geometry, thereby eliminating false positives and skipping expensive Signed Distance Field (SDF) evaluations for objects that do not contribute to the final image [S1, unofficial][S2]. It differs from sibling methods like [Grid Resolution Calculation](2908-grid-resolution-calculation.md) by focusing on spatial culling rather than grid density, and from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by addressing runtime traversal efficiency rather than memory footprint.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of geometric primitives (e.g., SDF primitives) and their associated axis-aligned bounding boxes (AABBs); the current camera frustum or ray origin/direction.
*   **Output:** A reduced set of active primitives for the current frame or ray, allowing the renderer to skip evaluation of inactive volumes.

**Data Structures:**
*   **Bounding Volume (BV):** An AABB defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$.
*   **Voxel Array Dimensions:** The dimensions of the 3D texture or grid are adjusted to match the bounding volume of the active data, rather than using a fixed global maximum [S1, unofficial].

**Steps:**
1.  **Compute Bounding Volumes:** For each primitive or object in the scene, compute its AABB. For SDF primitives, this is often a tight box surrounding the primitive's influence [S2].
2.  **Adjust Voxel Array Size:** Determine the global bounding box of all active geometry. Resize the voxel array dimensions $(W, H, D)$ to tightly enclose this volume, eliminating unused regions [S1, unofficial]. For example, if a landscape fits within $512 \times 321 \times 512$ voxels, use these dimensions instead of a fixed $512^3$ cube [S1, unofficial].
3.  **Frustum/Volume Culling:** For each ray or pixel, test if the ray intersects the bounding volume of each primitive. If no intersection, skip the SDF evaluation for that primitive [S2].
4.  **Evaluate SDF:** Only evaluate the distance function for primitives whose bounding volumes are intersected by the ray or fall within the active voxel region [S2].
5.  **Handle GPU Parallelism:** Leverage the GPU's MIMD (Multiple Instruction, Multiple Data) nature, where different Streaming Multiprocessors (SMs) can process different instruction streams. This allows cheaper threads (those skipping evaluations) to finish early, distributing expensive threads across available SMs [S2].

**Formulas:**
*   **AABB Intersection Test:** A ray $R(t) = O + tD$ intersects an AABB if:
    $$ t_{min} = \max \left( \frac{x_{min} - O_x}{D_x}, \frac{y_{min} - O_y}{D_y}, \frac{z_{min} - O_z}{D_z} \right) $$
    $$ t_{max} = \min \left( \frac{x_{max} - O_x}{D_x}, \frac{y_{max} - O_y}{D_y}, \frac{z_{max} - O_z}{D_z} \right) $$
    Intersection occurs if $t_{min} \le t_{max}$ and $t_{max} \ge 0$.
*   **Array Dimension Adjustment:** Let $B_{global}$ be the union of all primitive AABBs. The new voxel dimensions $(W', H', D')$ are:
    $$ W' = \lceil \frac{B_{global}.x_{max} - B_{global}.x_{min}}{voxel\_size} \rceil $$
    (Similarly for $H'$ and $D'$) [S1, unofficial].

**Parameters:**
*   **Voxel Size:** Determines the resolution. Smaller sizes increase fidelity but memory usage [S1, unofficial].
*   **Primitive Count:** The number of primitives affects the culling overhead. In tests, 20 expensive primitives showed significant gains [S2].

**Edge Cases and Pitfalls:**
*   **Fixed Array Sizes:** Using a fixed large array (e.g., $512^3$) for small geometry loads unnecessary data into GPU memory and causes false positives during traversal [S1, unofficial].
*   **Normal Calculation:** AABBs have flat faces, leading to hard, non-physically plausible lighting if normals are derived directly from the box faces [S1, unofficial].
*   **GPU Architecture:** While GPUs are often viewed as SIMD, they are effectively MIMD. This optimization relies on the ability to diverge threads efficiently; if the scene is uniform, culling benefits may be reduced [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_optimization(ray_origin, ray_direction, primitives):
    """
    Optimizes SDF evaluation by culling primitives outside their bounding volumes.
    
    Args:
        ray_origin: (x, y, z) tuple
        ray_direction: (x, y, z) tuple
        primitives: List of objects with .sdf() and .aabb() methods.
                    .aabb() returns (min_x, min_y, min_z, max_x, max_y, max_z)
    
    Returns:
        List of active primitives for this ray.
    """
    active_primitives = []
    
    for prim in primitives:
        min_x, min_y, min_z, max_x, max_y, max_z = prim.aabb()
        
        # Ray-AABB intersection test
        # Handle division by zero for parallel rays
        inv_dir_x = 1.0 / ray_direction[0] if ray_direction[0] != 0 else float('inf')
        inv_dir_y = 1.0 / ray_direction[1] if ray_direction[1] != 0 else float('inf')
        inv_dir_z = 1.0 / ray_direction[2] if ray_direction[2] != 0 else float('inf')
        
        t_min = max(
            (min_x - ray_origin[0]) * inv_dir_x,
            (min_y - ray_origin[1]) * inv_dir_y,
            (min_z - ray_origin[2]) * inv_dir_z
        )
        t_max = min(
            (max_x - ray_origin[0]) * inv_dir_x,
            (max_y - ray_origin[1]) * inv_dir_y,
            (max_z - ray_origin[2]) * inv_dir_z
        )
        
        # Check for intersection
        if t_min <= t_max and t_max >= 0:
            active_primitives.append(prim)
            
    return active_primitives

def adjust_voxel_dimensions(primitives, voxel_size):
    """
    Adjusts voxel array dimensions to fit the bounding volume of primitives.
    
    Args:
        primitives: List of objects with .aabb() method.
        voxel_size: Float, size of one voxel.
        
    Returns:
        (width, height, depth) tuple of voxel counts.
    """
    if not primitives:
        return (0, 0, 0)
        
    global_min_x = float('inf')
    global_min_y = float('inf')
    global_min_z = float('inf')
    global_max_x = float('-inf')
    global_max_y = float('-inf')
    global_max_z = float('-inf')
    
    for prim in primitives:
        min_x, min_y, min_z, max_x, max_y, max_z = prim.aabb()
        global_min_x = min(global_min_x, min_x)
        global_min_y = min(global_min_y, min_y)
        global_min_z = min(global_min_z, min_z)
        global_max_x = max(global_max_x, max_x)
        global_max_y = max(global_max_y, max_y)
        global_max_z = max(global_max_z, max_z)
        
    width = int((global_max_x - global_min_x) / voxel_size) + 1
    height = int((global_max_y - global_min_y) / voxel_size) + 1
    depth = int((global_max_z - global_min_z) / voxel_size) + 1
    
    return (width, height, depth)
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of primitives, but with a significantly reduced constant factor due to culling. In a scene with 20 expensive primitives at 720p, average frame duration dropped from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Memory Complexity:** Reduced by resizing the voxel array to the tightest bounding volume. A $512^3$ array might be reduced to $512 \times 321 \times 512$, saving memory and bandwidth [S1, unofficial].
*   **Strengths:** Eliminates false positives in traversal; reduces GPU load by skipping expensive SDF evaluations; leverages GPU MIMD capabilities for better thread distribution [S1, unofficial][S2].
*   **Limitations:** Requires computing and maintaining bounding volumes; AABBs can lead to less accurate normals for lighting [S1, unofficial]; overhead of culling may not be worth it for very simple scenes or uniform geometry [S2].
*   **When to Use:** Scenes with sparse geometry, large empty spaces, or expensive SDF primitives. Not recommended for dense, uniform voxel fields where culling offers little benefit.

## Background
Bounding Volume Optimization originated in the context of raymarching and voxel rendering to address the high cost of evaluating Signed Distance Fields (SDFs) for every primitive in a scene. It was notably discussed in works on raycasting-based voxel rendering and CUDA-based SDF raymarching [S1, unofficial][S2]. The technique is particularly relevant for real-time applications where frame rates are critical, such as in game development, where it allows for the separation of game world and entities, enabling physics simulations using voxel models as rigid bodies [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
