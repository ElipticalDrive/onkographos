# Orthogonal Opacity Map Creation

Also called: Orthogonal Opacity Map Generation, Projective Texturing Downsampling

## Overview
This method creates a 2D opacity map on a plane orthogonal to a major axis (typically the view direction) by projecting the rendered frame buffer onto that plane and downsampling the result. It solves the problem of efficiently generating a low-resolution occlusion mask for volume rendering culling, minimizing data transfer costs between GPU and CPU while preserving sufficient occlusion information. It differs from hierarchical approaches by using a fixed-resolution, multi-pass averaging strategy rather than a tree structure.

## How it works
**Inputs:**
- Current frame buffer $FB$ containing color/opacity data.
- Desired low-resolution map dimensions $n \times n$ (typical value: $64 \times 64$) [S1].
- Camera projection matrices used for the current frame.

**Outputs:**
- A low-resolution opacity map texture $T_m$ of size $n \times n$ stored in main memory.

**Data Structures:**
- $T_{FB}$: Texture copy of the frame buffer $FB$.
- $PB_{hm}$: High-resolution pixel buffer (optional intermediate, see Optimization).
- $T_{hm}$: Texture bound to $PB_{hm}$, holding the high-resolution opacity map.
- $PB_m$: Low-resolution pixel buffer of size $n \times n$.
- $T_m$: Texture bound to $PB_m$, holding the final low-resolution map.

**Steps:**

1.  **Frame Buffer Copy:** Copy the current frame buffer $FB$ into a texture $T_{FB}$ [S1].
2.  **Projective Texturing (High-Res Generation):**
    - Imagine a rectangle $R$ parallel to the object-aligned orthogonal plane, oriented orthogonal to the camera.
    - Apply projective texturing: treat $T_{FB}$ as a slide projected from the camera position onto $R$ [S1].
    - Orthogonally project the textured rectangle $R$ onto a high-resolution pixel buffer $PB_{hm}$. The resolution of $PB_{hm}$ must be sufficient so that each pixel in $FB$ maps to at least one pixel in $PB_{hm}$ [S1].
    - Bind $PB_{hm}$ to texture $T_{hm}$ [S1].
3.  **Multi-Pass Downsampling:**
    - Initialize $PB_m$ (size $n \times n$) with zeros or clear it.
    - Calculate the number of passes required:
      $$ P = \lceil (N/2) / n^2 \rceil $$
      where $N$ is the side length of the high-resolution map ($N \times N$) and $n$ is the side length of the low-resolution map ($n \times n$) [S1].
    - For pass $k$ from $0$ to $P-1$:
      - Set the blending function to `MIN` [S1].
      - Configure texture coordinates for $T_{hm}$ such that each pixel in $PB_m$ samples the average of 4 neighboring texels in $T_{hm}$. This is achieved by aligning the center of each $PB_m$ pixel with the center of 4 texels in $T_{hm}$ and using linear interpolation [S1].
      - Translate the texture coordinates (or view volume) so that the next set of 4 uncontributed texels in $T_{hm}$ are sampled. The translation ensures that all texels in $T_{hm}$ are eventually covered [S1].
      - Render $T_{hm}$ to $PB_m$ using the configured coordinates and `MIN` blending.
4.  **Readback:** Read the contents of $PB_m$ to main memory [S1].

**Optimization:**
- Combine projection and downsampling into a single step by targeting projective texturing directly to $PB_m$ (the low-resolution pixel buffer) instead of creating $T_{hm}$ explicitly [S1].
- Since hardware auto-generated texture coordinates (TexGen) cannot be easily translated for downsampling in this direct path, apply a translation to the view volume using `glOrtho()` in OpenGL to shift the sampling region for each pass [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_orthogonal_opacity_map(FB, camera_proj, n=64):
    """
    Creates a low-resolution orthogonal opacity map from the frame buffer.
    
    Args:
        FB: Frame buffer texture (width W, height H).
        camera_proj: Projection matrix used for rendering FB.
        n: Desired low-resolution map size (n x n).
        
    Returns:
        map_data: 2D array of size (n, n) containing opacity values.
    """
    # Assume W and H are dimensions of FB
    W, H = FB.width, FB.height
    
    # Determine high-res map size N. 
    # Note: The source implies N is derived from the projection such that 
    # each FB pixel maps to at least one PBhm pixel. 
    # In the optimized path, we target PBm directly.
    # The formula for passes uses N. If N is not explicitly defined as a 
    # separate variable from the projection geometry, we infer it from 
    # the context of "high-resolution map size". 
    # For the optimized path, we treat the projection target as the low-res buffer.
    # However, the formula P = ceil((N/2)/n^2) suggests N is the source resolution 
    # for the downsampling. In the combined step, the "source" is the projected 
    # image. Let's assume N is the resolution of the projected rectangle R.
    # If we project FB onto R, and R is mapped to PBm, the effective N 
    # corresponds to the resolution of the projected image on the plane.
    # For simplicity in implementation without explicit N, we rely on the 
    # pass count logic. If N is not given, we calculate passes based on 
    # the coverage of the projected area.
    
    # Let's assume N is the side length of the high-res map if created explicitly.
    # In the optimized path, we simulate the multi-pass coverage.
    # The source states: "total rendering passes needed is ceil((N/2)/n^2)"
    # We need N. If N is not provided, we cannot compute P exactly.
    # However, the optimization combines steps. 
    # We will implement the generic multi-pass logic assuming N is known 
    # or derived from the projection matrix and view volume.
    
    # Placeholder for N derivation if not provided by caller.
    # In practice, N might be the resolution of the texture TFB projected 
    # onto the plane. If TFB is WxH, and projection scales it, N is the 
    # resulting width/height on the plane.
    # For the purpose of this algorithm, we assume N is passed or calculated.
    # If not, we use a default or error.
    # Let's assume N is the width of the projected rectangle R.
    
    if N is not defined:
        # Estimate N based on FB size and projection? 
        # The source does not specify how N is derived from FB.
        # It says "PBhm has enough resolution so that each FB pixel maps to at least one pixel".
        # This implies N >= max(W, H) or similar depending on aspect.
        # We will assume N is provided or calculated externally.
        raise ValueError("N (high-res map size) must be defined.")

    P = ceil((N / 2) / (n * n))
    
    # Initialize low-res buffer
    PBm = zeros((n, n))
    
    # Setup rendering state
    set_blend_func(MIN)
    
    for k in range(P):
        # Calculate texture coordinate offset for this pass
        # The source says: "translate the texture coordinates so that each pixel 
        # in PBm maps to the center of another 4 texels in Thm that have not contributed"
        # In the optimized path, we translate the view volume (glOrtho).
        
        if use_optimized_path:
            # Apply view volume translation
            translate_view_volume(offset_k)
            # Render FB directly to PBm using projective texturing
            # Texture coordinates are auto-generated by hardware
            render_projective_texture(FB, PBm)
        else:
            # Standard path: Project FB to Thm first (if not already done)
            # Then downsample Thm to PBm
            set_texture_coords(offset_k)
            render_texture(Thm, PBm)
            
    # Read back
    map_data = read_pixel_buffer(PBm)
    return map_data
```

**Edge Cases and Pitfalls:**
- **Precision:** Linear interpolation is used for averaging 4 texels. Ensure floating-point precision is sufficient for opacity values [S1].
- **Boundaries:** Ensure the view volume translation covers the entire high-resolution map without gaps or overlaps that would skew the average [S1].
- **Degenerate Input:** If $N < 2n^2$, the number of passes is 1, but coverage might be incomplete if the translation logic assumes a grid. The formula assumes $N$ is large enough for multiple passes [S1].

## Subtopics

### [Hidden Volume Removal using Hierarchical Occlusion Maps](2624-hidden-volume-removal-using-hierarchical-occlusion-maps.md)
Uses a hierarchical tree structure for occlusion culling, offering better adaptability to varying density but higher complexity. Choose when scene density varies significantly.

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
The parent method that uses this opacity map for culling. Choose this when simple, fast culling along a major axis is sufficient.

## Operating characteristics
- **Time Complexity:** $O(P \cdot n^2)$ where $P = \lceil (N/2)/n^2 \rceil$. Each pass renders $n^2$ pixels.
- **Memory Complexity:** $O(N^2)$ for the high-resolution map if created explicitly, or $O(n^2)$ if optimized. Transfer cost is $O(n^2)$ to main memory [S1].
- **Performance:** Typical low-resolution map size is $64 \times 64$ [S1]. Reduces transfer costs and comparison overhead for occlusion clipping [S1].
- **Strengths:** Simple implementation on GPU; effective for orthogonal views; reduces data transfer [S1].
- **Limitations:** Fixed resolution; may lose fine occlusion details; requires multiple passes for downsampling [S1].

## Background
Originated in the context of texture-based volume rendering, specifically for empty space skipping and occlusion clipping. The method was proposed by Mueller et al. in "Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering" [S1]. It is used in real-time volume rendering applications where culling occluded voxels improves performance.

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
