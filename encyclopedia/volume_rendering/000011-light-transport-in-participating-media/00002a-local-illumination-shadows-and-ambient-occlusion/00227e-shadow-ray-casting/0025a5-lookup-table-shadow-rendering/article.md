# Lookup Table Shadow Rendering

Also called: One-pass Shadow with Lookup Tables, Shadow Rendering, VolPack Shadow Algorithm

## Overview
Lookup Table Shadow Rendering is a fast, one-pass shadowing algorithm implemented within the VolPack volumetric rendering library [S1]. It solves the problem of computing shadows for participating media by pre-computing shadow contributions into lookup tables, avoiding the per-ray cost of traditional shadow ray casting during the main rendering pass [S1]. It is distinguished from its siblings by its reliance on pre-computed tables for immediate shading, whereas methods like [Shadow Ray Casting](2796-shadow-ray-casting.md) compute shadows dynamically per sample, and [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md) focuses on local occlusion rather than directional light shadows.

## How it works
**Inputs and Outputs**
*   **Input:** Volumetric data (voxel grid), light source parameters, and a configured shadow lookup shader [S1].
*   **Output:** A rendered image with volumetric shadows applied [S1].

**Data Structures**
The method relies on lookup tables (LUTs) that map optical depth or distance to shadow intensity factors [S1]. The specific layout of these tables is not detailed in the provided sources, but they are accessed via the `vpSetShadowLookupShader` function [S1].

**Steps**
1.  **Initialize VolPack Context:** Create and configure the VolPack context [S1].
2.  **Configure Shadow Lookup Shader:** Call `vpSetShadowLookupShader` to define how shadows are computed and stored in the lookup tables [S1]. The specific parameters for this function are described in the VolPack man page [S1].
3.  **Set Image Buffer:** Declare the array where the result will be stored using `vpSetImage` [S1].
    *   `vpSetImage(vpc, image, width, height, bytes_per_scan, pixel_type)` [S1].
4.  **Render:** Execute the rendering pass. VolPack uses the pre-configured lookup tables to apply shadow effects to the volumetric data in a single pass [S1].

**Formulas**
While the exact lookup table generation formula is not provided, the general transparency model used in VolPack for fog/shadowing is:
$$ T = front\_factor \times e^{-density \times depth} $$
where:
*   $T$ is the transparency [S1].
*   $front\_factor$ is a scaling factor (typically 1.0, but can be larger to brighten foreground) [S1].
*   $density$ controls the fog density or how rapidly objects recede into darkness [S1].
*   $depth$ is the distance from the front plane of the viewing volume (0 at front, 1 at back) [S1].

**Parameters**
*   `front_factor`: Controls foreground brightness [S1].
*   `density`: Controls fog density [S1].
*   `depth`: Normalized distance along the view ray [S1].

**Edge Cases and Pitfalls**
*   **Precision:** The lookup tables must be configured correctly to avoid artifacts. Incorrect configuration may lead to incorrect shadow intensities [S1].
*   **Boundary Conditions:** The `depth` parameter ranges from 0 to 1, corresponding to the front and back planes of the viewing volume [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize VolPack context
vpc = vpCreateContext()

# Configure the shadow lookup shader
# The specific parameters depend on the VolPack man page for vpSetShadowLookupShader
vpSetShadowLookupShader(vpc, shadow_params)

# Set the image buffer for rendering
image_data = allocate_image(width, height, bytes_per_scan, pixel_type)
vpSetImage(vpc, image_data, width, height, bytes_per_scan, pixel_type)

# Render the volume
# This internal step uses the lookup tables to compute shadows
vpRender(vpc)

# Cleanup
vpDestroyContext(vpc)
```

## Subtopics

### Voxel Shadow Raycasting
A shadowing algorithm that casts rays from a light source to each voxel to determine if it is in shadow, darkening the voxel index accordingly [S2, unofficial]. It is chosen for CPU-only voxel engines where pre-computed lookup tables are not feasible or when dynamic voxel modification is required [S2, unofficial]. It differs from Lookup Table Shadow Rendering by computing shadows per-voxel during a pre-processing or update step rather than during the final render pass [S2, unofficial].

**Steps:**
1.  Iterate through all voxels in the grid: `x` from 0 to `xsize-1`, `y` from 0 to `ysize-1`, and `z` from 0 to `zsize-1` [S2, unofficial].
2.  Retrieve the current voxel's palette index: `voxel = get_voxel_pal(x, y, z)` [S2, unofficial].
3.  If the voxel is not empty (`voxel != 0`), cast a ray from the light position `(lx, ly, lz)` to the voxel position `(x, y, z)` using `throw_ray_to_pos` [S2, unofficial]. Let the result be `(hx, hy, hz, i)`, where `i` is the index of the first non-empty voxel hit along the ray [S2, unofficial].
4.  If the ray hit a voxel (`i != 0`), darken the current voxel by subtracting `depth`: `voxel = voxel - depth` [S2, unofficial].
5.  Write the modified voxel index back: `set_voxel_pal(x, y, z, voxel)` [S2, unofficial].

**Parameters:**
*   `lx, ly, lz`: World coordinates of the light source [S2, unofficial].
*   `depth`: Amount to darken the voxel index if in shadow, typically between 1 and 32 [S2, unofficial].
*   `xsize, ysize, zsize`: Dimensions of the current voxel level [S2, unofficial].

**Pitfalls:**
*   The voxel index must not drop below the lower bound of its tone group. Calculate `color_margin = math.floor((voxel - 1) / 32) * 32 + 1`. If `voxel < color_margin`, clamp it to `color_margin` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The algorithm is described as "fast" and "one-pass" [S1].
*   **Memory Complexity:** Depends on the size of the lookup tables [S1]. Not given in the sources.
*   **Strengths:** Fast rendering due to pre-computed shadows [S1].
*   **Limitations:** Requires configuration of lookup shaders [S1]. Less flexible than dynamic shadow ray casting for complex light interactions [S1].
*   **When to use:** When rendering volumetric data with VolPack and performance is critical [S1].
*   **When not to use:** When dynamic, per-frame shadow updates are required without re-configuring lookup tables [S1].

## Background
The method is part of the VolPack volumetric rendering library [S1]. VolPack was developed by Stanford University and is documented in the VolPack User's Guide [S1]. The library supports various volumetric rendering techniques, including this fast one-pass shadow algorithm [S1].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
