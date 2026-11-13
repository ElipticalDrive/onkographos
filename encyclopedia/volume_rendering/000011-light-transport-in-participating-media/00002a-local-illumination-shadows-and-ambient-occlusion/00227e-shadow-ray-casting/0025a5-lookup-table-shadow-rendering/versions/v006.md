# Lookup Table Shadow Rendering

Also called: One-pass Shadow with Lookup Tables, Shadow Rendering, VolPack Shadow Algorithm

## Overview
Lookup Table Shadow Rendering is a fast, one-pass shadowing technique implemented within the VolPack volume rendering system [S1]. It solves the problem of computing shadows for participating media efficiently by precomputing shadow contributions into lookup tables, avoiding the per-voxel ray casting overhead of standard shadow ray methods [S1]. Unlike [Shadow Ray Casting](2796-shadow-ray-casting.md), which evaluates visibility per sample or per voxel via explicit integration, this method accelerates the process by mapping shadow conditions to pre-calculated values, trading flexibility for speed [S1].

## How it works
The method relies on the VolPack API to configure and execute the shadow pass [S1].

**Inputs and Outputs:**
- **Input:** A volume dataset (voxels), light source parameters, and rendering context [S1].
- **Output:** A rendered image with shadows applied to the volume [S1].

**Data Structures:**
- **Lookup Table:** The specific layout of the lookup table is not defined in the sources. It is implied to store shadow factors or transmittance values indexed by depth or density [S1].
- **VolPack Context:** A state structure managed by the VolPack library [S1].

**Steps:**
1.  **Initialize VolPack Context:** Create and configure the VolPack rendering context [S1].
2.  **Configure Shadow Lookup Shader:** Call the function `vpSetShadowLookupShader` to define how shadows are computed using lookup tables [S1]. The specific parameters for this function are detailed in the VolPack man page [S1].
3.  **Set Image Buffer:** Declare the array where the result will be stored using `vpSetImage` [S1].
    ```python
    # Pseudocode for setting the image buffer in VolPack
    # Note: This is a C API call, adapted to Python-like pseudocode for clarity
    def set_image_buffer(context, image_array, width, height, bytes_per_scan, pixel_type):
        """
        Declares the array that VolPack should store the image into.
        """
        # vpSetImage(vpc, image, width, height, bytes_per_scan, pixel_type)
        context.image_buffer = image_array
        context.image_width = width
        context.image_height = height
        context.bytes_per_scan = bytes_per_scan
        context.pixel_type = pixel_type
    ```
4.  **Render:** Execute the rendering pass. VolPack internally uses the lookup table to apply shadow factors to the volume data [S1].

**Formulas:**
The transparency of the fog (which is modulated by shadowing) is calculated as [S1]:
$$ T = front\_factor \cdot \exp(-density \cdot depth) $$
Where:
- $T$ is the transparency of the fog.
- $front\_factor$ is a scaling factor (1.0 or larger to brighten the foreground) [S1].
- $density$ controls the "density" of the fog, determining how rapidly objects recede into darkness [S1].
- $depth$ is the distance from the front plane of the viewing volume (0 at the front, 1 at the back) [S1].

Each voxel color component is multiplied by the fog transparency $T$ during rendering [S1].

**Parameters:**
- **Density:** Controls fog density. Typical values are not specified, but it affects the exponential decay of light [S1].
- **Front Factor:** Typically 1.0, but can be larger to brighten the foreground [S1].

**Edge Cases and Pitfalls:**
- The exact implementation details of the lookup table generation and indexing are not provided in the sources [S1].
- Precision issues related to the lookup table resolution are not discussed [S1].

## Subtopics

### Voxel Shadow Raycasting
A shadowing algorithm that casts rays from a light source to each voxel to determine if it is in shadow, darkening the voxel index accordingly [S2, unofficial]. It is distinct from Lookup Table Shadow Rendering as it performs explicit ray casting per voxel rather than using precomputed tables [S2, unofficial]. Choose this method when explicit control over voxel-level shadowing is required and performance constraints are less critical than in large-scale volume rendering [S2, unofficial].

**Steps:**
1.  Iterate $x$ from 0 to $xsize-1$, $y$ from 0 to $ysize-1$, and $z$ from 0 to $zsize-1$ [S2, unofficial].
2.  Retrieve the current voxel's palette index: `voxel = get_voxel_pal(x, y, z)` [S2, unofficial].
3.  If the voxel is not empty (`voxel != 0`), cast a ray from the light position $(lx, ly, lz)$ to the voxel position $(x, y, z)$ using `throw_ray_to_pos` [S2, unofficial]. Let the result be $(hx, hy, hz, i)$, where $i$ is the index of the first non-empty voxel hit along the ray [S2, unofficial].
4.  If the ray hit a voxel ($i \neq 0$), darken the current voxel by subtracting depth: `voxel = voxel - depth` [S2, unofficial].
5.  Write the modified voxel index back: `set_voxel_pal(x, y, z, voxel)` [S2, unofficial].

**Parameters:**
- $lx, ly, lz$: The world coordinates of the light source [S2, unofficial].
- $depth$: The amount to darken the voxel index if in shadow, typically between 1 and 32 [S2, unofficial].
- $xsize, ysize, zsize$: The dimensions of the current level in voxels [S2, unofficial].

**Pitfalls:**
- The voxel index must not drop below the lower bound of its tone group [S2, unofficial]. Calculate `color_margin = math.floor((voxel-1)/32)*32 + 1`. If `voxel < color_margin`, clamp it to `color_margin` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. The lookup table approach is described as "fast" and "one-pass" [S1].
- **Memory Complexity:** Not given in the sources. The lookup table size is not specified [S1].
- **Measured Performance:** Not given in the sources [S1].
- **Strengths:** Fast, one-pass shadowing for volume rendering [S1].
- **Limitations:** Limited flexibility due to the use of lookup tables; specific implementation details are opaque [S1].
- **When to use:** When rendering volumes with VolPack and performance is a priority [S1].
- **When not to use:** When dynamic, per-voxel shadow ray casting is required or when using non-VolPack systems [S1].

## Background
The method was introduced as part of the VolPack volume rendering system [S1]. VolPack is a software library for volume rendering, and this specific shadowing technique was designed to provide efficient shadowing within that framework [S1]. The Voxel Shadow Raycasting variant is part of the "Doommy" voxel engine, an unofficial CPU-based voxel renderer [S2, unofficial].

## Sources
[S1] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S2] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
