# Per-Field Max Pyramid in Deviation Space

Also called: deviation-space max pyramid, maxPyramidTexture

## Overview
A max-value pyramid stored in deviation space (normalized distance from contrast center) used for transfer-function-adaptive skipping during volume ray marching [S1, unofficial]. It is built from the raw volume cube, not the display mip chain, to avoid hiding thin bright filaments that might be smoothed out by box filtering [S1, unofficial]. Unlike standard mipmaps which store averaged or filtered values for display, this structure stores the maximum possible signal intensity relative to a threshold, enabling the renderer to skip empty or low-intensity regions dynamically based on live contrast settings [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw volume data cube (dimensions $D \times D \times D$), `contrastCenter` (scalar), `halfRange` (scalar).
*   **Output:** A 3D texture mip chain (`maxPyramidTexture`) where each level $k$ stores the maximum deviation value for its corresponding spatial region.

**Data Structures:**
*   **FieldEntry:** Contains a new field `maxPyramidTexture` [S1, unofficial].
*   **Bind Group:** Binding 5 is added to the shared `@group(0)` bind-group layout [S1, unofficial].
*   **Texture Properties:** The texture is `textureLoad`-only, with no sampler [S1, unofficial].

**Steps:**
1.  **Initial Reduction (Deviation Conversion):**
    The raw volume cube is reduced through three chained $2\times$ max-reductions. This process uses two upload-time scratch textures as intermediaries [S1, unofficial].
    *   **First Reduction:** Converts raw values to normalized deviation. For each voxel value $v$, the deviation $d$ is calculated as:
        $$ d = \frac{|v - \text{contrastCenter}|}{\text{halfRange}} $$
        This deviation is non-negative by construction [S1, unofficial].
    *   **Subsequent Reductions:** The remaining two reductions compose via plain `max` operations on the deviation values. Since deviation is non-negative, standard max-composition correctly propagates the maximum intensity information up the hierarchy without further normalization [S1, unofficial].

2.  **Mip Chain Generation:**
    After the initial reductions, the pyramid's own chain is filled via the `generateMipChain3d` primitive [S1, unofficial]. This primitive fills 3D-texture mip chains via depthSlice render passes [S1, unofficial].

**Parameters:**
*   `contrastCenter`: The center value for deviation calculation [S1, unofficial].
*   `halfRange`: The half-range used for normalization [S1, unofficial].
*   Typical values are not explicitly given in the sources, but the method supports live slider changes, implying these are dynamic uniforms [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Thin Filaments:** Building from the display mips (box-filtered) can hide thin bright filaments. The method explicitly avoids this by building from the raw cube [S1, unofficial].
*   **Precision:** The deviation formula involves division by `halfRange`. If `halfRange` is zero, the operation is undefined. The sources do not specify a guard for this case [G: Implementers should ensure `halfRange > 0` or handle division by zero explicitly].
*   **Shader Access:** No shader declares binding 5 yet in the source context, but the layout allows unused entries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_per_field_max_pyramid(raw_volume: Texture3D, contrast_center: float, half_range: float) -> Texture3D:
    """
    Builds the per-field max pyramid in deviation space.
    raw_volume: The input raw volume cube (D x D x D).
    contrast_center: The center value for deviation calculation.
    half_range: The half-range used for normalization.
    Returns: The maxPyramidTexture mip chain.
    """
    # Step 1: Initial Reduction through three chained 2x max-reductions
    # Using two upload-time scratch textures as intermediaries
    
    # Level 0: Raw volume
    current_level = raw_volume
    
    # First reduction: Convert to deviation space
    # Only the first reduction converts raw values to normalized deviation
    next_level = create_scratch_texture(current_level.width // 2, current_level.height // 2, current_level.depth // 2)
    
    for z in range(0, current_level.depth, 2):
        for y in range(0, current_level.height, 2):
            for x in range(0, current_level.width, 2):
                # Gather 8 voxels from the 2x2x2 block
                v000 = current_level.load(x, y, z)
                v001 = current_level.load(x, y, z + 1)
                v010 = current_level.load(x, y + 1, z)
                v011 = current_level.load(x, y + 1, z + 1)
                v100 = current_level.load(x + 1, y, z)
                v101 = current_level.load(x + 1, y, z + 1)
                v110 = current_level.load(x + 1, y + 1, z)
                v111 = current_level.load(x + 1, y + 1, z + 1)
                
                # Calculate deviation for each voxel
                d000 = abs(v000 - contrast_center) / half_range
                d001 = abs(v001 - contrast_center) / half_range
                d010 = abs(v010 - contrast_center) / half_range
                d011 = abs(v011 - contrast_center) / half_range
                d100 = abs(v100 - contrast_center) / half_range
                d101 = abs(v101 - contrast_center) / half_range
                d110 = abs(v110 - contrast_center) / half_range
                d111 = abs(v111 - contrast_center) / half_range
                
                # Max of deviations
                max_dev = max(d000, d001, d010, d011, d100, d101, d110, d111)
                
                next_level.store(x // 2, y // 2, z // 2, max_dev)
    
    # Second reduction: Plain max (deviation is non-negative)
    current_level = next_level
    next_level = create_scratch_texture(current_level.width // 2, current_level.height // 2, current_level.depth // 2)
    
    for z in range(0, current_level.depth, 2):
        for y in range(0, current_level.height, 2):
            for x in range(0, current_level.width, 2):
                v000 = current_level.load(x, y, z)
                v001 = current_level.load(x, y, z + 1)
                v010 = current_level.load(x, y + 1, z)
                v011 = current_level.load(x, y + 1, z + 1)
                v100 = current_level.load(x + 1, y, z)
                v101 = current_level.load(x + 1, y, z + 1)
                v110 = current_level.load(x + 1, y + 1, z)
                v111 = current_level.load(x + 1, y + 1, z + 1)
                
                max_val = max(v000, v001, v010, v011, v100, v101, v110, v111)
                next_level.store(x // 2, y // 2, z // 2, max_val)
    
    # Third reduction: Plain max
    current_level = next_level
    next_level = create_scratch_texture(current_level.width // 2, current_level.height // 2, current_level.depth // 2)
    
    for z in range(0, current_level.depth, 2):
        for y in range(0, current_level.height, 2):
            for x in range(0, current_level.width, 2):
                v000 = current_level.load(x, y, z)
                v001 = current_level.load(x, y, z + 1)
                v010 = current_level.load(x, y + 1, z)
                v011 = current_level.load(x, y + 1, z + 1)
                v100 = current_level.load(x + 1, y, z)
                v101 = current_level.load(x + 1, y, z + 1)
                v110 = current_level.load(x + 1, y + 1, z)
                v111 = current_level.load(x + 1, y + 1, z + 1)
                
                max_val = max(v000, v001, v010, v011, v100, v101, v110, v111)
                next_level.store(x // 2, y // 2, z // 2, max_val)
    
    # Step 2: Fill the pyramid's own chain via generateMipChain3d
    # The final reduced level is the base of the pyramid.
    # generateMipChain3d will handle the rest of the mip chain generation.
    pyramid_texture = generate_mip_chain_3d(next_level)
    
    return pyramid_texture
```

## Subtopics

### Max-Pyramid Level of Detail (LOD)
A pre-computed hierarchy of maps for efficient calculation of maximum function values on intervals, used to estimate maximums during rendering [S2]. It starts with initial data at level $n$ (forming a $2^n \times 2^n$ array). To generate level $n-1$, the maximum is chosen from every four adjacent values of level $n$, and the other three values are discarded [S2]. This repeats until level 0, which consists of a single value (the global maximum) [S2]. Level $k$ is a $2^k \times 2^k$ array [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method runs at 1/3 resolution with a hard 128-step cap [S1, unofficial].
*   **Scaling:** The skip cutoff is derived in-shader from live contrast/trim uniforms, so slider changes retune skipping with nothing to rebuild [S1, unofficial]. It is correct for sequential (MCPM) and divergent (CF-4) palettes [S1, unofficial].
*   **Strengths:** Avoids hiding thin bright filaments by building from the raw cube [S1, unofficial]. Enables TF-adaptive skipping [S1, unofficial].
*   **Limitations:** Requires additional upload-time scratch textures [S1, unofficial].
*   **When to use:** When dynamic, transfer-function-adaptive skipping is needed for volume ray marching [S1, unofficial].
*   **When not to use:** When thin bright filaments are not a concern and display mips are sufficient [S1, unofficial].

## Background
The concept of using max-pyramids for level-of-detail in volume rendering has roots in earlier work on GPU-based volume-oriented rendering of functionally defined objects [S2]. The specific implementation as a "Per-Field Max Pyramid in Deviation Space" is part of the volume-raymarch-acceleration plan, specifically Task 3 [S1, unofficial]. It was introduced in the `skymap` project by rulkens [S1, unofficial].

## Sources
[S1] feat(volumes): raymarch acceleration — GPU pyramids, TF-adaptive skipping, cone LOD, https://github.com/rulkens/skymap/pull/556, fetched 2026-10-28 (unofficial)
[S2] GPU-based volume-oriented rendering of functionally defined objects, https://doi.org/10.21279/1454-864x-18-i2-005, fetched 2026-10-26
