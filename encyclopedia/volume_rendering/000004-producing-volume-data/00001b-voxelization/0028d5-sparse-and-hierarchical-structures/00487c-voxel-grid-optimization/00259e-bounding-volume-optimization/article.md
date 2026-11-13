# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering by restricting ray marching or voxel traversal to the minimal axis-aligned bounding box (AABB) that encloses active geometry, thereby eliminating evaluations of non-contributing empty space [S1, unofficial][S2]. It sets itself apart from sibling methods like [Grid Resolution Calculation](2908-grid-resolution-calculation.md) by focusing on spatial culling of the traversal domain rather than just determining grid dimensions, and from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by addressing traversal efficiency rather than storage footprint.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scene primitives (e.g., SDF objects) and their associated bounding volumes (AABBs).
*   **Output:** A traversal mask or culling decision that skips evaluation of primitives that do not contribute to the minimal distance field or active voxel regions.

**Data Structures:**
*   **Bounding Volume:** An Axis-Aligned Bounding Box (AABB) defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$.
*   **Voxel Array Dimensions:** The dimensions of the 3D volume array are adjusted to match the actual extents of the voxel data, rather than using a fixed power-of-two size. For example, if the maximum height of a voxel landscape is 321 voxels within a 512-width base, the array size is adapted to $512 \times 321 \times 512$ instead of a fixed $512^3$ [S1, unofficial].

**Steps:**
1.  **Compute Bounding Volumes:** For each primitive in the scene, compute its AABB. This defines the spatial region where the primitive's Signed Distance Field (SDF) or geometry is relevant [S2].
2.  **Adapt Array Dimensions:** Determine the global bounding box of all active primitives. Adjust the voxel array dimensions to tightly enclose this region. This eliminates unused regions that would otherwise cause false positives during traversal [S1, unofficial].
3.  **Traverse and Cull:** During ray marching or voxel traversal, check if the current ray segment or voxel position intersects with the bounding volume of each primitive.
4.  **Conditional Evaluation:** Only evaluate the expensive SDF map function for primitives whose bounding volumes intersect the current ray or voxel location. Primitives outside the bounding volume are skipped [S2].
5.  **Handle Dynamic Scenes:** For dynamic scenes, update the bounding volumes and array dimensions as primitives move. This allows dense voxel models to serve as bounding boxes for rigid bodies, where voxels act as contact points and mass parts [S1, unofficial].

**Formulas:**
*   **Array Size Adaptation:** If the scene extents are $[X_{min}, X_{max}] \times [Y_{min}, Y_{max}] \times [Z_{min}, Z_{max}]$ and voxel size is $v$, the array dimensions $D_x, D_y, D_z$ are:
    $$ D_x = \lceil \frac{X_{max} - X_{min}}{v} \rceil, \quad D_y = \lceil \frac{Y_{max} - Y_{min}}{v} \rceil, \quad D_z = \lceil \frac{Z_{max} - Z_{min}}{v} \rceil $$
    [S1, unofficial]
*   **Culling Condition:** A primitive $P$ with AABB $B_P$ is evaluated for a ray origin $O$ and direction $D$ at step $t$ if:
    $$ O + tD \in B_P $$
    [S2]

**Parameters:**
*   **Voxel Size:** Determines the resolution and the tightness of the bounding volume adaptation. Not given in the sources.
*   **Primitive Count:** The number of primitives affects the overhead of bounding volume checks. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **False Positives:** Using a fixed, oversized array (e.g., $512^3$ for a $512 \times 321 \times 512$ scene) loads unnecessary data into GPU memory and causes false positives during traversal [S1, unofficial].
*   **Lighting Artifacts:** AABBs have flat faces with constant normals, which can lead to hard, physically implausible lighting if not handled carefully (e.g., sharp transitions between lit and unlit faces) [S1, unofficial].
*   **GPU Architecture:** Modern GPUs are MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD. This allows different Streaming Multiprocessors (SMs) to process different instruction streams, helping to distribute expensive threads after cheaper culling operations finish early [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(scene_primitives, ray_origin, ray_direction, voxel_size):
    # Step 1: Compute global bounding box of all primitives
    global_min = [inf, inf, inf]
    global_max = [-inf, -inf, -inf]
    
    for primitive in scene_primitives:
        bbox = compute_aabb(primitive)
        global_min = min(global_min, bbox.min)
        global_max = max(global_max, bbox.max)
        
    # Step 2: Adapt array dimensions
    array_dims = [
        ceil((global_max[0] - global_min[0]) / voxel_size),
        ceil((global_max[1] - global_min[1]) / voxel_size),
        ceil((global_max[2] - global_min[2]) / voxel_size)
    ]
    
    # Step 3: Traverse and cull
    distance = 0.0
    while distance < max_ray_length:
        current_pos = ray_origin + ray_direction * distance
        
        # Check if current position is within global bounds
        if not is_inside_bbox(current_pos, global_min, global_max):
            break
            
        min_dist = inf
        for primitive in scene_primitives:
            # Step 4: Conditional evaluation
            if is_bbox_intersect_ray(primitive.aabb, ray_origin, ray_direction, distance):
                sdf_val = evaluate_sdf(primitive, current_pos)
                min_dist = min(min_dist, sdf_val)
                
        if min_dist == inf:
            # No primitives found, skip ahead or break
            break
            
        distance += min_dist
        
    return distance
```

## Subtopics

### 3D Voxel Grid Smoothing
Smooths the voxel grid to reduce aliasing and improve visual quality. Choose when visual fidelity is more important than raw traversal speed. [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)

### Binary Voxel RLE Encoding
Compresses binary voxel data using Run-Length Encoding. Choose when memory bandwidth is the bottleneck. [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)

### Coordinate Grid Generation
Generates the coordinate grid for voxel traversal. Choose when setting up the traversal structure. [Coordinate Grid Generation](1273-coordinate-grid-generation.md)

### Occupancy Voxel Creation
Creates occupancy voxels from geometry. Choose when converting mesh data to voxel data. [Occupancy Voxel Creation](2628-occupancy-voxel-creation.md)

### Grid Resolution Calculation
Calculates the optimal grid resolution. Choose when balancing memory and accuracy. [Grid Resolution Calculation](2908-grid-resolution-calculation.md)

### Voxelization Optimization
Optimizes the voxelization process. Choose when accelerating the conversion from geometry to voxels. [Voxelization Optimization](3191-voxelization-optimization.md)

### Bucket-based LBVH construction
Constructs a Linear Bounding Volume Hierarchy using bucketing. Choose for complex scenes with many primitives to accelerate traversal. [Bucket-based LBVH construction](5163-bucket-based-lbvh-construction.md)

## Operating characteristics
*   **Time Complexity:** Reduces the number of SDF evaluations from $O(N \cdot M)$ to $O(K \cdot M)$, where $N$ is the total number of primitives, $M$ is the number of ray steps, and $K$ is the number of primitives intersecting the ray's bounding volume. In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Memory Complexity:** Reduces memory usage by adapting the voxel array size to the actual data extents. A fixed $512^3$ array is replaced by a tighter $512 \times 321 \times 512$ array, saving memory and reducing cache misses [S1, unofficial].
*   **Strengths:** Eliminates false positives, reduces GPU load by skipping expensive computations, and leverages GPU MIMD architecture to distribute workload efficiently [S1, unofficial][S2].
*   **Limitations:** AABBs can cause lighting artifacts due to flat normals [S1, unofficial]. The overhead of bounding volume checks must be justified by the savings in SDF evaluations.
*   **When to Use:** Use when the scene contains many primitives, especially expensive SDFs, and when the geometry is sparse or confined to a small region within a larger volume.
*   **When Not to Use:** Avoid when the scene is uniformly dense or when the overhead of bounding volume checks exceeds the savings from culling.

## Background
Bounding Volume Optimization originated as a technique to minimize the cost of the map function in raymarching by skipping SDF evaluations for non-contributing objects [S2]. It is widely used in real-time raymarching and voxel rendering to improve performance, particularly in scenes with complex geometry or large empty spaces [S1, unofficial][S2].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
