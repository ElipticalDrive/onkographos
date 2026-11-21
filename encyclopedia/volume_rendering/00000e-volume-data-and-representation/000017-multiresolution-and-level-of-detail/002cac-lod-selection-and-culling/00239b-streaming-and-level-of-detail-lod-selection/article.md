# Streaming and Level-of-Detail (LOD) selection

Also called: additive ladders, frustum-culled streaming

## Overview
Streaming and Level-of-Detail (LOD) Selection is a method for visualizing massive volumetric scenes that exceed local rendering or memory budgets by adaptively loading data based on viewpoint position, bounding memory usage to only visible data and the required resolution [S1, unofficial]. It sets itself apart from static LOD selection by dynamically culling invisible blocks and adapting resolution based on the camera's distance and the block's projected size, ensuring that only necessary data resides in memory [S1, unofficial]. Unlike [CP-LOD](9646-cp-lod-coverage-preserving-prefix-lod.md), which focuses on coverage-preserving prefix structures, this method emphasizes the temporal management of data transfer and memory reuse to maintain navigation fluidity during viewpoint changes [S1, unofficial].

## How it works
The method operates on a scene graph where data is organized into spatial tiles and additive ladders. The core mechanism involves streaming data in prefix order and selecting the coarsest substitutive level that satisfies a projected footprint constraint, while frustum-culling partition tiles to prevent fetching full detail for coarse or off-screen regions [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A compiled archive (e.g., `.luxar.zarr`) containing Gaussian splat data organized in chunks, along with metadata defining additive ladders, centers, cholesky matrices, amplitudes, and chunk bounds [S1, unofficial].
*   **Output:** A rendered view where data is streamed progressively, with resolution adapted to the viewer's position and screen area [S1, unofficial].

**Data Structures:**
The scene is stored in a hierarchical archive structure. A typical `.luxar.zarr` archive includes:
*   `zarr.json`: Metadata for the Zarr store.
*   `scene/`: Scene dimensions and configuration.
*   `nuclei/` (or other primitive types):
    *   `centers`: Splat positions.
    *   `cholesky`: Splat covariance/shape parameters.
    *   `amplitudes`: Splat intensities.
    *   `chunk_bounds`: Boundaries for spatial tiling.
    *   `additive_0 ... additive_3`: LOD groups representing additive levels [S1, unofficial].
*   `cells/`, `tracks/`, `surface/`, `overlays/`: Other scene elements [S1, unofficial].

**Algorithm Steps:**
1.  **Scene Initialization:** Load the scene archive. The viewer initializes a multi-level cache including decoded per-slice geometry, decoded chunks in memory, compressed chunks in memory, and compressed chunks in the browser’s origin-private file system [S1, unofficial].
2.  **LOD Ordering:** Order the additive ladders for efficient streaming.
    *   If the number of splats $N \le 5,000$, use greedy ordering [S1, unofficial].
    *   If $N > 5,000$, use $O(N \log N)$ self-energy ordering [S1, unofficial].
3.  **Frustum Culling:** Identify partition tiles that are off-screen or coarse. These regions are never fetched at full detail [S1, unofficial].
4.  **LOD Selection:** For each visible tile/splat group, select the LOD level.
    *   Calculate the projected median splat footprint.
    *   Select the coarsest substitutive level where the projected median splat footprint $\le 1.5$ CSS pixels [S1, unofficial].
    *   If no such level exists, select the finest level [S1, unofficial].
    *   Apply a 10% hysteresis against coarsening to prevent flickering [S1, unofficial].
    *   **Fallback:** If no footprint is stamped or stamped dimensions are not displayed, fall back to the screen-area fraction the node’s bounding box projects to [S1, unofficial].
5.  **Streaming and Caching:** Fetch data in prefix order. After an on-disk hit or remote fetch, a prefetcher requests the adjacent chunk in the ordering. An idle-time prefetcher warms the adjacent slice to hide remote-fetch latency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(splat_footprint, current_level, hysteresis_threshold=0.1):
    """
    Selects the LOD level based on projected footprint and hysteresis.
    
    Args:
        splat_footprint (float): Projected median splat footprint in CSS pixels.
        current_level (int): Current LOD level index (0 is finest, N is coarsest).
        hysteresis_threshold (float): Hysteresis factor against coarsening.
        
    Returns:
        int: New LOD level index.
    """
    # Threshold for coarsest acceptable level
    max_footprint = 1.5
    
    # Determine target level based on footprint
    # Assume levels are ordered such that higher index = coarser
    # Find coarsest level where footprint <= max_footprint
    target_level = None
    for level in range(num_levels):
        if get_footprint_for_level(level) <= max_footprint:
            target_level = level
            
    if target_level is None:
        # Finest only when none does
        target_level = 0
        
    # Apply hysteresis against coarsening
    # If we are currently at a finer level than target, we might want to stay
    # unless the footprint is significantly smaller
    if target_level > current_level:
        # Moving to coarser level
        # Check if we should stay finer due to hysteresis
        # Hysteresis of 10% against coarsening implies we need a stronger signal
        # to switch to coarser, or we stay at current if close
        # Simplified logic: if target is coarser, ensure significant difference
        if current_level < target_level:
            # Check if current level is still acceptable with some margin
            if get_footprint_for_level(current_level) <= max_footprint * (1 - hysteresis_threshold):
                return current_level
                
    return target_level

def stream_scene(scene_archive, camera_view):
    """
    Streams scene data based on LOD selection and frustum culling.
    """
    # 1. Frustum cull partition tiles
    visible_tiles = frustum_cull(scene_archive.tiles, camera_view)
    
    # 2. Order additive ladders
    splat_count = scene_archive.get_splat_count()
    if splat_count <= 5000:
        ordering = greedy_ordering(scene_archive.additive_ladders)
    else:
        ordering = self_energy_ordering(scene_archive.additive_ladders)
        
    # 3. Select LOD for each visible tile
    for tile in visible_tiles:
        footprint = calculate_projected_footprint(tile, camera_view)
        lod_level = select_lod_level(footprint, tile.current_lod)
        tile.lod_level = lod_level
        
    # 4. Fetch data in prefix order
    for level in ordering:
        data = fetch_chunk(level, scene_archive)
        cache_store(level, data)
        
    # 5. Prefetch adjacent chunks
    prefetch_adjacent_chunks(scene_archive, ordering)
```

**Parameters:**
*   **Footprint Threshold:** 1.5 CSS pixels [S1, unofficial].
*   **Hysteresis:** 10% against coarsening [S1, unofficial].
*   **LOD Ordering Switch:** 5,000 splats (greedy below, self-energy above) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Footprint:** If no footprint is stamped or stamped dimensions are not displayed, the system falls back to screen-area fraction [S1, unofficial].
*   **Precision:** The hysteresis mechanism prevents rapid switching between LOD levels, which could cause visual flickering [S1, unofficial].
*   **Boundary:** The switch between greedy and self-energy ordering at 5,000 splats is a hard threshold [S1, unofficial].

## Subtopics

### [Dynamic Loading and Progressive Update](4577-dynamic-loading-and-progressive-update.md)
Dynamic Loading and Progressive Update is a strategy for visualizing massive volumetric scenes by adaptively loading data based on the viewpoint position, bounding memory usage to only the visible data and the required resolution ("useful resolution"). It sets itself apart from static LOD selection by dynamically culling invisible blocks and adapting resolution based on the camera's distance and the block's projected size, ensuring that only necessary data resides in memory. Unlike [CP-LOD](9646-cp-lod-coverage-preserving-prefix-lod.md), which focuses on coverage-preserving prefix structures, this method emphasizes the temporal management of data transfer and memory reuse to maintain navigation fluidity during viewpoint changes.

### [CP-LOD (Coverage-Preserving Prefix LOD)](9646-cp-lod-coverage-preserving-prefix-lod.md)
CP-LOD is a runtime selection method that traverses a factree on the GPU to produce a leaf-only cut bounded by a capacity budget $K$, using Morton-strided subsampling to preserve visual coverage. It differs from naive threshold schedules by being resolution-invariant by construction, producing identical materialized sets at different resolutions (e.g., $800^2$ and $1920 \times 1080$).

### Level-of-Detail Grammar
Level-of-Detail Grammar is a system for managing scenes that exceed rendering budgets by using additive ladders, substitutive levels, and spatial tiles to adjust detail based on data arrival, distance, and screen position [S1, unofficial]. It uses additive ladders to paint a coarse scene from the first chunk and sharpen it as more data arrives, and uses substitutive levels and spatial tiles to reduce the cost of distant or off-screen objects [S1, unofficial]. The `.luxar.zarr` archive structure includes LOD groups: `additive_0 ... additive_3` for additive levels, alongside `centers`, `cholesky`, `amplitudes`, and `chunk_bounds` [S1, unofficial]. It is used to handle scenes that outgrow a single rendering budget, enabling scalable interactive web visualization [S1, unofficial].

### Additive levels of detail for Gaussian splats
Additive levels of detail for Gaussian splats is an order-optimisation framework for streaming progressive loading of Gaussian splats, building additive ladders where each rung adds detail without replacing previous data [S1, unofficial]. It computes a closed-form quadratic-assignment AUC to evaluate splat ordering, calculates a monotone-submodular utility function on the splat Gram matrix, and applies a greedy selection algorithm that guarantees a uniform $(1 - 1/e)$ approximation bound [S1, unofficial]. It is implemented using sparse-Gram storage and sparse-greedy selection to optimize memory and compute [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** LOD ordering is $O(N)$ for greedy (up to 5,000 splats) and $O(N \log N)$ for self-energy ordering (above 5,000 splats) [S1, unofficial].
*   **Memory Complexity:** Bounded by the visible data and required resolution, with multi-level caching (decoded geometry, decoded chunks, compressed chunks, origin-private file system) [S1, unofficial].
*   **Measured Performance:** The Drosophila scene opens in 19.06 MB, against 5.27 MB at the compiler’s 64 KB chunks, which in turn make a step cost 266–268 requests [S1, unofficial].
*   **Strengths:** Scalable from a handful of primitives to millions of points and splats while remaining interactive; hides network latency with prefetching; adaptive resolution based on screen area [S1, unofficial].
*   **Limitations:** Requires a compiled archive structure; relies on HTTP range requests and browser caching; self-energy ordering has no approximation guarantee [S1, unofficial].
*   **When to Use:** For large-scale volumetric scenes that exceed local rendering budgets and require interactive web visualization [S1, unofficial].
*   **When Not to Use:** For small scenes where static LOD is sufficient; for scenes where strict approximation guarantees are required for all ordering methods [S1, unofficial].

## Background
The method originates from the Luxar system, which provides Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data [S1, unofficial]. It was developed to handle scenes that outgrow a single rendering budget, enabling scalable interactive web visualization [S1, unofficial]. The system uses a level-of-detail grammar to manage additive ladders and substitutive levels, and a client-side cache with prefetching to hide network latency [S1, unofficial]. The viewer is implemented in WebGL or WebGPU, with HDR tone mapping and nD sliders for time, channel, and any hidden axis [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
