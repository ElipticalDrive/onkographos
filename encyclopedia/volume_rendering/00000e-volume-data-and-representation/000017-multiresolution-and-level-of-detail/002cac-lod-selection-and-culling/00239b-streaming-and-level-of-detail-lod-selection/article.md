# Streaming and Level-of-Detail (LOD) selection

Also called: additive ladders, frustum-culled streaming

## Overview
Streaming and Level-of-Detail (LOD) selection is a method for visualizing massive scenes that exceed rendering or memory budgets by dynamically selecting data resolution based on viewpoint position and screen footprint [S1, unofficial]. It solves the problem of navigating arbitrarily large datasets without stalling the frame rate or exceeding memory limits by combining additive ladders (progressive detail) with substitutive levels (coarse approximations) and spatial culling [S1, unofficial]. It sets itself apart from its siblings by explicitly managing the temporal aspect of data arrival and resolution selection through a "level-of-detail grammar" that prioritizes visual coverage preservation during streaming, whereas methods like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) focus on static hierarchy traversal, and [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md) relies on fixed spatial bricks rather than splat-based footprints.

## How it works
The method operates on Gaussian splats organized into an additive ladder structure within a compressed archive (e.g., `.luxar.zarr`). The core logic involves selecting the coarsest substitutive level that satisfies a screen-space footprint constraint, applying hysteresis to prevent flickering, and frustum-culling irrelevant spatial tiles.

### Inputs and Outputs
*   **Input:** A scene archive containing Gaussian splats with attributes: centers ($\mu$), Cholesky factors ($L$ for covariance), and amplitudes ($a$). The archive is structured with LOD groups (`additive_0` to `additive_3`), `centers`, `cholesky`, `amplitudes`, and `chunk_bounds` [S1, unofficial].
*   **Output:** A subset of splats streamed and rendered at the appropriate resolution level, along with prefetched adjacent chunks to hide network latency [S1, unofficial].

### Data Structures
The scene is stored in a Zarr-based format. Key fields include:
*   `additive_k`: LOD groups representing additive ladders. Each rung adds detail without replacing previous data [S1, unofficial].
*   `centers`: 3D coordinates of splat centers.
*   `cholesky`: Lower triangular Cholesky decomposition of the splat covariance matrix.
*   `amplitudes`: Opacity or intensity values.
*   `chunk_bounds`: Spatial boundaries for tiling and culling [S1, unofficial].

### Selection Algorithm
1.  **Ordering:** Additive ladders are ordered to optimize loading. For $N \le 5,000$ splats, a greedy ordering is used. For $N > 5,000$, an $O(N \log N)$ self-energy ordering is applied [S1, unofficial].
2.  **Footprint Calculation:** For each splat or tile, calculate the projected median splat footprint in CSS pixels.
3.  **Level Selection:** Select the coarsest substitutive level where the projected median splat footprint $\le 1.5$ CSS pixels [S1, unofficial].
    *   If no level satisfies the condition, select the finest level.
    *   Apply a 10% hysteresis against coarsening to prevent visual flickering when the camera moves slightly [S1, unofficial].
4.  **Fallback:** If no footprint is stamped or stamped dimensions do not match the display, fall back to the screen-area fraction of the node's bounding box [S1, unofficial].
5.  **Culling:** Frustum-cull partition tiles. Coarse or off-screen regions are never fetched at full detail [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(splats, camera_view, threshold_pixels=1.5, hysteresis=0.1):
    """
    Select the coarsest LOD level for a set of splats based on projected footprint.
    
    Args:
        splats: List of splat objects with 'footprint' (in CSS pixels) or 'bbox' (bounding box).
        camera_view: Camera projection matrix and viewport dimensions.
        threshold_pixels: Maximum allowed footprint size (1.5).
        hysteresis: Hysteresis factor against coarsening (0.1).
        
    Returns:
        int: Index of the selected LOD level.
    """
    # Calculate projected footprints for all splats
    footprints = []
    for splat in splats:
        if 'footprint' in splat:
            footprints.append(splat['footprint'])
        else:
            # Fallback: screen-area fraction of bounding box
            projected_area = project_bbox_to_screen(splat['bbox'], camera_view)
            footprints.append(projected_area)
            
    if not footprints:
        return 0  # Default to finest if no data

    median_footprint = median(footprints)
    
    # Determine candidate levels based on threshold
    # Assume levels are sorted coarsest to finest, or vice versa.
    # Here we assume level 0 is coarsest, level N is finest.
    # We want the coarsest level (lowest index) where footprint <= threshold.
    
    candidate_level = None
    for level_idx, level_splats in enumerate(get_ladder_levels()):
        # Calculate footprint for this level's representative splats
        level_footprint = calculate_level_footprint(level_splats, camera_view)
        
        if level_footprint <= threshold_pixels:
            candidate_level = level_idx
            break
            
    # If no level satisfies the condition, use the finest level
    if candidate_level is None:
        candidate_level = get_max_level_index()
        
    # Apply hysteresis against coarsening
    # If the current level is finer than candidate, check if we should stay fine
    current_level = get_current_level()
    if current_level < candidate_level:
        # Moving from fine to coarse
        # Allow coarsening only if the footprint is significantly smaller than threshold
        # Hysteresis: require footprint to be <= threshold * (1 - hysteresis) to coarsen?
        # Or: require current level to be sufficiently coarser?
        # Source says "10% hysteresis against coarsening".
        # Interpretation: Do not coarsen unless the new level is 10% coarser in footprint?
        # Or: The threshold for coarsening is stricter.
        # Let's implement as: only coarsen if new footprint <= threshold * (1 - hysteresis)
        if median_footprint > threshold_pixels * (1 - hysteresis):
            return current_level  # Stay at current finer level
            
    return candidate_level

def project_bbox_to_screen(bbox, camera_view):
    """Project a 3D bounding box to screen space and return area fraction."""
    corners = transform_corners(bbox, camera_view)
    screen_rect = get_screen_rect(corners)
    return screen_rect.area / total_screen_area()
```

### Parameters
*   **Footprint Threshold:** 1.5 CSS pixels [S1, unofficial].
*   **Hysteresis:** 10% against coarsening [S1, unofficial].
*   **LOD Ordering Switch:** 5,000 splats (greedy below, self-energy above) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Missing Footprint:** If footprint data is not stamped or dimensions mismatch, the fallback to screen-area fraction may be less accurate for non-spherical splats [S1, unofficial].
*   **Precision:** Floating-point precision in footprint calculation may cause flickering near the threshold boundary; hysteresis mitigates this [S1, unofficial].
*   **Degenerate Input:** Empty splat lists or zero-area bounding boxes should be handled by defaulting to the finest level or skipping culling [G].

## Subtopics

### [Dynamic Loading and Progressive Update](4577-dynamic-loading-and-progressive-update.md)
Dynamic Loading and Progressive Update is a strategy for visualizing massive scenes by dynamically and adaptively loading data based on viewpoint position, bounding memory usage to visible data and required resolution. It solves the problem of navigating arbitrarily large datasets without exceeding memory limits or stalling the frame rate. It sets itself apart from its sibling [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md) by explicitly addressing the temporal aspect of data updates: while the parent selects the coarsest substitutive level based on projected splat footprint, this method focuses on the incremental update of the data structure (brick resolution and N3-tree subdivision) and spreading heavy loads across multiple frames to maintain real-time fluidity.

### [CP-LOD (Coverage-Preserving Prefix LOD)](9646-cp-lod-coverage-preserving-prefix-lod.md)
CP-LOD is a runtime selection method that traverses a factree on the GPU to produce a leaf-only cut bounded by a capacity budget $K$, using Morton-strided subsampling to preserve visual coverage. It differs from naive threshold schedules by being resolution-invariant by construction, producing identical materialized sets at different resolutions (e.g., $800^2$ and $1920 \times 1080$).

## Operating characteristics
*   **Time Complexity:** LOD ordering is $O(N \log N)$ for large scenes ($N > 5,000$) and greedy for smaller scenes [S1, unofficial]. Selection is $O(1)$ per frame after footprint precomputation.
*   **Memory Complexity:** Bounded by the visible data and required resolution. The archive structure allows chunked loading, reducing peak memory usage [S1, unofficial].
*   **Measured Performance:** The system scales from a handful of primitives to millions of splats while remaining interactive. For example, a Drosophila scene with 2.4M splats opens in 19.06 MB, leveraging 64 KB chunks for efficient streaming [S1, unofficial].
*   **Strengths:** Handles scenes that outgrow a single rendering budget; enables scalable interactive web visualization; hides network latency with prefetching [S1, unofficial].
*   **Limitations:** Requires pre-computed additive ladders and footprint stamps; greedy ordering is only optimal up to 5,000 splats [S1, unofficial].
*   **When to Use:** Ideal for web-based visualization of large scientific datasets (e.g., microscopy) where network bandwidth and client memory are constrained [S1, unofficial].
*   **When Not to Use:** When real-time, unstructured data arrival is required without pre-computation, or when strict offline rendering is needed [G].

## Background
The method originates from the Luxar system, introduced in a 2026 preprint by the authors of "Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data" [S1, unofficial]. It was developed to address the challenges of visualizing multidimensional scientific data (e.g., 2.4M-splat frames) in web browsers, leveraging WebGL/WebGPU and Zarr-based storage. The "level-of-detail grammar" and additive ladders were designed to provide an axis of economy for scenes that exceed rendering budgets, complementing the spatial economy of substitutive levels and tiles [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
