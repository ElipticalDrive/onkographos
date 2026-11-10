# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of voxel grid generation and ray marching by eliminating empty space and non-contributing geometry from the active computation domain [S1, unofficial][S2]. It achieves this by either adapting the 3D voxel array dimensions to the actual extent of the data or by culling primitives using bounding volumes before evaluating expensive Signed Distance Fields (SDFs) [S1, unofficial][S2]. Unlike sibling methods that focus on data encoding or smoothing, this method specifically targets the reduction of false positives and redundant map evaluations by restricting the active volume [S1, unofficial][S2].

## How it works
The method operates in two distinct modes depending on the rendering pipeline: spatial culling via array resizing and geometric culling via bounding volume testing.

### Inputs and Outputs
*   **Inputs**:
    *   Voxel data extent (min/max coordinates or dimensions) [S1, unofficial].
    *   Scene primitives with associated SDF map functions [S2].
    *   Bounding volumes (typically Axis-Aligned Bounding Boxes, AABBs) for each primitive [S1, unofficial][S2].
*   **Outputs**:
    *   A resized voxel texture/array matching the data extent [S1, unofficial].
    *   A culling mask or early-out decision for ray-primitive intersections [S2].

### Data Structures
1.  **Voxel Array**: A 3D texture or buffer. The dimensions $(D_x, D_y, D_z)$ are set to the minimal bounding box of the active voxel data, rather than a fixed power-of-two size [S1, unofficial].
2.  **Bounding Volume List**: An array of AABBs, where each AABB is defined by two 3D vectors: $\mathbf{min} = (x_{min}, y_{min}, z_{min})$ and $\mathbf{max} = (x_{max}, y_{max}, z_{max})$ [S2].

### Steps
1.  **Determine Data Extent**: Calculate the minimum and maximum coordinates occupied by the voxel data [S1, unofficial].
2.  **Resize Array**: Allocate the voxel texture with dimensions equal to the data extent. For example, if the landscape height is 321 voxels within a 512x512 footprint, use dimensions $512 \times 321 \times 512$ instead of a fixed $512^3$ [S1, unofficial].
3.  **Construct Bounding Volumes**: For each scene primitive, compute its AABB [S2].
4.  **Ray-Volume Intersection Test**: For each ray (or pixel), test intersection against the bounding volumes before evaluating the SDF [S2].
5.  **Early Out**: If a ray does not intersect a primitive's bounding volume, skip the evaluation of that primitive's SDF map function [S2].
6.  **Traverse**: Perform ray marching only within the resized array bounds [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounding_volume_optimization(ray_origin, ray_direction, scene_primitives, voxel_array):
    """
    ray_origin: vec3
    ray_direction: vec3
    scene_primitives: list of objects, each having .aabb (min, max) and .sdf_function()
    voxel_array: 3D array with dimensions matching the data extent
    """
    # Step 1: Initialize minimum distance found so far
    min_dist = float('inf')
    
    # Step 2: Cull primitives using bounding volumes
    for primitive in scene_primitives:
        # Test ray against AABB
        if ray_intersects_aabb(ray_origin, ray_direction, primitive.aabb):
            # Step 3: Evaluate SDF only if bounding volume is hit
            # This skips expensive SDF evaluations for non-contributing objects
            dist = primitive.sdf_function(ray_origin, ray_direction)
            if dist < min_dist:
                min_dist = dist
        else:
            # Skip this primitive
            continue
            
    # Step 4: March within the resized voxel array bounds
    # The array dimensions are already set to the minimal extent
    # to avoid false positives from empty space [S1, unofficial]
    current_pos = ray_origin
    for step in range(MAX_STEPS):
        if current_pos is out_of_bounds(voxel_array):
            break
            
        # Sample voxel data or use SDF result
        # ...
        
        current_pos += ray_direction * step_size
        
    return min_dist

def ray_intersects_aabb(origin, direction, aabb):
    """
    Standard ray-AABB intersection test.
    aabb.min and aabb.max are vec3.
    """
    inv_dir = 1.0 / direction
    t_min = (aabb.min - origin) * inv_dir
    t_max = (aabb.max - origin) * inv_dir
    
    t1 = min(t_min)
    t2 = max(t_max)
    
    if t1 > t2:
        return False
        
    # Check if the intersection is within the bounding box range
    if t2 < 0:
        return False
        
    return True
```

### Parameters and Edge Cases
*   **Array Dimensions**: Must be calculated dynamically based on the scene content to avoid loading unused data into GPU memory [S1, unofficial].
*   **AABB Normal Limitations**: Using AABBs for lighting can result in hard, physically implausible lighting because the normals of the cube faces do not provide smooth gradients [S1, unofficial].
*   **GPU Architecture**: Modern GPUs are MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD. This allows different Streaming Multiprocessors (SMs) to process different instruction streams, helping distribute expensive threads after cheaper parts finish early [S2].

## Operating characteristics
*   **Performance**: In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Memory**: Reduces memory bandwidth and cache pressure by loading only the necessary voxel data (e.g., $512 \times 321 \times 512$ instead of $512^3$) [S1, unofficial].
*   **Complexity**: The cost of bounding volume tests is negligible compared to SDF evaluations, leading to a net performance gain in complex scenes [S2].
*   **Limitations**: AABBs do not provide visually appealing lighting due to flat normals [S1, unofficial].

## Background
The concept of adapting voxel array sizes to reduce false positives and memory usage was discussed in the context of voxel landscapes [S1, unofficial]. The application of bounding volume optimization to SDF ray marching to minimize map function evaluation costs was explored in CUDA-based raymarching research [S2].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
