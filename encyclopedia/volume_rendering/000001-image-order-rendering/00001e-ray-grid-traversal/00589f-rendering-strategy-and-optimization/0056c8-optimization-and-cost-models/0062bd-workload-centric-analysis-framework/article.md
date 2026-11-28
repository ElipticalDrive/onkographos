# Workload-Centric Analysis Framework

Also called: 5-count model, N-resident to N-blend pipeline

## Overview
The Workload-Centric Analysis Framework is a methodological tool for connecting individual rendering optimizations to their systemic effects by tracing the forward rendering path through five specific workload counts: $N_{resident}$, $N_{active}$, $N_{pair}$, $N_{eval}$, and $N_{blend}$ [S1, unofficial]. It solves the problem of predicting how a specific optimization (e.g., culling or prefiltering) impacts overall performance by accounting for the non-linear ratios between stages, which vary with scene content, viewpoint, resolution, and backend semantics [S1, unofficial]. Unlike sibling methods that focus on hardware-specific instructions or grid resolution costs, this framework provides a backend-agnostic abstraction of the Gaussian Splatting rendering pipeline to isolate the cost of algorithmic decisions from hardware execution details [S1, unofficial].

## How it works
The framework models the rendering pipeline as a sequence of five stages. Each stage produces a count representing a specific subset of work items. The counts are defined as follows:

1.  **$N_{resident}$**: The count of Gaussians stored in the rendering memory pool and directly accessible to the renderer. The memory tier (device-local, unified, or host-mapped) must be reported [S1, unofficial]. In hierarchical or streamed systems, Gaussians outside the measured rendering pool are excluded [S1, unofficial].
2.  **$N_{active}$**: The count of Gaussians retained after frame selection, visibility culling, and projected-footprint tests. This is the post-projection count of distinct Gaussians producing at least one materialized association. It excludes rejected inputs that still incurred Projection work [S1, unofficial].
3.  **$N_{pair}$**: The count of Gaussian–tile association records created during the Sorting stage, where projected footprints are expanded. One active Gaussian can generate several records. Depth ordering reorganizes these records without changing their number [S1, unofficial].
4.  **$N_{eval}$**: The count of Gaussian response or contribution tests executed by pixel threads traversing tile lists. A rejected candidate still counts because its test has already run [S1, unofficial].
5.  **$N_{blend}$**: The count of contributions that pass the tests and update pixel color and transmittance. Within the same frame and backend, $N_{blend} \le N_{eval}$ [S1, unofficial].

**Formulas and Bounds**
The theoretical lower bound for evaluation cost is reached when $N_{eval} = N_{blend}$, meaning every non-contributing candidate is rejected before evaluation [S1, unofficial]. Ideal lossless prefiltering leaves $N_{blend}$ unchanged [S1, unofficial].

**Parameters and Pitfalls**
*   **Memory Tier**: The definition of $N_{resident}$ depends on the memory tier. If the tier is not reported, the count is ambiguous [S1, unofficial].
*   **Non-Monotonic Ratios**: Reducing one count (e.g., $N_{active}$) does not necessarily reduce later counts (e.g., $N_{eval}$) proportionally. The ratios between successive counts vary with scene content, viewpoint, resolution, tile configuration, selection/culling policy, and backend semantics [S1, unofficial].
*   **Prefiltering Cost**: Work performed by prefiltering must be accounted for separately from the savings in $N_{eval}$ or $N_{blend}$ [S1, unofficial].
*   **System Cost**: The five counts describe stored Gaussians and rendering operations, but system cost also depends on work per item, transfers, synchronization, temporary buffers, cached results, and optimizer data. Backward computation and parameter updates add further cost [S1, unofficial].

**Pseudocode**
The following pseudocode illustrates the logical flow of the counts. It assumes a standard Gaussian Splatting backend where Gaussians are projected, sorted, and evaluated per pixel.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analyze_workload(gaussians, camera, resolution, tile_size):
    # Stage 1: Resident Count
    # N_resident is the total number of Gaussians in the accessible memory pool.
    # This is an input parameter, not computed by the renderer loop.
    N_resident = len(gaussians)
    
    # Stage 2: Active Count
    # Perform frame selection, visibility culling, and projected-footprint tests.
    active_gaussians = []
    projection_work_count = 0
    
    for g in gaussians:
        projection_work_count += 1
        if is_visible(g, camera) and has_footprint(g, camera, resolution):
            active_gaussians.append(g)
            
    N_active = len(active_gaussians)
    
    # Stage 3: Pair Count
    # Expand active Gaussians into Gaussian-tile association records.
    # One Gaussian may generate multiple records if it overlaps multiple tiles.
    association_records = []
    
    for g in active_gaussians:
        tiles = get_overlapping_tiles(g, camera, resolution, tile_size)
        for tile in tiles:
            association_records.append((g, tile))
            
    N_pair = len(association_records)
    
    # Sort records by depth (optional, does not change N_pair)
    association_records.sort(key=lambda x: x[0].depth)
    
    # Stage 4: Evaluation Count
    # Traverse tile lists and execute contribution tests.
    N_eval = 0
    
    # Iterate over all pixels in the tile
    for tile in unique_tiles(association_records):
        for pixel in tile.pixels:
            # Get list of Gaussians associated with this pixel via the tile
            candidates = get_candidates_for_pixel(pixel, association_records)
            
            for candidate in candidates:
                N_eval += 1
                # Execute Gaussian response/contribution test
                contribution = evaluate_gaussian(candidate, pixel)
                
                # Stage 5: Blend Count
                # If the contribution passes the test (e.g., alpha > threshold)
                if is_contributing(contribution):
                    blend_pixel(pixel, contribution)
                    
    # Note: N_blend is implicitly the number of successful blends.
    # In a real implementation, this would be tracked explicitly.
    N_blend = count_successful_blends()
    
    return {
        "N_resident": N_resident,
        "N_active": N_active,
        "N_pair": N_pair,
        "N_eval": N_eval,
        "N_blend": N_blend
    }
```

## Operating characteristics
*   **Time Complexity**: The framework itself has no computational complexity; it is an analytical model. However, it highlights that the cost of the rendering pipeline is dominated by the stage with the highest work-per-item cost multiplied by its count. Since $N_{eval}$ can be significantly larger than $N_{blend}$, optimizations targeting $N_{eval}$ (like prefiltering) are critical [S1, unofficial].
*   **Memory Complexity**: $N_{resident}$ determines the memory footprint. The memory tier (device-local, unified, host-mapped) affects access latency and bandwidth, which are not captured by the count alone but are essential for performance modeling [S1, unofficial].
*   **Strengths**: It provides a standardized vocabulary for comparing optimizations across different backends and hardware. It exposes the disconnect between culling effectiveness ($N_{active}$) and actual evaluation savings ($N_{eval}$) [S1, unofficial].
*   **Limitations**: It does not account for work-per-item variations, synchronization overhead, or backward computation costs. It assumes a specific forward path (Splats to Silicon) and may not apply to other volume rendering techniques [S1, unofficial].
*   **When to Use**: When analyzing the impact of algorithmic changes (culling, sorting, prefiltering) on overall rendering performance, especially in Gaussian Splatting systems.
*   **When Not to Use**: When hardware-specific bottlenecks (e.g., memory bandwidth saturation, cache misses) dominate over algorithmic workload, or when analyzing non-Gaussian volume rendering methods.

## Background
The Workload-Centric Analysis Framework was introduced in the paper "From Splats to Silicon: Rethinking Computational Efficiency of 3DGS" [S1, unofficial]. It emerged from the need to systematically evaluate the efficiency of 3D Gaussian Splatting rendering pipelines, which involve complex interactions between projection, sorting, and evaluation stages. The framework addresses the gap in understanding how individual optimizations propagate through the rendering stack, providing a structured way to measure and compare computational efficiency [S1, unofficial].

## Sources
[S1] From Splats to Silicon: Rethinking Computational Efficiency of 3DGS, https://arxiv.org/abs/2609.06157, fetched 2026-10-29 (unofficial)
