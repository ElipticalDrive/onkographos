# Volume Access Caching

Also called: Back-sample caching, Iteration-level volume access caching

## Overview
Volume Access Caching is a technique used within the Evaluation Cache framework to reduce redundant memory bandwidth consumption during ray-marching volume rendering. It solves the problem of repeatedly fetching the same voxel data when the sampling step for the current iteration overlaps with the end point of the previous iteration. It sets itself apart from sibling methods like [Edge Point Caching](891-edge-point-caching.md) by focusing on temporal coherence between consecutive samples along a single ray, rather than spatial coherence across neighboring pixels or rays.

## How it works
**Inputs and Outputs:**
*   Input: The current ray parameter $t_{current}$, the previous ray parameter $t_{prev}$, the step size $\Delta t$, and the volume data structure.
*   Output: The interpolated voxel value at $t_{current}$.

**Data Structures:**
*   `cached_value`: A scalar or vector holding the voxel value sampled at the previous iteration's end point.
*   `cached_t`: A scalar holding the ray parameter $t$ corresponding to `cached_value`.

**Steps:**
1.  **Calculate Previous End Point:** Determine the end point of the previous sample. If this is the first iteration, no cache exists. Otherwise, the previous end point is $t_{prev} + \Delta t$.
2.  **Check Cache Hit:** Compare the current sampling position $t_{current}$ with the cached position $t_{cached}$.
    *   If $t_{current} == t_{cached}$ (within floating-point tolerance), the value is already in `cached_value`. Return `cached_value`.
    *   Otherwise, perform a texture lookup or grid traversal to fetch the value at $t_{current}$.
3.  **Update Cache:** Store the fetched value in `cached_value` and the current parameter $t_{current}$ in `cached_t`.
4.  **Return Value:** Return the fetched value.

**Formulas:**
Let $V(t)$ be the volume function.
Let $t_i$ be the parameter for the $i$-th sample.
Let $\Delta t$ be the step size.
$t_i = t_{i-1} + \Delta t$.

The cache hit condition is:
$$ t_i == t_{i-1} + \Delta t $$

If the condition holds, $V(t_i)$ is retrieved from cache. Otherwise, $V(t_i)$ is computed via trilinear interpolation or higher-order scheme, and stored in cache for the next iteration ($i+1$).

**Parameters:**
*   Step size $\Delta t$: Typically determined by the desired sampling density or adaptive sampling criteria.
*   Tolerance: A small epsilon for floating-point comparison, e.g., $10^{-6}$.

**Edge Cases and Pitfalls:**
*   **First Iteration:** The cache must be initialized or marked as invalid. [G] Initialize `cached_t` to $-\infty$ or use a boolean flag `cache_valid = false`.
*   **Variable Step Size:** If $\Delta t$ varies per iteration (adaptive sampling), the cache hit condition must account for the specific $\Delta t$ used in the previous step. The cache stores the absolute position $t$, so variable steps do not break the logic, but the overlap must be exact.
*   **Precision Errors:** Floating-point inaccuracies may cause $t_i$ to differ slightly from $t_{i-1} + \Delta t$. Use a small tolerance or round to the nearest representable voxel coordinate if the volume is discrete.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_volume_value(t_current, t_prev, delta_t, volume_data, cache):
    """
    Retrieve volume value with caching.
    
    Args:
        t_current: Current ray parameter.
        t_prev: Previous ray parameter.
        delta_t: Step size used in previous iteration.
        volume_data: Volume data structure (e.g., texture handle, grid pointer).
        cache: Object containing 'value' and 't' fields.
        
    Returns:
        Interpolated volume value at t_current.
    """
    # Calculate expected position of previous sample's end
    t_prev_end = t_prev + delta_t
    
    # Check if cache is valid and position matches
    if cache.is_valid and abs(cache.t - t_prev_end) < 1e-6:
        # Cache hit: reuse previous sample's end value
        return cache.value
    
    # Cache miss or invalid: fetch new value
    # Perform trilinear interpolation or higher-order sampling at t_current
    value = volume_data.sample(t_current)
    
    # Update cache for next iteration
    cache.value = value
    cache.t = t_current
    cache.is_valid = True
    
    return value
```

## Subtopics

### [VDB Value Accessor Caching](8756-vdb-value-accessor-caching.md)
VDB Value Accessor Caching is a mechanism for coordinate-based value retrieval in OpenVDB structures that achieves average constant time complexity by exploiting the fixed four-level tree depth and spatial coherence of access patterns. Unlike traditional top-down tree traversal, this method effectively performs a bottom-up versus top-down traversal, which is very fast for typical access patterns like Finite-Difference stencils that are spatially coherent.

### Volume File Caching
Volume File Caching manages the loading and sharing of volume grid data from disk to reduce I/O and memory duplication [S2]. It operates as follows:
*   **On-Demand Loading:** Volume datablocks with a filepath are loaded on-demand during modifier evaluation, drawing, UI updates, or Python access [S2].
*   **Two-Stage Load:** `BKE_volume_load()` loads the list of grids, names, metadata, and transforms. `BKE_volume_grid_load()` loads the grid tree and voxel data into memory [S2].
*   **Global Cache:** A global volume file cache shares grid memory between original and Copy-on-Write (CoW) datablocks, and between different volume datablocks referencing the same filepath. Only file-loaded grids are cached; procedurally generated grids are not [S2].
*   **Reference Counting:** Grid memory is freed immediately when the user count drops to zero [S2].
*   **Metadata vs. Tree Separation:** The cache distinguishes between metadata users and tree users. If tree users drop to zero, the grid tree is freed, but metadata remains in memory for metadata users [S2].
*   **Access Functions:** Specific functions like `BKE_volume_grid_openvdb_for_metadata()` and `BKE_volume_grid_openvdb_for_read()` ensure the correct level of data is loaded and locked [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample after the first fetch, assuming a cache hit. The cost of a cache miss is the cost of one volume lookup (texture fetch or tree traversal).
*   **Memory Complexity:** $O(1)$ additional memory per ray or per thread, storing only the last sample's value and parameter.
*   **Performance:** Reduces memory bandwidth usage by approximately 50% for standard first-order integration with constant step size, as every other sample is a cache hit [S1]. For second-order pre-integration, the number of texture accesses remains $N$, but the back sample is cached for the next iteration, maintaining efficiency [S1].
*   **Strengths:** Simple to implement, low overhead, effective for coherent sampling patterns.
*   **Limitations:** Only effective when consecutive samples overlap or are adjacent. Ineffective for random access or highly adaptive sampling with large, non-overlapping steps.
*   **When to Use:** Direct volume rendering with ray marching, especially with constant or smoothly varying step sizes.
*   **When Not to Use:** Random access volume queries, or when memory bandwidth is not the bottleneck (e.g., compute-bound transfer function evaluations).

## Background
Volume Access Caching is a standard optimization in direct volume rendering, often implicit in implementations of pre-integration. The specific mention of caching the "back sample" for the next iteration is highlighted in the context of Second Order Pre-Integrated Volume Rendering [S1]. The technique leverages the fact that pre-integration or higher-order schemes often require evaluating the volume at points that are shared between consecutive integration steps. The concept is distinct from file-level caching, which is a data management strategy described in Blender's volume system [S2].

## Sources
[S1] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
[S2] Volume - Blender Developer Documentation, https://developer.blender.org/docs/features/objects/volume/, fetched 2026-10-27
