# Inter-Ray Coherency

Also called: Reference Ray Caster

## Overview
Inter-ray coherency accelerates volume rendering by exploiting the spatial similarity between adjacent rays, allowing the renderer to skip redundant computation for rays that traverse identical or similar volume structures. Unlike [Intra-ray Parallelism](3135-intra-ray-parallelism.md), which optimizes the processing of a single ray, or [Temporal Differential Rendering](5499-temporal-differential-rendering.md), which leverages changes between frames, this method operates within a single frame by caching state from previously processed rays. It sets itself apart from [Exploiting Temporal Coherence (C-Buffer)](5560-exploiting-temporal-coherence-c-buffer.md) by focusing on spatial ray bundles rather than pixel buffers, and from [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md) by avoiding temporal blending in favor of explicit geometric caching.

## How it works
Inter-ray coherency relies on the observation that neighboring rays in a scanline or bundle often pass through the same sequence of voxels or cells. By storing the traversal state of a "reference" ray, subsequent rays can reuse this information to bypass expensive intersection tests or data lookups.

### Inputs and Outputs
*   **Input:** Volume dataset (voxel grid), camera parameters (eye point, direction), iso-surface threshold (for first-hit casting), and a cache buffer storing the state of the previously processed ray.
*   **Output:** Rendered image with reduced computational overhead.

### Data Structures
The core data structure is a **Reference Ray Buffer** [S1, unofficial]. It stores:
1.  **Cell List:** A sequence of cells (or voxels) encountered along the last processed ray, from the eye point to the first iso-surface intersection or volume exit.
2.  **Intersection Flag:** A boolean flag or bitmask indicating whether a portion of the iso-surface is contained within each cell in the list. This flag determines if an explicit ray-surface intersection test is required for that cell.

### Algorithm Steps
The method, often implemented as a "Reference Ray Caster," proceeds as follows [S1, unofficial]:

1.  **Initialize Reference Ray:** For the first ray in a bundle or scanline, perform a standard fast voxel traversal to find the entry point into the volume.
2.  **Traverse and Intersect:**
    *   Iterate through the cells along the ray path.
    *   For each cell, check the **Intersection Flag** from the reference buffer (if available).
    *   If the flag indicates an iso-surface portion is present, perform a fast ray-surface intersection test.
    *   If the flag indicates no surface, or if no reference is available, perform the standard traversal and intersection logic.
3.  **Update Reference Buffer:** Upon completing the ray, update the Reference Ray Buffer with the current ray's cell sequence and intersection flags. This buffer serves as the "memory" for the next ray in the coherent bundle.
4.  **Early Termination:** If the reference ray exited the volume or hit the iso-surface at a specific depth, subsequent rays can be terminated early if their reference state indicates the same, provided the geometric coherence holds.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state for the current ray bundle
reference_cells = []
reference_flags = [] # Boolean: iso-surface present in cell?

def render_ray_bundle(rays):
    for i, ray in enumerate(rays):
        # Step 1: Check if we have a valid reference from the previous ray
        if i > 0 and reference_cells:
            # Use reference to skip initial traversal if possible
            # Note: Exact implementation of skipping depends on ray divergence
            # The core idea is to reuse the cell list and flags
            pass
        
        # Step 2: Standard traversal with optimization
        current_cells = []
        current_flags = []
        
        # Fast voxel traversal from eye point
        cell = get_first_voxel(ray)
        while cell is not inside_volume_exit(ray):
            current_cells.append(cell)
            
            # Check for iso-surface intersection
            # If we have a reference flag for this cell index, use it
            if i > 0 and len(reference_flags) > len(current_cells) - 1:
                if reference_flags[len(current_cells) - 1]:
                    # Reference says surface is here, perform fast intersection test
                    hit = fast_ray_surface_intersection(ray, cell)
                    if hit:
                        # Record hit and potentially terminate
                        break
                else:
                    # Reference says no surface, skip expensive test
                    pass
            else:
                # No reference or new cell, perform standard test
                hit = fast_ray_surface_intersection(ray, cell)
                if hit:
                    current_flags.append(True)
                    break
                else:
                    current_flags.append(False)
            
            cell = get_next_voxel(ray, cell)
        
        # Step 3: Update reference for next ray
        reference_cells = current_cells
        reference_flags = current_flags
        
        # Composite color based on cells and hits
        color = composite(current_cells, current_flags, ray)
        write_pixel(ray.pixel_id, color)
```

### Parameters and Edge Cases
*   **Iso-value:** The technique is sensitive to the iso-surface threshold. Changing the iso-value invalidates the reference flags, requiring recomputation [S1, unofficial].
*   **Ray Divergence:** If rays diverge significantly (e.g., wide field of view), the coherence between adjacent rays decreases, reducing the benefit.
*   **Precision:** Fast voxel traversal algorithms must be robust to floating-point precision errors to ensure consistent cell identification across neighboring rays.

## Subtopics

### Marching Caches
Marching Caches is an explicit data caching strategy that extends inter-ray coherency by maintaining a shared cache around a bundle of rays, which is incrementally updated as the ray front advances through the volume [S2]. It aligns a bundle of rays such that the ray front accesses the same data simultaneously, creating a cache containing the required data around the bundle [S2]. The cache is incrementally updated in the direction of the rays as they traverse the volume, where neighboring rays share fast memory to fill missing values [S2]. This approach reduces global memory bandwidth by orders of magnitude and achieves a speedup of four times compared to using implicit hardware caching [S2]. It supports various filters including Catmul-Rom, B-spline, ambient occlusion projection, trilinear, bilateral, gradient, and filtered gradient [S2]. The algorithm involves synchronizing the ray front, calculating cache updates, and handling cache updates during ray-casting [S2].

## Operating characteristics
*   **Performance:** The basic reference ray caster accelerates rendering by about 60 percent [S1, unofficial]. Marching Caches achieve a speedup of four times compared to implicit hardware caching [S2].
*   **Memory:** Marching Caches significantly reduce memory bandwidth to global memory by orders of magnitude [S2]. The reference ray caster requires a small buffer proportional to the ray length.
*   **Limitations:** The technique is most effective for iso-surface rendering where geometric coherence is high. It is less effective for volume rendering with complex transfer functions that require per-voxel computation regardless of geometry. Changing iso-values requires cache invalidation [S1, unofficial].

## Background
The concept of inter-ray coherency was introduced in the context of cell-based first-hit ray casting by Neubauer, Mroz, Hauser, and Wegenkittl [S1, unofficial]. Their reference implementation demonstrated significant performance gains by tracking rays and caching cell encounters. Later, the Marching Caches method was proposed to further exploit this coherence by explicitly managing shared memory for ray bundles, addressing the limitations of implicit hardware caching on parallel processors [S2].

## Sources
[S1] Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] Explicit Cache Management for Volume Ray-Casting on Parallel ..., https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2012/marchingcaches_final.pdf, fetched 2026-10-29
