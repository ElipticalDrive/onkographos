# Mip-Mapping for Style Functions
Also called: Mip-mapped Style Transfer Function
## Overview
Mip-mapping for style functions is a technique to avoid aliasing in style lookups while preventing mixing between styles at higher mip-map levels [S1, unofficial]. It addresses the visual artifacts caused by abrupt changes in sampling rates or data resolution as the camera or volume moves, ensuring smooth visual transitions between different levels of detail [S1, unofficial]. It distinguishes itself from static mipmapping by focusing on temporal stability rather than just spatial aliasing reduction [S1, unofficial].
## How it works
**Inputs:**
- Style Function Texture (SFT): Can be a 3D texture, a texture array, or a 2D texture.
- Data volume: 3D scalar field.
- Raycasting parameters: Camera position, direction, step size.

**Outputs:**
- Color and opacity for each sample point along the ray.

**Data Structures:**
- **SFT Layout:**
  - *3D Texture:* Not recommended for mipmapping due to style mixing [S1, unofficial].
  - *Texture Array:* Collection of 2D images in layers. Mipmapping performed separately per layer. Requires `EXT_texture_array` (GeForce 8 series) [S1, unofficial].
  - *2D Texture:* Styles arranged in a single 2D texture. Slightly complicates indexing but supported on wider hardware [S1, unofficial].
- **Index Function Texture:** Small, non-filtered texture for style selection [S1, unofficial].

**Steps:**
1. **Raycasting Loop:** For each sample point along the ray:
   a. Compute the projected voxel size at the current sample point.
   b. Determine the base mip-map level based on the projected voxel size relative to the texture resolution.
   c. Calculate curvature and gradient magnitude at the sample point.
   d. Bias the mip-map level using a function of curvature and gradient magnitude to handle high-curvature areas and zero-magnitude gradients [S1, unofficial].
   e. Perform texture lookup on the SFT using the biased mip-map level.
   f. Perform texture lookup on the Index Function Texture.
   g. Combine results to determine color and opacity.

**Formulas:**
- Mip-map level $L$ is determined by the projected voxel size $s$ and texture resolution $R$:
  $$ L = \log_2 \left( \frac{R}{s} \right) $$
  [S1, unofficial]
- Bias function $B$:
  $$ L_{final} = L + B(\kappa, |\nabla f|) $$
  Where $\kappa$ is curvature and $|\nabla f|$ is gradient magnitude. The bias increases $L$ in high-curvature areas to smooth transitions and handles zero-magnitude gradients by preventing unreliable normal predictions [S1, unofficial].

**Parameters:**
- Bias function coefficients: Not given in the sources.
- Texture resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
- **3D Texture Mixing:** If the SFT is a 3D texture, mipmapping causes undesired mixing between styles at higher levels. Solution: Disable mipmapping for 3D SFT [S1, unofficial].
- **Undefined Derivatives:** Standard hardware mip-map level determination using screen-space derivatives is undefined in conditionals or loops (e.g., raycasting) [S1, unofficial].
- **High Curvature:** High curvature areas cause rapid gradient direction variation, leading to artifacts [S1, unofficial].
- **Zero-Magnitude Gradient:** Zero-magnitude gradient vectors are unreliable for normal prediction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_with_mipmapped_style(volume, camera, sft, index_func, sft_type='2d'):
    # Initialize output buffer
    output = empty_buffer()
    
    for each pixel in viewport:
        ray = camera.generate_ray(pixel)
        color = vec3(0.0)
        opacity = 0.0
        
        for sample along ray:
            # Get data value and gradient
            data_val = volume.sample(sample.position)
            gradient = volume.compute_gradient(sample.position)
            grad_mag = length(gradient)
            
            # Compute curvature (simplified)
            # Note: Exact curvature formula not provided in sources [G]
            curvature = compute_curvature(gradient, sample.position)
            
            # Determine base mip-map level from projected voxel size
            # Note: Projected voxel size calculation depends on camera projection matrix [G]
            projected_voxel_size = compute_projected_voxel_size(sample.position, camera)
            base_level = log2(texture_resolution / projected_voxel_size)
            
            # Bias mip-map level based on curvature and gradient magnitude
            # Note: Exact bias function not provided in sources [G]
            bias = compute_bias(curvature, grad_mag)
            mip_level = base_level + bias
            
            # Clamp mip_level to valid range
            mip_level = clamp(mip_level, 0, max_mip_level)
            
            # Lookup style
            if sft_type == '3d':
                # Disable mipmapping for 3D SFT to avoid style mixing
                style = sft.sample(data_val, mip_level=0)
            elif sft_type == 'array':
                style = sft.sample_layer(data_val, mip_level=mip_level)
            elif sft_type == '2d':
                style = sft.sample_2d(data_val, mip_level=mip_level)
            
            # Lookup index
            idx = index_func.sample(data_val)
            
            # Combine style and index
            final_color, final_opacity = combine_style(style, idx)
            
            # Accumulate
            color = blend(color, final_color, opacity, final_opacity)
            opacity = 1.0 - (1.0 - opacity) * (1.0 - final_opacity)
            
            if opacity > threshold:
                break
        
        output[pixel] = color
    return output
```
## Subtopics

### [Mipmapped Smooth Transitions](2973-mipmapped-smooth-transitions.md)
Mipmapped Smooth Transitions is a technique used within Ray-Guided Streaming to provide temporal coherence and anti-aliasing during volume rendering. It addresses the visual artifacts caused by abrupt changes in sampling rates or data resolution as the camera or volume moves, ensuring smooth visual transitions between different levels of detail. It distinguishes itself from static mipmapping by focusing on temporal stability rather than just spatial aliasing reduction.

## Operating characteristics
**Time and Memory Complexity:**
- Time: Additional texture fetches required for mipmapping and bias calculation. Maximum of four additional texture fetches compared to regular 1D transfer function (two for index, two for style) [S1, unofficial].
- Memory: Depends on SFT layout. Texture array requires storing multiple 2D layers [S1, unofficial].

**Measured Performance:**
- Performance comparison on GeForce 8 series hardware:
  - Figure 7(a): Regular TF 11.7 fps, Style TF 11.9 fps [S1, unofficial].
  - Figure 7(b): Regular TF 10.5 fps, Style TF 9.6 fps [S1, unofficial].
  - Figure 7(c): Regular TF 10.1 fps, Style TF 8.1 fps [S1, unofficial].
  - Figure 7(d): Regular TF 12.5 fps, Style TF 12.8 fps [S1, unofficial].
- Data dimensions: $256 \times 256 \times 230$. Viewport size: $512 \times 512$. Object sample distance: 0.5 [S1, unofficial].

**Scaling:**
- Performance benefits from texture caching, especially for the small index function texture [S1, unofficial].
- EXT_texture_buffer_object extension can provide additional performance gains for index function texture on GeForce 8 series [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Avoids aliasing, prevents style mixing (with proper layout), supports temporal stability [S1, unofficial].
- Limitations: Requires manual mip-map level computation, undefined hardware derivatives in loops, potential artifacts in high-curvature areas [S1, unofficial].

**When to Use:**
- Use when aliasing in style lookups is a concern and temporal stability is desired [S1, unofficial].
- Avoid using 3D textures for SFT if mipmapping is enabled [S1, unofficial].

## Background
Origin: Style Transfer Functions for Illustrative Volume Rendering [S1, unofficial].
History: Developed to address aliasing issues in style lookups during raycasting [S1, unofficial].
Usage: Used in illustrative volume rendering to provide smooth visual transitions [S1, unofficial].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
