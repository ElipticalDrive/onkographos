# Volume Access Caching


## Overview
Volume Access Caching is an optimization for direct volume rendering that reduces redundant memory bandwidth by caching the volume value at the end of one ray-marching step for use as the starting value of the next step [S1]. It solves the problem of repeatedly fetching overlapping voxel data between consecutive iterations along a single ray, distinguishing itself from spatial caching methods like [Edge Point Caching](891-edge-point-caching.md) by exploiting temporal coherence along the ray [G].

## How it works
**Inputs and Outputs:**
*   Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, step size $\Delta t$, volume data structure, and number of steps $N$.
*   Output: Accumulated radiance or density along the ray.

**Data Structures:**
*   `cached_value`: A scalar or vector holding the volume value sampled at the previous iteration's end point.
*   `t`: A scalar tracking the current ray parameter.

**Steps:**
1.  **Initialize Cache:** Sample the volume at the ray origin $\mathbf{o}$ and store the result in `cached_value`. Set $t = 0$.
2.  **Iterate:** For each step $i$ from $0$ to $N-1$:
    a. Use `cached_value` as the current sample at parameter $t$.
    b. Perform the required integration or compositing operation using the current sample.
    c. Advance the ray parameter: $t \leftarrow t + \Delta t$.
    d. Sample the volume at the new position $\mathbf{o} + t\mathbf{d}$ and store the result in `cached_value` for the next iteration.
3.  **Return:** Output the accumulated result.

**Formulas:**
Let $V(\mathbf{p})$ be the volume function.
Let $t_i$ be the ray parameter at step $i$, with $t_0 = 0$ and $t_{i+1} = t_i + \Delta t$.
The cache hit condition is implicit in the loop structure: the value $V(\mathbf{o} + t_i\mathbf{d})$ is retrieved from `cached_value` at the start of iteration $i$, and $V(\mathbf{o} + t_{i+1}\mathbf{d})$ is computed and cached at the end of iteration $i$.
The total number of texture accesses per ray is exactly $N$, independently of the integration order [S1].

**Parameters:**
*   Step size $\Delta t$: Determines sampling density; typically derived from voxel size or adaptive criteria [G].
*   Number of steps $N$: Determines ray length coverage [G].

**Edge Cases and Pitfalls:**
*   **First Iteration:** The cache must be explicitly initialized with the ray origin sample before the loop begins [G].
*   **Floating-Point Drift:** Accumulating $t$ directly can cause positional drift over many steps. Recompute the sampling position as $\mathbf{o} + t\mathbf{d}$ each iteration to maintain alignment with voxel coordinates [G].
*   **Early Exit:** If the ray exits the volume bounds before $N$ steps, the loop must terminate early and the accumulated result returned [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_with_back_sample_cache(origin, direction, step_size, volume_data, num_steps):
    """
    Ray-marches through a volume using back-sample caching.
    Matches the optimization described in [S1]: exactly N texture accesses,
    caching the back sample for the next iteration.
    """
    t = 0.0
    # Step 1: Initialize cache with the first sample at the ray origin
    cached_value = volume_data.sample(origin)
    
    accumulated = 0.0
    
    # Step 2: Iterate N times
    for _ in range(num_steps):
        # Step 2a: Use cached value as current sample
        # Step 2b: Perform integration/compositing (simplified accumulation)
        accumulated += cached_value
        
        # Step 2c: Advance ray parameter
        t += step_size
        
        # Step 2d: Sample back point for next iteration and update cache
        next_pos = origin + direction * t
        cached_value = volume_data.sample(next_pos)
        
    # Step 3: Return result
    return accumulated
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
*   **Time Complexity:** $O(N)$ total per ray, with $O(1)$ work per step after initialization [S1].
*   **Memory Complexity:** $O(1)$ additional memory per ray or thread, storing only the last sample's value and parameter [G].
*   **Performance:** Reduces texture accesses to exactly $N$ per ray, matching the theoretical minimum for sequential sampling, independently of the pre-integration order [S1].
*   **Strengths:** Minimal overhead, highly effective for coherent ray-marching patterns, and simplifies shader logic by eliminating redundant lookups [G].
*   **Limitations:** Only beneficial when consecutive samples overlap or are adjacent; ineffective for random access queries or highly discontinuous sampling patterns [G].
*   **When to Use:** Direct volume rendering with ray marching, particularly with constant or smoothly varying step sizes [G].
*   **When Not to Use:** Random access volume queries, or when memory bandwidth is not the bottleneck [G].

## Background
Volume Access Caching originates from direct volume rendering optimizations for pre-integration schemes, specifically highlighted in Second Order Pre-Integrated Volume Rendering [S1]. The technique leverages the mathematical property that higher-order integration or pre-integration requires evaluating the volume at points shared between consecutive integration steps, making the back sample inherently reusable [S1]. It is a standard implementation detail in modern direct volume renderers and is distinct from file-level data management strategies [S2].

## Sources
[S1] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
[S2] Volume - Blender Developer Documentation, https://developer.blender.org/docs/features/objects/volume/, fetched 2026-10-27
