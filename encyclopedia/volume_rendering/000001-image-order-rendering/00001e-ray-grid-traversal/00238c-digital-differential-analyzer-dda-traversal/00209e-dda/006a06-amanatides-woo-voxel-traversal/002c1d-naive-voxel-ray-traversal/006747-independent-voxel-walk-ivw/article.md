# Independent Voxel Walk (IVW)


## Overview
Independent Voxel Walk (IVW) is a GPU-based ray traversal method where each ray is assigned to an independent thread that walks through a uniform grid, loading triangle data for the current voxel and checking for intersections until the first hit is found [S1]. It differs from sibling methods like Enforced Coherence (EC) by assuming no coherence between rays, relying instead on modern GPU memory hierarchies (L1/L2 caches) to handle the resulting memory access patterns, whereas EC explicitly reorders rays to exploit spatial coherence [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rays (primary and/or reflection), a uniform grid data structure containing triangles, and the ray origins and directions.
*   **Output:** The first intersection point (or null) for each ray.

**Data Structures:**
*   **Uniform Grid:** A 3D array or hash map mapping voxel coordinates to a list of triangle indices contained within that voxel. The grid is constructed per frame or per stage [S1].
*   **Ray Data:** Origin ($O$) and Direction ($D$) for each ray.

**Algorithm Steps:**
1.  **Parallel Assignment:** Assign each ray to an independent GPU thread [S1].
2.  **Traversal Loop:** For each thread (ray), execute the following loop until an intersection is found or the ray exits the scene:
    a.  **Determine Current Voxel:** Compute the voxel coordinates containing the current ray position. This is typically done using the ray's origin (for the first step) or the current position along the ray [S1].
    b.  **Load Triangles:** Retrieve the list of triangle indices associated with the current voxel from global memory [S1].
    c.  **Serial Intersection Check:** Iterate through the triangles in the current voxel in serial order. For each triangle, perform a ray-triangle intersection test [S1].
    d.  **Termination:** If an intersection is found, record the hit and terminate the ray's traversal [S1]. If no intersection is found, advance the ray to the next voxel along its path and repeat [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Algorithm 2: Independent Voxel Walk [S1]
# Assumes a global function determineVoxel(ray) that returns the voxel ID
# containing the ray's current position.
# Assumes a global function getTriangles(voxel_id) that returns a list of triangle indices.
# Assumes a function checkIntersection(ray, triangle) that returns true if hit.

for ray in all_rays in parallel do:
    while not ray.has_intersection:
        voxel_id = determineVoxel(ray)
        triangles = getTriangles(voxel_id)
        
        for triangle in triangles in serial do:
            if checkIntersection(ray, triangle):
                ray.record_intersection(triangle)
                ray.has_intersection = True
                break
```

**Parameters and Typical Values:**
*   **Grid Type:** Uniform grid [S1].
*   **Hardware:** Tested on Nvidia GTX480 (Fermi architecture) and GTX280 [S1].

**Edge Cases and Pitfalls:**
*   **Lack of Coherence:** On older GPUs with large SIMD width and no caching (e.g., GTX280), the lack of coherence between rays can incur a heavy performance penalty due to redundant memory loads [S1].
*   **Cache Dependency:** Performance is heavily dependent on the presence of L1 and L2 caches. If multiple rays check the same voxel simultaneously or close in time, the cache can mitigate the lack of coherence [S1].
*   **Empty Voxels:** Models with a lot of empty space create many ray-voxel pairs, which can affect performance if the grid is not optimized, though IVW avoids the sorting overhead of EC [S1].

## Subtopics

### Enforced Coherence (EC)
A method that reorders rays to process all rays passing through a voxel together, exploiting shared memory. It is faster on older, cache-less GPUs but slower or comparable on modern cached GPUs [S1]. [Enforced Coherence](5193-enforced-coherence.md)

### Uniform Grid Traversal
The underlying spatial partitioning structure used by IVW. It provides simple voxel lookup but may have high memory overhead for sparse scenes [S1]. [Uniform Grid](5191-uniform-grid.md)

## Operating characteristics
*   **Time Complexity:** Linear in the number of rays and the average number of voxels traversed per ray, multiplied by the average number of triangles per voxel. The serial intersection check within a voxel adds a constant factor per voxel [S1].
*   **Memory Complexity:** Depends on the grid structure and the number of triangles. The grid is rebuilt per frame or stage [S1].
*   **Measured Performance:** On an Nvidia GTX480, IVW achieved 14 fps on the Conference model with one light source and one reflection bounce, compared to 8 fps for the Enforced Coherence method [S1].
*   **Scaling:** Expected to benefit from future GPUs and manycore architectures as cores become more independent and caching mechanisms become more flexible [S1].
*   **Strengths:** Simple implementation; no need for ray sorting or reordering; exploits L1/L2 caches effectively on modern GPUs [S1].
*   **Limitations:** Poor performance on older GPUs without caches; performance degrades with high triangle density per voxel [S1].
*   **When to Use:** Modern GPUs with L1/L2 caches (e.g., Fermi and later); scenes where ray coherence is low or unpredictable [S1].
*   **When Not to Use:** Older GPUs without caches (e.g., GTX280); scenes with high spatial coherence where Enforced Coherence can exploit shared memory more effectively [S1].

## Background
Independent Voxel Walk was introduced in the context of raytracing dynamic scenes on the GPU, specifically as a comparison to the Enforced Coherence method [S1]. It leverages the moderate L1 and L2 caches present in GPUs like the Nvidia GTX480 (Fermi architecture) to handle the memory access patterns of independent rays [S1]. The method is part of a broader effort to optimize raytracing on GPUs by exploiting different aspects of the hardware memory hierarchy [S1].

## Sources
[S1] Raytracing dynamic scenes on the GPU using grids, https://ieeexplore.ieee.org/abstract/document/5728799/, fetched 2026-10-25
