# Texture Uniting (Multi-Channel Packing)

Also called: Multi-Channel Packing, Texture Packing/Unpacking Scheme

## Overview
Texture Uniting is a technique to combine four separate 3D textures (noise, detail, light, mask) into a single RGBA 3D texture to optimize memory bandwidth and sampling [S1, unofficial]. It uses linear clamping sampling with normalized coordinates to handle resolution differences between source textures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Four source 3D textures (`noise_texture`, `detail_texture`, `light_texture`, `mask_texture`) with integer dimensions (`noise_texture_size`, `detail_texture_size`, `light_texture_size`, `mask_texture_size`) [S1, unofficial].
*   **Output:** A single unified 3D texture (`united_texture`) with integer dimension `united_texture_size` (assumed cubic) [S1, unofficial].

**Data Structures:**
*   `united_texture`: A 3D texture resource with RGBA format (e.g., `RGBA32`, 8 bits per channel) [S2].
*   Source textures: 3D textures with single-channel (R) or RGBA formats, typically normalized to [0, 1] [S1, unofficial].

**Steps:**
1.  Initialize the `united_texture` with dimensions `united_texture_size` [S1, unofficial].
2.  For each voxel coordinate `id.xyz` in the output volume:
    a.  Compute normalized coordinates for sampling: `uv = id.xyz / united_texture_size` [S1, unofficial].
    b.  Sample the noise channel: `x = noise_texture.SampleLevel(sampler_linear_clamp, uv, 0).r` [S1, unofficial].
    c.  Sample the detail channel: `y = detail_texture.SampleLevel(sampler_linear_clamp, uv, 0).r` [S1, unofficial].
    d.  Sample the light channel: `z = light_texture.SampleLevel(sampler_linear_clamp, uv, 0).r` [S1, unofficial].
    e.  Sample the mask channel and normalize: `w = mask_texture.SampleLevel(sampler_linear_clamp, uv, 0).r / 32.0` [S1, unofficial].
    f.  Write the packed value: `united_texture[id.xyz] = (x, y, z, w)` [S1, unofficial].

**Formulas:**
*   Normalized coordinate: $uv = \frac{id}{S_{united}}$ [S1, unofficial].
*   Noise value: $x = T_{noise}(uv)$ [S1, unofficial].
*   Detail value: $y = T_{detail}(uv)$ [S1, unofficial].
*   Light value: $z = T_{light}(uv)$ [S1, unofficial].
*   Mask value: $w = \frac{T_{mask}(uv)}{32.0}$ [S1, unofficial].

Where $S_{united}$ is `united_texture_size`, and $T_{channel}$ denotes the sampling function for the respective source texture [S1, unofficial].

**Parameters:**
*   `united_texture_size`: int dimension of the output unified texture (assumed cubic) [S1, unofficial].
*   `noise_texture_size`, `detail_texture_size`, `light_texture_size`, `mask_texture_size`: int dimensions of the source textures [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Resolution Mismatch:** Source textures may have different resolutions [S1, unofficial]. The implementation uses `SampleLevel` with normalized coordinates rather than integer indexing, allowing bilinear filtering across resolution boundaries between source textures [S1, unofficial].
*   **Mask Normalization:** The mask value is divided by 32.0 [S1, unofficial].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unite_textures(noise_tex, detail_tex, light_tex, mask_tex, united_size):
    """
    Combines four 3D textures into a single RGBA 3D texture.
    
    Args:
        noise_tex: 3D texture object for noise channel
        detail_tex: 3D texture object for detail channel
        light_tex: 3D texture object for light channel
        mask_tex: 3D texture object for mask channel
        united_size: int, dimension of the output cubic texture
        
    Returns:
        united_tex: 3D list of RGBA tuples
    """
    # Initialize output texture as a 3D list of RGBA tuples
    united_tex = [[[None for _ in range(united_size)] for _ in range(united_size)] for _ in range(united_size)]
    inv_size = 1.0 / united_size
    for z in range(united_size):
        for y in range(united_size):
            for x in range(united_size):
                # Compute normalized coordinates
                uv = (x * inv_size, y * inv_size, z * inv_size)
                # Sample each channel (assuming tex.sample(uv, mip) returns a tuple (r, g, b, a))
                x_val = noise_tex.sample(uv, 0)[0]
                y_val = detail_tex.sample(uv, 0)[0]
                z_val = light_tex.sample(uv, 0)[0]
                w_val = mask_tex.sample(uv, 0)[0] / 32.0
                united_tex[z][y][x] = (x_val, y_val, z_val, w_val)
    return united_tex
```

## Subtopics

### [Texture Packing Strategy](3196-texture-packing-strategy.md)
Texture Packing Strategy maps a single underlying volume dataset to multiple distinct GPU textures, each scaled by a specific scalar multiplier to reduce GPU memory consumption. It solves excessive memory usage when rendering high-resolution volumes with multiple attribute channels by avoiding raw voxel data duplication. Unlike [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which combines attributes into a single RGBA texture for simultaneous access, this strategy separates attributes into individual scalar textures to minimize memory bandwidth when attributes are accessed independently.

### PBR Material Channel Packing
Packs PBR material properties into standard texture formats: BaseColor and Alpha into RGBA, Metallic and Roughness into RGB (R=0, G=Roughness, B=Metallic) [S3, unofficial]. Steps: Pack BaseColor/Alpha by concatenating `[base_color, alpha]` along the last axis; Pack Metallic/Roughness by concatenating `[zeros_like(metallic), roughness, metallic]` along the last axis [S3, unofficial]. Parameters: `alphaMode` is 'OPAQUE', `doubleSided` is True if not remesh else False [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Reduces texture binding overhead, enables simultaneous access to multiple attributes, handles resolution mismatches via filtering [S1, unofficial].
*   **Limitations:** Requires all attributes to be sampled simultaneously; if attributes are accessed independently, separate textures may be more efficient [S1, unofficial]. The mask normalization factor (32.0) is specific to the source implementation [S1, unofficial].
*   **When to use:** When rendering volumes where multiple attributes (noise, detail, light, mask) are needed per voxel and are accessed together [S1, unofficial].
*   **When not to use:** When attributes are accessed independently or have vastly different access patterns [S1, unofficial].

## Background
Origin: This technique is described in the context of volumetric smoke rendering, specifically in the `VolumetricSmoke` project by furkanunsal42 [S1, unofficial]. It is used to optimize the rendering pipeline by combining multiple texture lookups into a single unified texture, reducing the overhead associated with multiple texture bindings and sampling operations [S1, unofficial].

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
[S2] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
[S3] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
