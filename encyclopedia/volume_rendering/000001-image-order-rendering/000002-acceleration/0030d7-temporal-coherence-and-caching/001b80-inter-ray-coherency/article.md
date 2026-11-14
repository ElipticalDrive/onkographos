# Inter-Ray Coherency

Also called: Reference Ray Caster

## Overview
Inter-Ray Coherency accelerates volume rendering by exploiting the spatial correlation between adjacent rays, assuming that neighboring rays traverse similar voxel sequences and encounter similar surface structures [G]. It sets itself apart from sibling methods like [Temporal Differential Rendering](5499-temporal-differential-rendering.md) by leveraging spatial proximity within a single frame rather than temporal consistency across frames, and from [Intra-ray Parallelism](3135-intra-ray-parallelism.md) by focusing on data reuse between distinct rays rather than parallelizing the computation along a single ray path [G].

## How it works
**Inputs:** A 3D volume dataset, a set of eye rays (one per pixel), and a buffer storing the traversal state of the previously processed ray [S1, unofficial].
**Outputs:** A rendered image with reduced computational overhead [G].

**Data Structures:**
1. **Reference Buffer:** Stores the sequence of cell indices encountered by the last processed ray [S1, unofficial].
2. **Intersection Flag:** A boolean indicating whether a portion of the iso-surface is contained within the stored cells [S1, unofficial].

**Algorithm Steps:**
1. **Ray Traversal:** Track rays from the eye point to the first iso-surface intersection or volume exit using fast voxel traversal [S1, unofficial].
2. **Coherency Check:** Compare the current ray's path against the data stored in the Reference Buffer [S1, unofficial].
3. **Early Termination/Skip:** If the current ray visits the same cells as the reference ray and the Intersection Flag indicates no iso-surface intersection in those cells, skip the expensive ray-surface intersection tests for those segments [S1, unofficial].
4. **Intersection Test:** If the coherency check fails or the flag indicates a potential intersection, perform fast ray-surface intersection tests [S1, unofficial].
5. **Buffer Update:** After processing the ray, update the Reference Buffer with the cells encountered by the current ray and update the Intersection Flag based on the result [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(volume_data, rays):
    # Inter-ray coherency state
    reference_cells = []  # Cell indices from the last processed ray
    reference_has_intersection = False  # True if iso-surface found in reference_cells

    for ray in rays:
        current_cells = []
        has_intersection = False
        acc_color = [0.0, 0.0, 0.0]
        acc_alpha = 0.0

        t = 0.0
        step = volume_data.step_size
        max_dist = volume_data.max_distance

        while t < max_dist:
            cell = volume_data.get_cell(ray.origin + ray.direction * t)
            current_cells.append(cell)

            # Coherency check against reference buffer
            if cell in reference_cells:
                if not reference_has_intersection:
                    # Skip expensive surface intersection test [S1, unofficial]
                    pass
                else:
                    if volume_data.ray_surface_intersect(ray, cell):
                        has_intersection = True
                        acc_color, acc_alpha = volume_data.composite_hit(ray, cell, acc_color, acc_alpha)
                        break
            else:
                if volume_data.ray_surface_intersect(ray, cell):
                    has_intersection = True
                    acc_color, acc_alpha = volume_data.composite_hit(ray, cell, acc_color, acc_alpha)
                    break

            # Standard volume compositing for non-surface cells [G]
            sample = volume_data.sample(cell)
            acc_color, acc_alpha = volume_data.composite(sample, acc_color, acc_alpha)

            t += step

        # Update reference buffer for next ray
        reference_cells = current_cells
        reference_has_intersection = has_intersection

        yield ray.id, acc_color + [acc_alpha]
```

**Parameters:**
*   **Reference Buffer Size:** Not explicitly defined, but effectively limited by the ray length through the volume [G].
*   **Iso-value Threshold:** Used to determine surface presence; distance coding was not used in the reference implementation because it would require recomputation for each new iso-value [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Iso-value Changes:** Changing the iso-value invalidates the reference buffer unless distance coding is employed, which adds computational cost [S1, unofficial].
*   **Discontinuous Geometry:** In regions with high-frequency detail or discontinuities, the assumption of inter-ray coherency breaks down, leading to missed optimizations or incorrect skips [G].
*   **Synchronization:** In parallel implementations, updating the reference buffer must be handled carefully to avoid race conditions if multiple rays update it simultaneously [G].

## Subtopics

### Marching Caches
An explicit data caching strategy that exploits inter-ray coherence by maintaining a shared cache around a bundle of rays, which is incrementally updated as the ray front advances through the volume [S2]. It differs from the basic Reference Ray Caster by explicitly managing a shared memory cache for a bundle of rays rather than a single reference ray, allowing for higher-order filters and significant bandwidth reduction [S2]. Choose this method when memory bandwidth is the bottleneck and higher-order reconstruction filters are required [S2].

**Steps:**
1. **Align Bundle:** Align a bundle of rays such that the ray front accesses the same data simultaneously [S2].
2. **Create Cache:** Create a cache containing the required data around the bundle of rays [S2].
3. **Incremental Update:** Incrementally update the cache in the direction of the rays as they traverse the volume, where neighboring rays share fast memory to fill missing values [S2].
4. **Synchronize:** Synchronize the ray front and incrementally update the cache as the ray front advances [S2].

**Parameters:**
*   **Filters:** Supports Catmul-Rom, B-spline, ambient occlusion projection, trilinear, bilateral, gradient, and filtered gradient [S2].

**Performance:**
*   Achieved a speedup of four times compared to using implicit hardware caching [S2].
*   Reduced memory bandwidth to global memory by orders of magnitude [S2].

## Operating characteristics
**Time Complexity:** The method reduces the number of expensive ray-surface intersection tests [G]. The reference implementation accelerated rendering by about 60 percent [S1, unofficial]. Marching Caches achieved a four-fold speedup [S2].
**Memory Complexity:** Requires additional memory for the reference buffer or marching cache [G]. Marching Caches reduce global memory bandwidth, which is critical for performance on GPUs [S2].
**Strengths:** Significant performance gains in coherent scenes; reduces memory bandwidth pressure [G].
**Limitations:** Performance degrades in scenes with high-frequency details or when iso-values change frequently [G]. The reference implementation did not use distance coding, limiting flexibility with iso-values [S1, unofficial].

## Background
The concept of Inter-Ray Coherency was introduced in the context of cell-based first-hit ray casting by Neubauer, Mroz, Hauser, and Wegenkittl [S1, unofficial]. The Marching Caches variant was later proposed to address memory bandwidth limitations on parallel processors by explicitly managing shared memory caches [S2].

## Sources
[S1] Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] Explicit Cache Management for Volume Ray-Casting on Parallel ..., https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2012/marchingcaches_final.pdf, fetched 2026-10-29
