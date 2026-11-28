# Streaming and Level-of-Detail (LOD) selection

Also called: additive ladders, frustum-culled streaming

## Overview
Streaming and Level-of-Detail (LOD) Selection is a method for visualizing massive volumetric scenes by adaptively loading data based on viewpoint position, bounding memory usage to only visible data and the required resolution [S1, unofficial]. It sets itself apart from static LOD selection by dynamically culling invisible blocks and adapting resolution based on the camera's distance and the block's projected size, ensuring that only necessary data resides in memory [S1, unofficial]. Unlike [CP-LOD](9646-cp-lod-coverage-preserving-prefix-lod.md), which focuses on coverage-preserving prefix structures, this method emphasizes the temporal management of data transfer and memory reuse to maintain navigation fluidity during viewpoint changes [S1, unofficial].

## How it works
The method operates on a scene graph where data is organized into spatial tiles and additive ladders [S1, unofficial]. The core mechanism involves streaming data in prefix order and selecting the coarsest substitutive level that satisfies a projected footprint constraint, while frustum-culling partition tiles to prevent fetching full detail for coarse or off-screen regions [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A compiled archive containing Gaussian splat data organized in chunks, along with metadata defining additive ladders, splat parameters, and chunk bounds [S1, unofficial].
*   **Output:** A rendered view where data is streamed progressively, with resolution adapted to the viewer's position and screen area [S1, unofficial].

**Data Structures:**
The scene is stored in a hierarchical archive structure containing spatial tiles and additive ladders [S1, unofficial]. Primitive data includes splat centers, covariance parameters (e.g., Cholesky factors), amplitudes, and chunk bounds [S1, unofficial]. The archive supports multi-level caching of decoded geometry, decoded chunks, compressed chunks, and origin-private file system storage [S1, unofficial].

**Algorithm Steps:**
1.  **Scene Initialization:** Load the scene archive [S1, unofficial]. Initialize a multi-level cache comprising decoded per-slice geometry, decoded chunks in memory, compressed chunks in memory, compressed chunks in the browser’s origin-private file system, and HTTP range requests [S1, unofficial].
2.  **LOD Ordering:** Order the additive ladders for efficient streaming [S1, unofficial]. Use greedy ordering for $N \le 5,000$ splats, and $O(N \log N)$ self-energy ordering for $N > 5,000$ splats [S1, unofficial].
3.  **Frustum Culling:** Identify partition tiles that are off-screen or coarse [S1, unofficial]. These regions are never fetched at full detail [S1, unofficial].
4.  **LOD Selection:** For each visible tile/splat group, select the LOD level [S1, unofficial]. Calculate the projected median splat footprint [S1, unofficial]. Select the coarsest substitutive level where the projected median splat footprint $\le 1.5$ CSS pixels [S1, unofficial]. If no such level exists, select the finest level [S1, unofficial]. Apply a 10% hysteresis against coarsening to prevent flickering [S1, unofficial]. Fallback: If no footprint is stamped or stamped dimensions are not displayed, fall back to the screen-area fraction the node’s bounding box projects to [S1, unofficial].
5.  **Streaming and Caching:** Fetch data in prefix order [S1, unofficial]. After an on-disk hit or remote fetch, a prefetcher requests the adjacent chunk in the ordering [S1, unofficial]. An idle-time prefetcher warms the adjacent slice to hide remote-fetch latency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(footprints, current_level, hysteresis=0.1):
    """
    Selects LOD level based on projected footprint and hysteresis.
    footprints: list of floats, footprints for each level (0=finest, N=coarsest)
    current_level: int, current LOD index
    hysteresis: float, 0.1 for 10% against coarsening
    Returns: int, new LOD level index
    """
    max_footprint = 1.5
    target_level = None
    for level in range(len(footprints)):
        if footprints[level] <= max_footprint:
            target_level = level
    if target_level is None:
        target_level = 0  # Finest only when none does

    # Apply 10% hysteresis against coarsening
    if target_level > current_level:
        # Moving to coarser level
        # Check if current level is still acceptable with margin
        if footprints[current_level] <= max_footprint * (1.0 - hysteresis):
            return current_level
    return target_level

def greedy_ordering(splats):
    """Greedy selection up to 5000 splats."""
    return sorted(range(len(splats)), key=lambda i: splats[i].area, reverse=True)

def self_energy_ordering(splats):
    """O(N log N) self-energy ordering for >5000 splats."""
    return sorted(range(len(splats)), key=lambda i: splats[i].self_energy)

def frustum_cull(tiles, camera):
    """Returns tiles intersecting the camera frustum."""
    return [t for t in tiles if t.bbox.intersects(camera.frustum)]

def calculate_projected_footprint(tile, camera):
    """Returns projected median splat footprint in CSS pixels."""
    return tile.median_splat_size * camera.pixel_scale

def fetch_chunk(level, archive):
    """Fetches chunk data from archive or cache."""
    return archive.get_chunk(level)

def cache_store(level, data):
    """Stores data in multi-level cache."""
    pass  # Implementation details omitted

def stream_scene(scene_archive, camera_view):
    splat_count = scene_archive.get_splat_count()
    if splat_count <= 5000:
        ordering = greedy_ordering(scene_archive.splats)
    else:
        ordering = self_energy_ordering(scene_archive.splats)

    visible_tiles = frustum_cull(scene_archive.tiles, camera_view)
    for tile in visible_tiles:
        footprint = calculate_projected_footprint(tile, camera_view)
        lod_level = select_lod_level(tile.footprints, tile.current_lod)
        tile.lod_level = lod_level

    for level in ordering:
        data = fetch_chunk(level, scene_archive)
        cache_store(level, data)

    # Prefetch adjacent chunks and slices
    for i in range(len(ordering) - 1):
        scene_archive.prefetch(ordering[i + 1])
    scene_archive.prefetch_adjacent_slice()
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

### [Level-of-Detail Grammar](3113-level-of-detail-grammar.md)
Level-of-Detail Grammar is a system for managing scenes that exceed rendering budgets by using additive ladders, substitutive levels, and spatial tiles to adjust detail based on data arrival, distance, and screen position. It uses additive ladders to paint a coarse scene from the first chunk and sharpen it as more data arrives, and uses substitutive levels and spatial tiles to reduce the cost of distant or off-screen objects. It is used to handle scenes that outgrow a single rendering budget, enabling scalable interactive web visualization.

### Additive levels of detail for Gaussian splats
Additive levels of detail for Gaussian splats is an order-optimisation framework for streaming progressive loading of Gaussian splats, building additive ladders where each rung adds detail without replacing previous data [S1, unofficial]. It computes a closed-form quadratic-assignment AUC to evaluate splat ordering, calculates a monotone-submodular utility function on the splat Gram matrix, and applies a greedy selection algorithm that guarantees a uniform $(1 - 1/e)$ approximation bound [S1, unofficial]. It is implemented using sparse-Gram storage and sparse-greedy selection to optimize memory and compute [S1, unofficial].

### StreamingFactor LOD Control
StreamingFactor controls LOD granularity and memory usage; higher values (e.g., 2.0) yield finer distant LODs and higher SSIM (~0.88-0.92) with fewer active chunks, while lower values reduce VRAM usage [S2]. At StreamingFactor 2.0 on San Miguel (8K), average SSIM is 0.888 with 341 max active chunks; at 2.4, SSIM is 0.924 with 470 chunks [S2]. At StreamingFactor 2.0 on Sponza (8K), average SSIM is 0.8 with 473 max active chunks; at 2.4, SSIM is 0.86 with 731 chunks [S2]. At StreamingFactor 2.0 on Greenfield (8K), average SSIM is 0.866 with 130 max active chunks; at 2.4, SSIM is 0.891 with 168 chunks [S2].

## Operating characteristics
*   **Time Complexity:** LOD ordering is $O(N)$ for greedy (up to 5,000 splats) and $O(N \log N)$ for self-energy ordering (above 5,000 splats) [S1, unofficial].
*   **Memory Complexity:** Bounded by the visible data and required resolution, with multi-level caching (decoded geometry, decoded chunks, compressed chunks, origin-private file system) [S1, unofficial].
*   **Measured Performance:** The Drosophila scene opens in 19.06 MB, against 5.27 MB at the compiler’s 64 KB chunks, which in turn make a step cost 266–268 requests [S1, unofficial].
*   **Strengths:** Scalable from a handful of primitives to millions of points and splats while remaining interactive; hides network latency with prefetching; adaptive resolution based on screen area [S1, unofficial].
*   **Limitations:** Requires a compiled archive structure; relies on HTTP range requests and browser caching; self-energy ordering has no approximation guarantee [S1, unofficial].
*   **When to Use:** For large-scale volumetric scenes that exceed local rendering budgets and require interactive web visualization [S1, unofficial].
*   **When Not to Use:** For small scenes where static LOD is sufficient; for scenes where strict approximation guarantees are required for all ordering methods [S1, unofficial].

## Background
The method originates from the Luxar system, which provides Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data [S1, unofficial]. The system uses a level-of-detail grammar to manage additive ladders and substitutive levels, and a client-side cache with prefetching to hide network latency [S1, unofficial]. The viewer is implemented in WebGL or WebGPU, with HDR tone mapping and nD sliders for time, channel, and any hidden axis [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games, https://doi.org/10.1145/3728299, fetched 2026-10-26
