# Ray-Guided Grid Leaping

Also called: Out-of-core ray-guided rendering, Ray-guided volume rendering

## Overview
Ray-Guided Grid Leaping is an out-of-core volume rendering technique that couples the rendering process with the identification of which sub-volumes ('bricks') must be loaded, using ray-sampling characteristics to determine future data requirements in concert with standard ray traversal and accumulation [S1]. It differs from its sibling [Greedy Brick Request Strategy](2843-greedy-brick-request-strategy.md) by performing brick lookup once per brick rather than every sample, maintaining a simple ray-marching core, and utilizing an out-of-core, progressive rendering methodology that breaks the GPU-memory-size barrier [S1].

## How it works
**Inputs:**
*   Input volume data existing at a single resolution.
*   Transfer function.
*   Camera parameters.

**Outputs:**
*   Rendered image.
*   Updated GPU texture pool with paged-in bricks.

**Data Structures:**
1.  **Multiresolution Hierarchy:** A preprocess built from the single-resolution input data, consisting of simple multiresolution Cartesian grids (bricks) [S1].
2.  **Lookup Table:** Stores the status for each brick index. It has three states:
    *   Empty
    *   Non-empty and present on the GPU
    *   Non-empty and absent [S1]
3.  **GPU Hash Table:** A lock-free hash table used to record missing bricks for feedback to the I/O layer [S1].

**Parameters:**
*   **Pixel Error Threshold:** The level of detail is computed to maintain a pixel error of less than one [S1].

**Algorithm Steps:**
1.  **Preprocessing:** Build a multiresolution hierarchy as a preprocess for input data which exist at only one resolution [S1].
2.  **Ray Processing (Per Ray):**
    a.  Compute the level of detail (LOD) required to maintain a pixel error of less than one [S1].
    b.  Compute a brick index from the LOD and the position in the volume [S1].
    c.  Use the brick index to fetch information from the lookup table to identify the brick's state:
        i.  If the brick is **empty**, skip the brick and repeat the process at the brick’s exit point [S1].
        ii. If the brick is **non-empty and present** on the GPU, ray-cast that brick [S1].
        iii. If the brick is **non-empty and absent** from GPU memory, return the finest coarser level available and add the missing entry to the GPU hash table [S1].
3.  **End of Frame:**
    a.  Read back the GPU hash table to host memory [S1].
    b.  Use the read-back data to page in bricks from disk or cache [S1].
    c.  Upload paged-in bricks to a GPU texture pool [S1].
4.  **Subsequent Frame:** Use the portion of the brick pool for sampling [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing step (CPU)
def build_multiresolution_hierarchy(single_resolution_data):
    """
    Builds a multiresolution hierarchy of bricks from single-resolution input.
    Returns a structure allowing lookup of brick data at various LODs.
    """
    # Implementation details for bricking and LOD generation are not provided in sources.
    pass

# Per-frame rendering step (GPU Kernel)
def ray_guided_grid_leaping_ray_kernel(ray, lookup_table, brick_pool, missing_bricks_hash_table):
    """
    Processes a single ray using Ray-Guided Grid Leaping.
    """
    # Initialize ray marching state
    t = ray.origin
    color = vec4(0.0)
    alpha = 0.0
    
    # Ray marching loop
    while t is within volume bounds:
        # 1. Compute Level of Detail (LOD)
        # The LOD is computed to maintain a pixel error of less than one.
        lod = compute_lod_for_pixel_error_threshold(ray, t, threshold=1.0)
        
        # 2. Compute Brick Index
        # Derived from LOD and current position in volume.
        brick_idx = compute_brick_index(lod, t)
        
        # 3. Lookup Brick Status
        status = lookup_table.get(brick_idx)
        
        if status == EMPTY:
            # Skip empty brick and jump to exit point
            t = get_brick_exit_point(brick_idx, t)
            continue
            
        elif status == NON_EMPTY_PRESENT:
            # Ray-cast the present brick
            brick_data = brick_pool.get(brick_idx)
            color, alpha, t = ray_cast_brick(ray, t, brick_data, color, alpha)
            
        elif status == NON_EMPTY_ABSENT:
            # Return finest coarser level available
            # Add missing entry to GPU hash table for I/O feedback
            missing_bricks_hash_table.insert(brick_idx)
            
            # Use coarser level for sampling
            coarser_brick_idx = get_coarser_brick(brick_idx)
            coarser_data = brick_pool.get(coarser_brick_idx)
            color, alpha, t = ray_cast_brick(ray, t, coarser_data, color, alpha)
            
    return color

# End-of-frame processing (CPU/GPU Sync)
def end_of_frame_processing(missing_bricks_hash_table, brick_pool, disk_cache):
    """
    Handles I/O for missing bricks identified during the frame.
    """
    # 1. Read back GPU hash table to host memory
    missing_bricks = missing_bricks_hash_table.read_back_to_host()
    
    # 2. Page in bricks from disk or cache
    paged_bricks = []
    for brick_idx in missing_bricks:
        data = disk_cache.load(brick_idx)
        paged_bricks.append((brick_idx, data))
        
    # 3. Upload paged-in bricks to GPU texture pool
    for brick_idx, data in paged_bricks:
        brick_pool.upload(brick_idx, data)
        
    # 4. Clear hash table for next frame
    missing_bricks_hash_table.clear()
```

**Edge Cases and Pitfalls:**
*   **Precision:** The level of detail calculation must ensure pixel error remains below one to maintain visual fidelity while optimizing performance [S1].
*   **Boundaries:** When skipping empty bricks, the algorithm must correctly calculate the exit point to resume marching [S1].
*   **Missing Data:** If a brick is non-empty but absent, the system must gracefully degrade to the finest coarser level available rather than failing [S1].

## Subtopics

### Greedy Brick Request Strategy
A strategy where a ray requests intermediate levels of detail (bricks) along its path when a resolution is missing, flooding the hash table with requests. This produces more responsive, iteratively-refined images and converges in fewer frames. [Link](2843-greedy-brick-request-strategy.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The renderer is designed to render massive datasets extremely fast on commodity GPU hardware [S1]. It breaks the GPU-memory-size barrier that limits data sizes in prior work [S1].
*   **Scaling:** The methodology allows graceful scaling down to consumer-level graphics cards [S1].
*   **Strengths:**
    *   Performs brick lookup once per brick instead of every sample, maintaining a simple and familiar ray-marching core [S1].
    *   Utilizes modern GPU features to implement a lock-free feedback data structure, enabling the implementation to spend more time computing on the GPU and less time pushing data around [S1].
    *   Out-of-core, progressive rendering methodology breaks the GPU-memory-size barrier [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When rendering massive datasets on commodity GPU hardware where GPU memory is insufficient to hold the entire volume [S1].
*   **When not to use:** Not given in the sources.

## Background
Ray-Guided Grid Leaping is reminiscent of the recent work of Hadwiger et al. [2] and Engel’s CERA-TVR [26], which in turn is based on the Gigavoxels system [3] [S1]. It shares with Hadwiger et al. the requirement of a set of simple multiresolution Cartesian grids and an OpenGL-based table to report missing bricks [S1]. From the CERA-TVR system, it inherits the idea to only recompute and request grid cells at boundaries [S1]. The work provides new depth to discussions of parameters relevant in the development of a ray-guided direct volume renderer, including strategies for loading higher resolution data, issues surrounding bricking and brick sizes, and the effect of brick layout strategies on large volume access times [S1].

## Sources
[S1] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
