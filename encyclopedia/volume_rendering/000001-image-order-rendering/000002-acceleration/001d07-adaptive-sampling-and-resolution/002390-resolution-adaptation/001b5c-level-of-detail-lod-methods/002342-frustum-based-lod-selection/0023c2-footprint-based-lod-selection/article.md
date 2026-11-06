# Footprint-based LOD Selection

Also called: Additive Ladder Level Selection, Coarsest substitutive level selection

## Overview
Footprint-based LOD Selection determines the appropriate level of detail for volumetric entities by ensuring that the projected size of a voxel (or splat) on the screen does not exceed a specific pixel threshold, thereby preventing aliasing and excessive memory bandwidth usage. Unlike its parent [Frustum-based LOD Selection](2986-frustum-based-lod-selection.md), which relies on distance to frustum faces, this method uses the actual projected footprint of the data, selecting the coarsest level in an additive ladder that satisfies the constraint. It is distinguished from [Cone-Footprint LOD and Step Sizing](5178-cone-footprint-lod-and-step-sizing.md) by its use of a fixed median footprint threshold rather than sample density proportional to screen content.

## How it works
**Inputs:**
*   A list of nodes/chunks in an additive ladder structure.
*   Camera projection matrix and viewport dimensions.
*   Stamped footprint dimensions for each node, or bounding box geometry.
*   Ordering of splats (Greedy or Self-energy).

**Outputs:**
*   Selected LOD level index for each node.
*   Fetch requests for the corresponding data chunks.

**Data Structures:**
*   **Node:** Contains bounding box, stamped footprint dimensions (width, height), and LOD level index.
*   **Additive Ladder:** A hierarchical structure where each level contains a subset of the data, ordered for efficient streaming.

**Steps:**
1.  **Project Footprint:** For each node, calculate the projected size of its footprint on the screen. If stamped footprint dimensions are available and match the display context, use them. Otherwise, calculate the screen-area fraction of the node’s bounding box [S1, unofficial].
2.  **Check Constraint:** Compare the projected median splat footprint against the threshold of 1.5 CSS pixels [S1, unofficial].
3.  **Select Level:**
    *   Select the coarsest substitutive level in the additive ladder whose projected median splat footprint is $\le 1.5$ CSS pixels [S1, unofficial].
    *   If no level satisfies the constraint, select the finest level [S1, unofficial].
4.  **Apply Hysteresis:** To prevent flickering between levels during camera movement, apply a 10% margin against coarsening. This means a level is only selected if the footprint is sufficiently smaller than the threshold to justify the switch, preventing rapid toggling [S1, unofficial].
5.  **Frustum Cull:** Frustum-cull partition tiles so that coarse or off-screen regions are never fetched at full detail [S1, unofficial].
6.  **Ordering:**
    *   If the number of splats is $\le 5,000$, use Greedy ordering [S1, unofficial].
    *   If the number of splats is $> 5,000$, use $O(N \log N)$ self-energy ordering [S1, unofficial].
    *   Note: Greedy ordering lowers the residual area under the loading curve by 0.2–2.6% relative to self-energy ordering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(node, camera, viewport, threshold=1.5, hysteresis_margin=0.10):
    """
    Selects the LOD level for a node based on its projected footprint.
    
    Args:
        node: Object containing bounding_box, stamped_footprint (optional), and levels (list of LOD levels).
        camera: Camera object with projection_matrix and view_matrix.
        viewport: Viewport dimensions (width, height).
        threshold: Maximum allowed footprint in CSS pixels (default 1.5).
        hysteresis_margin: Margin against coarsening (default 0.10).
        
    Returns:
        int: Index of the selected LOD level.
    """
    # Step 1: Calculate projected footprint
    if node.stamped_footprint and node.stamped_footprint.matches_display():
        projected_footprint = node.stamped_footprint.project(camera, viewport)
    else:
        # Fallback to bounding box screen-area fraction
        projected_footprint = node.bounding_box.project_screen_area(camera, viewport)
        
    # Step 2 & 3: Find coarsest level satisfying constraint
    selected_level_index = -1
    for level_index, level in enumerate(node.levels):
        # Calculate median splat footprint for this level
        median_footprint = level.calculate_median_splat_footprint(projected_footprint)
        
        if median_footprint <= threshold:
            selected_level_index = level_index
            break
            
    # If no level satisfies, select finest
    if selected_level_index == -1:
        selected_level_index = len(node.levels) - 1
        
    # Step 4: Apply hysteresis against coarsening
    # If we are considering a coarser level (lower index), ensure it is significantly smaller
    # to prevent flickering. This is a simplified representation of the 10% margin.
    if selected_level_index < current_level_index:
        # Check if the footprint is small enough to justify coarsening
        # The exact implementation of the 10% margin depends on the specific state tracking
        # of the previous frame's level.
        if not is_stable_for_coarsening(selected_level_index, current_level_index, hysteresis_margin):
            selected_level_index = current_level_index
            
    return selected_level_index

def is_stable_for_coarsening(new_level, old_level, margin):
    """
    Checks if coarsening is stable based on hysteresis margin.
    [G] Implementation detail: Typically involves comparing the projected size
    against a threshold scaled by (1 - margin).
    """
    # Placeholder for specific hysteresis logic not fully detailed in sources
    # Assuming a simple check: new level's footprint must be < threshold * (1 - margin)
    # relative to the old level's stability.
    return True # [G] Simplified for completeness; actual logic requires state
```

**Parameters:**
*   **Footprint Threshold:** 1.5 CSS pixels [S1, unofficial].
*   **Hysteresis Margin:** 10% against coarsening [S1, unofficial].
*   **Ordering Threshold:** 5,000 splats (Greedy below, Self-energy above) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **No Stamped Footprint:** If no footprint is stamped or dimensions do not match, fall back to screen-area fraction of the bounding box [S1, unofficial].
*   **No Satisfying Level:** If no level meets the footprint constraint, the finest level is selected [S1, unofficial].
*   **Flickering:** Without hysteresis, rapid changes in camera distance can cause flickering between LOD levels. A 10% margin is applied to prevent this [S1, unofficial].

## Subtopics

### [Cone-Footprint LOD and Step Sizing](5178-cone-footprint-lod-and-step-sizing.md)
A technique replacing fixed step counts with sample density proportional to screen content (cone footprint) rather than camera distance, to reduce stationary-shimmer/aliasing. It differs from Footprint-based LOD Selection by using a dynamic footprint based on cone projection rather than a fixed median splat threshold. Choose this when aliasing in static scenes is a primary concern. See [Cone-Footprint LOD and Step Sizing](5178-cone-footprint-lod-and-step-sizing.md).

### LOD Brightness Compensation
Applies a per-chunk exponent to counteract brightness loss from downsampling averaging, which destroys color-opacity correlation. The exponent multiplies the gamma exponent. The formula is $exponent = base\_exponent + \log_2(downsample) \times scale\_factor$. The old law used lift proportional to $(downsample - 1)$, which was incorrect because averaging reduces the same 2x2x2 neighborhood per level, making it one comparable step per level. Default refit is 0.08 per level. Seam response is monotonic across the whole slider (-33.0% at 0, -10.1% at 0.08). Pitfall: Old form caused level-6 bricks to ask for exponent -0.386 on a seven-level pyramid; the 0.25 clamp caught it, causing artifacts. Measured +35.7% at default against -34.0% with compensation off, overshooting null by 6x. Floor sat pinned at clamp, causing pale flash on zoom. [S2, unofficial]

## Operating characteristics
*   **Time Complexity:** Level selection is $O(L)$ per node, where $L$ is the number of levels. Ordering is $O(N)$ for Greedy (up to 5,000 splats) and $O(N \log N)$ for Self-energy (above 5,000 splats) [S1, unofficial].
*   **Memory Complexity:** Depends on the additive ladder structure and caching strategy (multi-level cache with decoded per-slice geometry, decoded chunks, compressed chunks, and HTTP range requests) [S1, unofficial].
*   **Performance:** Greedy ordering lowers the residual area under the loading curve by 0.2–2.6% relative to self-energy ordering [S1, unofficial].
*   **Strengths:** Prevents aliasing by controlling projected footprint size; reduces memory bandwidth by fetching only necessary detail; hysteresis prevents visual flickering.
*   **Limitations:** Requires stamped footprint data for optimal performance; fallback to bounding box may be less accurate; ordering complexity increases for large scenes.
*   **When to Use:** Interactive visualization of large volumetric datasets where aliasing and memory bandwidth are concerns.
*   **When Not to Use:** When stamped footprint data is unavailable and bounding box fallback is insufficient for quality requirements.

## Background
Footprint-based LOD Selection is used in scalable interactive web visualization of multidimensional scientific data, specifically in the Luxar project for Gaussian splatting microscopy data [S1, unofficial]. It is part of a broader system that includes additive ladders, multi-level caching, and prefetching to handle large datasets efficiently [S1, unofficial]. The method was developed to address the challenges of streaming and rendering large volumetric data in web browsers, where memory and bandwidth are limited [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
