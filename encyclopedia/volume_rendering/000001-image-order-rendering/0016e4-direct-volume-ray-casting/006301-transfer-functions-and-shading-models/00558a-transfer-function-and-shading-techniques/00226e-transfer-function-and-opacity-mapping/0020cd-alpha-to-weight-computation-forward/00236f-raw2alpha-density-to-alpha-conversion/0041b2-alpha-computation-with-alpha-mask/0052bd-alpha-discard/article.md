# Alpha Discard

Also called: Alpha-to-Weight Computation with Alpha Mask, Transparency Handling

## Overview
Alpha Discard is a fragment-level optimization that discards fragments with alpha values below a specified threshold to handle transparency [S1, unofficial]. It enables the renderer to skip blending operations for nearly transparent pixels, allowing the rendering engine to pass through to opaque geometry behind them, a technique known as alpha clip [S2]. It is distinct from Alpha Blend, which computes actual transparency but suffers from sorting and performance limitations with multiple surfaces [S2].

## How it works
**Inputs and Outputs:**
- **Input:** Fragment alpha from the diffuse sum, diffuse color alpha, optional alpha map texture, threshold parameter $T$, and flags [S1, unofficial].
- **Output:** Fragment is either discarded (not written to the framebuffer) or passed to the next rendering stage [S1, unofficial].

**Data Structures:**
- **Alpha Map:** A 2D texture (e.g., `m_AlphaMap`) where the red channel (`.r`) stores the alpha value for each texture coordinate [S1, unofficial].
- **Threshold:** A scalar value $m\_AlphaDiscardThreshold$ (float) [S1, unofficial].

**Steps:**
1. **Calculate Base Alpha:** Compute the initial alpha value by multiplying the diffuse sum's alpha by the diffuse color's alpha.
   $$ a = \text{DiffuseSum}.a \times \text{diffuseColor}.a $$
2. **Apply Alpha Map (Optional):** If an alpha map is defined, sample it using the new texture coordinates and multiply the result into the current alpha.
   $$ a = a \times \text{texture2D}(m\_AlphaMap, \text{newTexCoord}).r $$
3. **Discard Check:** If the discard flag is enabled and the computed alpha is strictly less than the threshold, discard the fragment.
   $$ \text{if } \text{DISCARD\_ALPHA} \text{ and } a < m\_AlphaDiscardThreshold \text{ then discard} $$

**Parameters:**
- **$m\_AlphaDiscardThreshold$:** The threshold value for alpha discard. Fragments with alpha below this value are culled [S1, unofficial].
- **DISCARD_ALPHA:** A compile-time or runtime flag that enables the discard logic [S1, unofficial].
- **ALPHAMAP:** A flag indicating the presence of an alpha map texture [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Texture Atlas Setup:** When using a texture atlas, a typical setup requires three distinct materials: opaque, alpha clip (using discard), and transparent (using blend) [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_alpha_discard(diffuse_sum_alpha, diffuse_color_alpha, alpha_map, new_tex_coord, alpha_discard_threshold, discard_alpha_enabled, alpha_map_defined):
    # Step 1: Calculate base alpha
    alpha = diffuse_sum_alpha * diffuse_color_alpha
    
    # Step 2: Apply alpha map if defined
    if alpha_map_defined:
        # Helper to sample texture red channel [G]
        def sample_texture_2d(tex, uv):
            return tex[uv]
        alpha = alpha * sample_texture_2d(alpha_map, new_tex_coord).r
        
    # Step 3: Discard if below threshold
    if discard_alpha_enabled:
        if alpha < alpha_discard_threshold:
            return "DISCARD"
            
    return alpha
```

## Subtopics

### Alpha Clip
A rendering technique that discards transparent pixels to allow rendering through the opaque pass [S2]. It is set apart from Alpha Blend by avoiding depth sorting and blending overhead, but lacks true visual transparency [S2]. Choose it when performance is critical and overlapping transparent surfaces are rare [S2]. [Alpha Clip](alpha-clip.md)

### Alpha Blend
A technique that performs actual transparency with proper blending accumulation [S2]. It is set apart from Alpha Clip by supporting true visual transparency, but requires strict depth sorting and suffers performance penalties with multiple surfaces [S2]. Choose it when accurate visual transparency is required and sorting is manageable [S2]. [Alpha Blend](alpha-blend.md)

### Transparency Index
A property in voxel models (e.g., `VoxelBlockyModel`) that tunes occlusion between faces [S2]. It is set apart from standard culling by allowing faces with different indices to remain visible, enabling visibility through layers like leaves behind glass [S2]. Choose it when fine-grained control over voxel occlusion is needed [S2]. [Transparency Index](transparency-index.md)

### Culls Neighbors
A property in voxel models (default true) that prevents rendering of neighboring voxel sides [S2]. It is set apart from disabled culling by reducing draw calls, but disabling it renders inner voxel sides to make foliage look denser [S2]. Choose it when performance is prioritized over dense foliage appearance [S2]. [Culls Neighbors](culls-neighbors.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Significantly faster than alpha blending for scenes with many nearly transparent surfaces, as it avoids blending operations and depth sorting [S2].
- **Strengths:** Efficient culling of transparent fragments; simple implementation; works well with texture atlases [S2].
- **Limitations:** Does not support true visual transparency; requires careful material setup (opaque, alpha clip, transparent) [S2].
- **When to Use:** When performance is critical and true visual transparency is not required (e.g., foliage, glass with simple occlusion) [S2].
- **When Not to Use:** When accurate visual transparency with multiple overlapping transparent surfaces is needed [S2].

## Background
Not given in the sources.

## Sources
[S1] destroflyer/cubes, https://github.com/destroflyer/cubes, fetched 2026-10-26 (unofficial)
[S2] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
