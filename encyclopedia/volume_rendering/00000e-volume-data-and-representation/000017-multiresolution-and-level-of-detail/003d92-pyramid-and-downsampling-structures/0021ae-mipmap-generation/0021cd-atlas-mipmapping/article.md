# Atlas Mipmapping

Also called: Mipmapping the Atlas

## Overview
Atlas Mipmapping addresses the undersampling artifacts that occur when a distant surfel covers few pixels but retains its full-resolution texture tile, causing a single pixel to span multiple texels during a bilinear fetch [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A level-0 texture atlas containing surfel tiles, where each tile is compressed (e.g., BC7) and optionally dictionary-encoded [S1, unofficial].
*   **Output:** A mip chain attached to the atlas, where each subsequent level is a box-filtered version of level 0, computed in the quantized domain and compressed like level 0 [S1, unofficial].

**Data Structures:**
The atlas is a 2D texture where each surfel occupies a rectangular tile [G]. The mip chain consists of $L$ levels, where each level $l$ is a downscaled version of the previous level [G].

**Steps:**
1.  **Level Selection:** The mip level is chosen once per surfel from the texel footprint of a one-pixel step, which the closed-form intersection of Sec. 3.3 already provides [S1, unofficial].
2.  **Mip Chain Generation:** Each level is a box filter of level 0, computed in the quantized domain and compressed like level 0 [S1, unofficial].
3.  **Compression:** Each level is compressed using the same scheme as level 0 (e.g., BC7 block compression) [S1, unofficial]. If block dictionary compression is used, the atlas is cut into 4 × 4 texel blocks, K-means clustering builds a codebook of $K=65,536$ centroids, and every block stores a 16-bit codebook index [S1, unofficial].

**Parameters:**
*   **Extra Levels:** Four extra levels add 33% to the atlas bytes [S1, unofficial].
*   **Default State:** Mipmapping is off by default, and every number in this paper is rendered at level 0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_atlas_mip_chain(atlas_level_0, num_extra_levels):
    levels = [atlas_level_0]
    for l in range(1, num_extra_levels + 1):
        # Each level is a box filter of level 0, computed in the quantized domain
        filtered = box_filter_downsample(levels[l-1], factor=2)
        # Compressed like level 0 (e.g., BC7 block compression)
        compressed = apply_block_compression(filtered)
        levels.append(compressed)
    return levels

def box_filter_downsample(data, factor):
    h, w = data.shape[:2]
    out = np.zeros((h // factor, w // factor, data.shape[2]), dtype=data.dtype)
    for y in range(0, h - factor + 1, factor):
        for x in range(0, w - factor + 1, factor):
            out[y//factor, x//factor] = data[y:y+factor, x:x+factor].mean(axis=(0, 1))
    return out

def apply_block_compression(data):
    # Standard block compression (e.g., BC7) applied per 4x4 tile
    # [G] Implementation uses hardware-accelerated codec; here represented as a block operation
    return data
```

**Edge Cases and Pitfalls:**
*   **Precision:** Filtering in the quantized domain may introduce artifacts if the compression scheme is not linear [S1, unofficial].
*   **Degenerate Input:** If a surfel is extremely small, it may map to a high mip level where it becomes a single texel or less [G].

## Subtopics

### [Anisotropic Texture Atlas Packing](2670-anisotropic-texture-atlas-packing.md)
Anisotropic Texture Atlas Packing is a method to pack surfel textures into a fixed-width atlas using anisotropic rectangular tiles sized according to the Nyquist rate of the hash grid, avoiding square padding on thin surfels. It addresses the inefficiency of square padding for non-uniform surfels by aligning tile dimensions with the surfel's tangent axis scales.

### [Texel Allocation via Rate-Distortion](2672-texel-allocation-via-rate-distortion.md)
This method optimizes the spatial footprint and texel count of surfels in a texture atlas by excluding zero-support regions and allocating texels based on visual importance rather than strict Nyquist limits. It addresses the inefficiency of uniform atlas sizing where distant surfels consume excessive memory for negligible visual contribution. It differs from its parent [Atlas Mipmapping](2639-atlas-mipmapping.md) by actively reducing the atlas size through rate-distortion optimization, and from its sibling [Anisotropic Texture Atlas Packing](2670-anisotropic-texture-atlas-packing.md) by focusing on per-surfel size allocation via error probing rather than geometric packing constraints.

## Operating characteristics
*   **Time Complexity:** Mip chain generation is $O(N)$ where $N$ is the number of texels, as each level is a constant-factor downsampling of the previous one [G]. Level selection is $O(1)$ per surfel after precomputation [G].
*   **Memory Complexity:** Adds 33% to the atlas size for 4 extra levels [S1, unofficial].
*   **Measured Performance:** On the "garden" scene, mipmapping adds 0.32 dB PSNR degradation and 0.045 LPIPS increase compared to the sharper level (27.20 PSNR / 0.8417 SSIM / 0.1270 LPIPS) versus no mips (27.06 PSNR / 0.8405 SSIM / 0.1247 LPIPS) [S1, unofficial]. Frame rate is unchanged on an RTX 5090 [S1, unofficial].
*   **Strengths:** Reduces undersampling artifacts for distant surfels without changing the rendering pipeline [G].
*   **Limitations:** Increases memory usage [G]. Filtering in the quantized domain may not be optimal for all compression schemes [G].
*   **When to Use:** When distant surfels cause aliasing or blurring due to undersampling [G].
*   **When Not to Use:** When memory is constrained and distant surfels are not a significant issue, or when the compression scheme does not support efficient quantized-domain filtering [G].

## Background
Originated in "Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting" (Johnson et al., 2026) [S1, unofficial]. It builds on rasterization optimizations from Speedy-Splat and FastGS [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
