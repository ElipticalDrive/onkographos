# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering by eliminating empty space or non-contributing geometry from the active computation domain [S1, unofficial][S2]. It achieves this by either resizing the voxel array to the minimal bounding box of the data or using bounding volumes to skip Signed Distance Field (SDF) evaluations for objects that do not contribute to the minimal distance field [S1, unofficial][S2]. Unlike sibling methods that focus on data compression or smoothing, this method specifically targets traversal efficiency and evaluation cost by localizing expensive computations to areas with complex geometry [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete voxel grid produced by [Voxelization](12-voxelization.md), a set of scene primitives (e.g., SDF objects), and their associated bounding volumes (typically Axis-Aligned Bounding Boxes, AABBs).
*   **Output:** An optimized rendering state where empty regions are culled or SDF evaluations are skipped, resulting in reduced frame duration.

**Data Structures:**
1.  **Voxel Array:** A 3D texture or buffer. The dimensions $(W, H, D)$ are dynamically adjusted to the minimal bounding box of the active voxel data. For example, if a landscape occupies $512 \times 321 \times 512$ voxels within a potential $512^3$ space, the array is resized to $512 \times 321 \times 512$ [S1, unofficial].
2.  **Bounding Volumes:** A list of AABBs, one per primitive or object group. Each AABB is defined by a minimum corner $(x_{min}, y_{min}, z_{min})$ and a maximum corner $(x_{max}, y_{max}, z_{max})$ [S2].

**Steps:**
1.  **Determine Active Bounds:** Calculate the minimal bounding box that encloses all non-empty voxels or relevant scene geometry. Resize the voxel array to these dimensions to eliminate unused memory and traversal steps [S1, unofficial].
2.  **Construct Bounding Volumes:** For each primitive in the scene, compute or retrieve its AABB. These volumes define the spatial extent of the primitive [S2].
3.  **Traversal/Culling:** During ray marching or voxel traversal:
    *   Check if the current ray segment or voxel coordinate intersects the bounding volume of a primitive.
    *   If the ray is outside the bounding volume, skip the evaluation of the primitive's SDF map function [S2].
    *   If the ray is inside, evaluate the SDF to determine the distance to the surface [S2].
4.  **Accumulate Distance:** Compute the minimum distance from the ray origin to all primitives whose bounding volumes were intersected. Empty regions contribute no distance (or infinite distance) and are skipped [S2].

**Formulas:**
*   **AABB Intersection Test:** A ray with origin $\mathbf{o}$ and direction $\mathbf{d}$ intersects an AABB defined by $\mathbf{min}$ and $\mathbf{max}$ if:
    $$ t_{min} = \max \left( \frac{min_x - o_x}{d_x}, \frac{min_y - o_y}{d_y}, \frac{min_z - o_z}{d_z} \right) $$
    $$ t_{max} = \min \left( \frac{max_x - o_x}{d_x}, \frac{max_y - o_y}{d_y}, \frac{max_z - o_z}{d_z} \right) $$
    Intersection occurs if $t_{min} \le t_{max}$ and $t_{max} \ge 0$. [G: Standard ray-AABB intersection logic].
*   **Array Indexing:** For a resized array of dimensions $W \times H \times D$, a voxel at logical coordinates $(i, j, k)$ is accessed at index $i + j \cdot W + k \cdot W \cdot H$ [G: Standard 3D array indexing].

**Parameters:**
*   **Bounding Volume Type:** AABBs are standard due to fast intersection tests [S1, unofficial].
*   **Resolution:** The voxel resolution must be sufficient to capture the geometry within the bounding box. [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Positives:** Using a fixed, oversized array (e.g., $512^3$ for a $512 \times 321 \times 512$ scene) loads unnecessary data into GPU memory and causes false positives during traversal [S1, unofficial].
*   **Lighting Artifacts:** Using AABBs for normal estimation can result in hard, non-physically plausible lighting because the normals of the cube faces do not match the curved surface of the primitive [S1, unofficial].
*   **GPU Architecture:** Modern GPUs are MIMD (Multiple Instruction, Multiple Data) rather than pure SIMD. Different Streaming Multiprocessors (SMs) can process different instruction streams simultaneously. This allows the GPU to finish computation of cheaper parts of the frame (where bounding volumes are skipped) early and distribute more expensive threads (where SDFs are evaluated) across available SMs [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(ray_origin, ray_direction, voxel_array, bounding_volumes):
    # 1. Determine if ray intersects any bounding volume
    hit_any = False
    min_dist = INFINITY
    
    for bv in bounding_volumes:
        # Check intersection with AABB
        t_min, t_max = intersect_aabb(ray_origin, ray_direction, bv.min, bv.max)
        
        if t_min <= t_max and t_max >= 0:
            hit_any = True
            # 2. If intersected, evaluate SDF to get precise distance
            # This is the expensive operation, skipped if bv is not intersected
            dist = evaluate_sdf(ray_origin, ray_direction, bv.primitive)
            if dist < min_dist:
                min_dist = dist
                
    if not hit_any:
        return BACKGROUND_COLOR
        
    # 3. Proceed with raymarching using min_dist
    # ... (standard raymarching loop)
    return final_color
```

## Subtopics

### 3D Voxel Grid Smoothing
Applies post-processing to smooth scalar fields. Choose when visual continuity is more important than traversal speed. [3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)

### Binary Voxel RLE Encoding
Compresses binary voxel data. Choose when memory bandwidth is the bottleneck and data is binary. [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)

### Coordinate Grid Generation
Generates the spatial coordinates for voxels. Choose when mapping voxel indices to world space is required. [Coordinate Grid Generation](1273-coordinate-grid-generation.md)

### Occupancy Voxel Creation
Creates binary occupancy masks. Choose when only presence/absence of geometry is needed. [Occupancy Voxel Creation](2628-occupancy-voxel-creation.md)

### Grid Resolution Calculation
Determines the optimal voxel size. Choose when balancing detail and performance. [Grid Resolution Calculation](2908-grid-resolution-calculation.md)

### Voxelization Optimization
Optimizes the generation of the voxel grid. Choose when the voxelization step is the bottleneck. [Voxelization Optimization](3191-voxelization-optimization.md)

### Bucket-based LBVH construction
Builds a Linear Bounding Volume Hierarchy. Choose when accelerating ray-primitive intersection for complex scenes. [Bucket-based LBVH construction](5163-bucket-based-lbvh-construction.md)

## Operating characteristics
*   **Time Complexity:** Reduces the number of SDF evaluations from $O(N \cdot M)$ to $O(N \cdot K)$, where $N$ is the number of rays, $M$ is the total number of primitives, and $K$ is the number of primitives intersecting the ray's bounding volume ($K \ll M$) [S2].
*   **Memory Complexity:** Reduces memory usage by resizing the voxel array to the minimal bounding box. For example, reducing from $512^3$ to $512 \times 321 \times 512$ saves approximately 37% of memory [S1, unofficial].
*   **Measured Performance:** In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Strengths:** Localizes expensive computations to areas with complex geometry; reduces false positives; separates game world and entities for physics simulation [S1, unofficial][S2].
*   **Limitations:** AABBs can cause visually unappealing lighting if used for normal estimation; requires pre-computation of bounding volumes [S1, unofficial].
*   **When to Use:** When the scene contains many primitives, most of which are empty or far from the camera.
*   **When Not to Use:** When the scene is uniformly dense or when lighting artifacts from AABB normals are unacceptable.

## Background
Bounding Volume Optimization originated in the context of raymarching and voxel rendering to address the high computational cost of evaluating Signed Distance Fields (SDFs) for every primitive in a scene [S2]. It was notably discussed in the context of optimizing CUDA-based raymarchers and voxel-based game engines [S1, unofficial][S2]. The technique leverages the MIMD nature of modern GPUs to distribute workload efficiently by skipping unnecessary calculations [S2].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
