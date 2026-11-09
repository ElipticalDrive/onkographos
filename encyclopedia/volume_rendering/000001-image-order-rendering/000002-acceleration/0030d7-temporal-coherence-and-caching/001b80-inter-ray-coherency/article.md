# Inter-Ray Coherency

Also called: Reference Ray Caster

## Overview
Inter-Ray Coherency accelerates volume ray casting by exploiting the spatial correlation between adjacent rays, assuming that neighboring rays traverse similar voxel paths and encounter similar surface structures. It sets itself apart from sibling methods like [Intra-ray Parallelism](3135-intra-ray-parallelism.md) by focusing on reducing redundant global memory accesses across a bundle of rays, rather than optimizing the computation within a single ray. Unlike [Temporal Differential Rendering](5499-temporal-differential-rendering.md) which relies on frame-to-frame changes, this method operates within a single frame to skip expensive intersection tests for rays that share cached traversal history.

## How it works
The method relies on the assumption that camera motion and volume changes are small between frames, but specifically exploits the coherence between rays cast to adjacent pixels in the same frame.

**Inputs and Outputs:**
*   **Input:** Volume dataset, camera parameters, iso-surface threshold.
*   **Output:** Rendered image with reduced traversal overhead.

**Data Structures:**
1.  **Reference Ray Buffer:** A buffer storing the sequence of cells (voxels) encountered along the most recently processed ray (or a reference ray).
2.  **Intersection Flag:** A boolean flag associated with the buffer, indicating whether a portion of the iso-surface is contained within the stored cells.

**Algorithm Steps:**
1.  **Ray Casting:** Cast rays from the eye point through each pixel.
2.  **Traversal:** Use a fast voxel traversal algorithm to step through the volume grid.
3.  **Coherency Check:** For the current ray, check if it visits the same cells as the reference ray stored in the buffer.
4.  **Intersection Test Optimization:**
    *   If the current ray visits the same cells as the reference ray:
        *   Check the intersection flag.
        *   If the flag indicates an iso-surface portion is present, execute the fast ray-surface intersection test.
        *   If the flag indicates no surface, skip the intersection test entirely.
    *   If the current ray visits different cells, perform standard traversal and intersection tests, then update the reference ray buffer with the new cell sequence and flag state.
5.  **Termination:** Stop traversal when the iso-surface is intersected or the ray exits the volume.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state for inter-ray coherency
reference_cells = []
reference_has_surface = False

def cast_ray_with_coherency(ray):
    global reference_cells, reference_has_surface
    
    # 1. Traverse the volume to find cells visited by the current ray
    current_cells = fast_voxel_traversal(ray)
    
    # 2. Check for inter-ray coherency
    if current_cells == reference_cells:
        # 3. Exploit coherency: use cached intersection info
        if reference_has_surface:
            # Perform fast ray-surface intersection test
            hit = fast_ray_surface_intersection(ray, current_cells)
            if hit:
                return hit
        # If no surface cached, or intersection failed, continue standard traversal
        # Note: In practice, this might fall back to full traversal or use the 
        # cached cells as a starting point.
    
    # 4. Standard traversal and intersection if coherency not used or failed
    for cell in current_cells:
        if ray_intersects_iso_surface(ray, cell):
            # Update reference buffer for next rays
            reference_cells = current_cells
            reference_has_surface = True
            return get_intersection_point(ray, cell)
            
    # Update reference buffer for rays that exit without hitting surface
    reference_cells = current_cells
    reference_has_surface = False
    return None
```

**Parameters:**
*   **Reference Ray Selection:** Typically the previously processed ray in scanline order.
*   **Performance:** This technique accelerates rendering by about 60 percent [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Disocclusions:** When a surface moves out of view or a new surface appears, the cached flag may be incorrect. The method must detect when the current ray's path diverges significantly from the reference ray to avoid stale data.
*   **Iso-value Changes:** Distance coding was not used in the reference implementation because it would require recomputation for each new iso-value [S1, unofficial].

## Subtopics

### Marching Caches
An explicit data caching strategy for volume ray-casting that exploits inter-ray coherence by maintaining a shared cache around a bundle of rays, which is incrementally updated as the ray front advances through the volume, reducing global memory bandwidth [S2]. It sets itself apart from the basic reference ray caster by actively managing a shared memory cache that "marches" with the ray bundle, allowing neighboring rays to share fast memory to fill missing values [S2]. Choose this method when memory bandwidth is the bottleneck and higher-order reconstruction filters are required [S2].

**Steps:**
1.  Align a bundle of rays such that the ray front accesses the same data simultaneously [S2].
2.  Create a cache containing the required data around the bundle of rays [S2].
3.  Incrementally update the cache in the direction of the rays as they traverse the volume, where neighboring rays share fast memory to fill missing values [S2].
4.  Synchronize the ray front and incrementally update the cache as the ray front advances [S2].

**Parameters:**
*   **Filters Supported:** Catmul-Rom, B-spline, ambient occlusion projection, trilinear, bilateral, gradient, and filtered gradient [S2].
*   **Performance:** Achieved a speedup of four times compared to using implicit hardware caching and reduced memory bandwidth to global memory by orders of magnitude [S2].

## Operating characteristics
*   **Time Complexity:** Reduces the number of expensive ray-surface intersection tests by exploiting spatial coherence. The reference implementation reports a 60% acceleration [S1, unofficial].
*   **Memory Complexity:** Requires additional storage for the reference ray buffer and intersection flags. Marching Caches require explicit shared memory management for the cache around the ray bundle [S2].
*   **Strengths:** Significant reduction in global memory bandwidth usage, especially with Marching Caches [S2]. Improved performance for high-fidelity rendering with complex filters [S2].
*   **Limitations:** Effectiveness depends on the coherence between adjacent rays. Performance may degrade in scenes with high-frequency details or large disocclusions. Not suitable for dynamic iso-values without recomputation [S1, unofficial].
*   **When to Use:** Static or slowly changing volumes where adjacent rays share similar traversal paths. Ideal for reducing memory bandwidth bottlenecks on GPUs.
*   **When Not to Use:** Scenes with rapid camera motion causing frequent disocclusions, or when iso-values change frequently.

## Background
The concept of Inter-Ray Coherency was introduced in the context of cell-based first-hit ray casting by Neubauer, Mroz, Hauser, and Wegenkittl [S1, unofficial]. It was designed to accelerate the visualization of medical data, such as virtual angioscopy, by reducing redundant intersection tests. The Marching Caches variant was later developed to further optimize memory access patterns on parallel processors by explicitly managing shared caches [S2].

## Sources
[S1] Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] Explicit Cache Management for Volume Ray-Casting on Parallel ..., https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2012/marchingcaches_final.pdf, fetched 2026-10-29
