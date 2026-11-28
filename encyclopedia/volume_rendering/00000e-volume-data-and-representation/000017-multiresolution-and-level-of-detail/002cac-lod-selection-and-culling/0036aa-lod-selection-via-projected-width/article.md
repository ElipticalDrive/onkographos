# LoD Selection via Projected Width

Also called: PathTracer MipMap selection

## Overview
LoD Selection via Projected Width determines the appropriate resolution level for volume rendering by calculating the screen-space footprint of the data, analogous to mip-map selection in path tracing [S1]. It solves the problem of static LOD allocation by adapting resolution to view-dependent importance based on the projected width of the volume elements on the camera plane [S1].

## How it works
**Inputs:**
*   Camera view and projection matrices.
*   Volume geometry or bounding box defining the spatial extent of the data [G].
*   List of pre-computed LOD resolutions [G].

**Outputs:**
*   Selected LOD level index for each rendering sample or voxel lookup [S1].

**Data Structures:**
*   The method assumes a hierarchy of pre-computed volume representations (mip levels) [S1].
*   Each level $i$ has a resolution $N_i$ [G].

**Steps:**
1.  **Load Volumes:** Load multiple computed LoD volumes simultaneously into memory, one per LoD level [S1].
2.  **Determine LOD Level:** Determine the required LoD level using a metric based on projected width on the camera, following usual PathTracer MipMap selection [S1].
3.  **Lookup Data:** Choose which volume to lookup for data at each voxel based on the required LoD level [S1].

**Formulas:**
Not given in the sources. The exact metric/threshold is not specified, relying on "usual" PathTracer MipMap selection heuristics [S1].

**Parameters:**
*   **First Level Resolution:** Controls the final resolution and must be chosen to adjust to the sparsity of the input model [S1].
*   **Second Level Resolution:** Controls the final resolution and must be chosen to adjust to the sparsity of the input model [S1].
*   **Sample Number:** Controls the final resolution and must be chosen to adjust to the sparsity of the input model [S1].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** Combining elevated sample numbers with high resolutions can lead to out-of-memory problems [S1].
*   **Sparsity:** Users must freely balance resolution and sample count to fit available GPU memory while preserving detail [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_lod_level(projected_width_pixels: float, lod_resolutions: list[int]) -> int:
    """
    Determines the required LoD level using a metric based on projected width on the camera,
    following usual PathTracer MipMap selection [S1].
    
    Args:
        projected_width_pixels: Width of the volume element in screen pixels.
        lod_resolutions: List of resolutions for each LOD level [N0, N1, ...].
        
    Returns:
        int: Index of the selected LOD level.
    """
    # The exact metric/threshold is not provided in the sources [S1].
    # Standard mip-map selection maps screen-space size to a discrete level index [G].
    base_resolution = lod_resolutions[0]
    selected_level = 0
    
    for level_idx, resolution in enumerate(lod_resolutions):
        # Scale projected width to the current resolution level
        # Not given in the sources. Standard scaling assumes resolution halves per level [G].
        scaled_width = projected_width_pixels * (base_resolution / resolution)
        
        # Select the coarsest level where the voxel covers approximately one pixel
        if scaled_width <= 1.0:
            selected_level = level_idx
            break
            
    return selected_level

def render_with_selected_lod(camera_matrices, volume_data, lod_levels, selected_level: int):
    """
    Loads multiple computed LoD volumes simultaneously and chooses which volume to lookup
    for data at each voxel based on the required LoD level [S1].
    """
    active_volume = lod_levels[selected_level]
    for voxel in volume_data.voxels:
        data = active_volume.lookup(voxel.coordinates)
        # Render data...
        pass
```

## Subtopics

### [Distance-Based Level Selection](14162-distance-based-level-selection.md)
Distance-Based Level Selection is a mechanism that selects the octree resolution level for each anchor point based on its Euclidean distance from the camera, ensuring closer areas use higher detail and distant areas use simplified representations. It sets itself apart from projected width methods by relying on geometric distance rather than screen-space footprint. Choose it when view-independent spatial partitioning is preferred or when camera projection is unavailable.

### [LOD Selection](22744-lod-selection.md)
This method provides seamless Level-of-Detail (LOD) for volume rendering by prefiltering a mixture of Gabor kernels based on camera parameters and depth. It selects the minimal set of primitives matching a desired frequency limit, achieved by masking out or culling Gabor kernels that would exceed the rendering bandwidth. It sets itself apart by using analytical Gabor Fields instead of discrete voxel levels, avoiding the high costs of Gaussian filtering or the quality degradation of pruning-based bandlimiting. Choose it when seamless frequency-domain LOD is required.

### Cube Map LOD Selection via Projected Corners
Calculates optimal cube-map mip level and ray sample count based on viewport projection of cube corners to maintain anti-aliasing and performance [S2, unofficial]. It projects 8 cube corners to viewport using `ProjectToViewport(i, worldViewProj, viewport)` [S2, unofficial]. It calculates ideal cube-map resolution: $s = \text{EstimateCubeEdgePixelSize}(v) / \text{upscale}$, where $\text{upscale} = 2.0$ [S2, unofficial]. It calculates ideal ray sample amount: $\text{raySampleAmt} = \text{raySampleCountScale} \times s / \sqrt{3.0}$, where $\text{raySampleCountScale} = 2.0$ [S2, unofficial]. It clamps ray sample count: $\text{raySampleCount} = \min(\text{raySampleCnt}, \text{raySampleCount})$, where $\text{raySampleCnt} = \lceil \text{raySampleAmt} \rceil$ [S2, unofficial]. It recalculates resolution from clamped samples: $s = \text{raySampleAmt} / \text{raySampleCountScale} \times \sqrt{3.0}$ [S2, unofficial]. It computes LOD level: $\text{level} = \max(\log_2(\text{cubeMapSize} / s), 0.0)$, returning $\min(\text{uint8\_t}(\text{level}), \text{numMips} - 1)$ [S2, unofficial].

## Operating characteristics
*   **Memory Complexity:** Loading multiple LOD volumes simultaneously requires memory proportional to the number of levels [S1]. Combining elevated sample numbers with high resolutions can lead to out-of-memory problems [S1].
*   **Strengths:** Adapts resolution to view-dependent importance based on projected width, ensuring consistent visual fidelity across the screen [S1].
*   **Limitations:** The pipeline exposes three main parameters (first level resolution, second level resolution, and sample number) that users must balance to fit GPU memory and adjust to sparsity [S1]. The exact selection metric relies on unspecified "usual" PathTracer MipMap selection heuristics [S1].

## Background
This method is described in the context of fast voxelization and level of detail for microgeometry rendering, such as vegetation and hair models [S1]. It leverages the concept of mip-map selection from path tracing to determine the appropriate resolution for volume data based on its projected size on the camera [S1].

## Sources
[S1] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
[S2] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
