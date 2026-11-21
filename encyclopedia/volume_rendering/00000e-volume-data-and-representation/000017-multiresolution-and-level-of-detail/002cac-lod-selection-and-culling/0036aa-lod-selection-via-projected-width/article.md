# LoD Selection via Projected Width

Also called: PathTracer MipMap selection

## Overview
LoD Selection via Projected Width determines the appropriate resolution level for volume rendering by calculating the screen-space footprint of the data, analogous to mip-map selection in path tracing [S1]. It solves the problem of static LOD allocation by adapting resolution to view-dependent importance based on the projected width of the volume elements on the camera plane [S1]. Unlike distance-based methods which rely on Euclidean distance from the camera, this approach uses the actual screen-space size to ensure consistent visual fidelity and anti-aliasing regardless of viewing angle or perspective distortion [S1].

## How it works
**Inputs:**
*   Camera view and projection matrices.
*   Volume geometry or bounding box defining the spatial extent of the data.
*   Base resolution of the finest LOD level.

**Outputs:**
*   Selected LOD level index for each rendering sample or voxel lookup.

**Data Structures:**
*   The method assumes a hierarchy of pre-computed volume representations (mip levels) [S1].
*   Each level $i$ has a resolution $N_i$. The hierarchy is typically generated such that $N_{i+1} = N_i / 2$ or similar scaling factor [G].

**Steps:**
1.  **Load Volumes:** Load multiple computed LoD volumes simultaneously into memory, one per LoD level [S1].
2.  **Calculate Projected Width:** For the region of interest (e.g., a voxel or brick), calculate its projected width $w$ in pixels on the camera viewport. This involves projecting the spatial extent of the volume element through the view-projection matrix to screen space [S1].
3.  **Determine LOD Level:** Select the LOD level based on the projected width $w$. The goal is to choose the coarsest level where the voxel still covers at least one pixel, or conversely, the finest level where the voxel does not significantly exceed one pixel to avoid aliasing. This follows the "usual PathTracer MipMap selection" logic [S1].
4.  **Lookup Data:** Use the selected LOD level to lookup data from the corresponding volume [S1].

**Formulas:**
Let $w$ be the projected width of the voxel in pixels. Let $L$ be the LOD level index, where $L=0$ is the finest resolution.
The selection metric typically compares $w$ against the pixel coverage of the voxel at each level. If the base voxel size is $v_0$, the size at level $L$ is $v_L = v_0 \cdot 2^L$. The projected width at level $L$ is $w_L = w \cdot 2^{-L}$.
The optimal level $L^*$ is often chosen such that $w_{L^*} \approx 1$ pixel, or bounded by a range to prevent over-sampling or under-sampling.
Specific implementation details for the exact thresholding function are not provided in the sources, but the principle is to map the continuous projected width to a discrete integer LOD index [S1].

**Parameters:**
*   **Base Resolution:** The resolution of the finest LOD level.
*   **Sample Number:** The number of samples per pixel, which interacts with the LOD selection to manage memory and performance [S1].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** Combining elevated sample numbers with high resolutions can lead to out-of-memory problems. The user must balance resolution and sample count based on available GPU memory [S1].
*   **Sparsity:** The choice of base resolutions should adjust to the sparsity of the input model to ensure sufficient detail where needed while saving memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_via_projected_width(camera, volume_bounding_box, lod_levels):
    """
    Selects the LOD level for a volume element based on its projected width.
    
    Args:
        camera: Object containing view and projection matrices.
        volume_bounding_box: Bounding box of the volume element in world space.
        lod_levels: List of pre-computed volume resolutions [N0, N1, ...].
        
    Returns:
        int: The index of the selected LOD level.
    """
    # 1. Calculate the projected width of the volume element in pixels
    # Project the bounding box corners to screen space
    corners = project_to_screen(volume_bounding_box, camera.view_matrix, camera.projection_matrix)
    
    # Calculate the width in pixels (max x - min x)
    projected_width = max(c.x for c in corners) - min(c.x for c in corners)
    
    # 2. Determine the required LOD level based on projected width
    # The goal is to match the voxel size to the pixel size
    # Assuming a standard mip-map chain where each level halves the resolution
    # We want the projected width to be approximately 1 pixel for the selected level
    
    selected_level = 0
    min_diff = float('inf')
    
    for level_idx, resolution in enumerate(lod_levels):
        # Calculate the effective projected width at this LOD level
        # If the base resolution is N0, and current is N_level, the scale factor is N0 / N_level
        # The projected width scales inversely with resolution
        scale_factor = lod_levels[0] / resolution
        effective_width = projected_width / scale_factor
        
        # We want effective_width to be close to 1.0 (or a specific threshold)
        # A simple heuristic is to pick the level where the voxel covers ~1 pixel
        diff = abs(effective_width - 1.0)
        
        if diff < min_diff:
            min_diff = diff
            selected_level = level_idx
            
        # Optimization: if effective_width < 1.0, we might be over-sampling, 
        # but we usually pick the finest level that doesn't under-sample significantly.
        # The exact thresholding logic depends on the specific "usual PathTracer MipMap selection" [S1]
        
    return selected_level

def render_with_lod(camera, volume_data, lod_levels, selected_level):
    """
    Renders the volume using the selected LOD level.
    """
    # Load multiple computed LoDs simultaneously (one per LoD level) [S1]
    active_volume = lod_levels[selected_level]
    
    # Choose which volume to lookup for data at each voxel based on required LoD level [S1]
    for each voxel in view:
        data = active_volume.lookup(voxel_coordinates)
        render(data)
```

## Subtopics

### [Distance-Based Level Selection](14162-distance-based-level-selection.md)
Distance-Based Level Selection is a mechanism that selects the octree resolution level for each anchor point based on its Euclidean distance from the camera, ensuring closer areas use higher detail and distant areas use simplified representations. It solves the problem of static LOD allocation by adapting resolution to view-dependent importance. Unlike LoD Selection via Projected Width, which relies on screen-space footprint, this method uses geometric distance to determine the appropriate octree level, providing a view-dependent hierarchy that balances rendering cost and visual fidelity.

## Operating characteristics
*   **Time Complexity:** The selection process is $O(1)$ per voxel or brick, as it involves a simple projection and lookup. However, loading multiple LOD volumes simultaneously increases memory bandwidth requirements [S1].
*   **Memory Complexity:** Requires storing all LOD levels in memory simultaneously if dynamic switching is performed per-voxel, which can lead to out-of-memory problems if the resolution and sample count are too high [S1].
*   **Strengths:** Provides view-dependent LOD that adapts to perspective distortion and viewing angle, ensuring consistent visual quality across the screen [S1].
*   **Limitations:** Requires careful tuning of base resolutions and sample counts to avoid memory issues, especially for sparse models [S1]. The exact selection metric is not fully specified, relying on "usual" path tracer heuristics [S1].

## Background
This method is described in the context of fast voxelization and level of detail for microgeometry rendering, such as vegetation and hair models [S1]. It leverages the concept of mip-map selection from path tracing to determine the appropriate resolution for volume data based on its projected size on the camera [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
[S2] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
