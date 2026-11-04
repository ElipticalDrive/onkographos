# Mip-Mapping for Style Functions
Also called: Mip-mapped Style Transfer Function
## Overview
Mip-mapping for style functions is a technique to avoid aliasing in style lookups while preventing mixing between styles at higher mip-map levels [S1, unofficial]. It builds on [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by adding anti-aliasing capabilities to the discrete style selection process [G]. Unlike standard hardware mip-mapping, it manually computes mip-map levels to avoid undefined derivatives in raycasting loops and prevents style boundary corruption [S1, unofficial].
## How it works
### Inputs and outputs
- **Input**: Raycasting sample points, data values, surface normals, style transfer function (SFT) texture, index function texture [S1, unofficial].
- **Output**: Mip-map level index for the SFT lookup [S1, unofficial].

### Data structures
- **Style Transfer Function (SFT)**: Can be stored as a 3D texture, a texture array, or a 2D texture [S1, unofficial].
    - **3D Texture**: Styles stored as slices [S1, unofficial].
    - **Texture Array**: Collection of 2D images in layers (e.g., `EXT_texture_array`) [S1, unofficial].
    - **2D Texture**: Styles arranged in a single 2D texture [S1, unofficial].

### Steps
1. **Determine Base Mip-map Level**:
   - Manually compute the projected voxel size at each sample point [S1, unofficial].
   - Calculate the screen-space derivative of the texture coordinates. Since raycasting involves loops/conditionals, hardware derivatives are undefined [S1, unofficial].
   - Compute the size of the projected voxel in texture space [S1, unofficial].
   - Determine the base mip-map level $L$ based on the ratio of the projected voxel size to the texture resolution [S1, unofficial].

2. **Bias Mip-map Level**:
   - Calculate curvature and gradient magnitude from the data and normals [S1, unofficial].
   - Apply a bias function $B$ to the base level $L$:
     $$ L_{final} = L + B(\text{curvature}, \|\nabla f\|) $$
   - The bias accounts for areas of high curvature where gradient direction varies quickly, which can lead to artifacts [S1, unofficial].
   - Handle zero-magnitude gradient vectors, which are unreliable for normal prediction [S1, unofficial].

3. **Lookup**:
   - Use $L_{final}$ to select the appropriate mip-map level for the SFT lookup [S1, unofficial].
   - Perform the texture fetch using the biased level [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mip_level(data_value, normal, voxel_size, texture_resolution):
    # 1. Manual projection size calculation
    projected_size = voxel_size * texture_resolution
    
    # 2. Base level determination
    max_level = int(log2(texture_resolution))
    base_level = clamp(log2(projected_size / texture_resolution), 0, max_level)
    
    # 3. Bias calculation
    # Compute gradient magnitude and curvature from normals/data [G]
    gradient_magnitude = length(normal)
    curvature = divergence(normal)
    
    # Sources do not specify the exact bias formula [G]
    safe_gradient = max(gradient_magnitude, 1e-6)
    bias = curvature / safe_gradient
    
    final_level = clamp(base_level + bias, 0, max_level)
    return final_level
```

### Edge cases and pitfalls
- **3D Texture Mixing**: If the SFT is a 3D texture, standard mipmapping causes undesired mixing between styles at higher levels. Solution: Disable mipmapping for 3D SFT or use texture arrays/2D textures [S1, unofficial].
- **Undefined Derivatives**: Standard hardware mip-map level determination using screen-space derivatives is undefined in conditionals or loops (e.g., raycasting) [S1, unofficial].
- **High Curvature**: High curvature areas cause rapid gradient direction variation, leading to artifacts [S1, unofficial].
- **Zero Gradient**: Zero-magnitude gradient vectors are unreliable for normal prediction [S1, unofficial].

## Subtopics

### [Mipmapped Smooth Transitions](2973-mipmapped-smooth-transitions.md)
Mipmapped Smooth Transitions is a technique that applies mipmapping to style transfer functions to ensure temporal coherence and anti-aliasing during interactive volume rendering. It distinguishes itself from standard mipmapping by specifically targeting the visual artifacts caused by rapid changes in style parameters, allowing for smooth interpolation between different visual representations (such as DVR and MIP) without the mixing of distinct styles at higher mip levels[782-mip-mapping-for-style-functions.md].

## Operating characteristics
- **Performance**: Requires a maximum of four additional texture fetches (two for index function, two for style function) compared to a regular 1D transfer function [S1, unofficial].
- **Caching**: The index function texture is small and does not require filtering, benefiting heavily from texture caching [S1, unofficial].
- **Hardware**: On GeForce 8 series, the index function can be implemented as a buffer texture using `EXT_texture_buffer_object` for additional performance gains [S1, unofficial].
- **Measured Performance**: In tests with a $256 \times 256 \times 230$ volume and $512 \times 512$ viewport, style transfer functions achieved comparable or slightly lower frame rates than regular transfer functions (e.g., 11.7 fps vs 11.9 fps for Figure 7(a)) [S1, unofficial].

## Background
Originated in the context of illustrative volume rendering to handle discrete style mappings efficiently [S1, unofficial]. The technique was developed to address the limitations of standard hardware mip-mapping when applied to multi-style textures, ensuring that style boundaries remain sharp while providing anti-aliasing benefits [S1, unofficial]. It is used in real-time volume rendering applications, particularly for medical imaging and scientific visualization [G].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://www.researchgate.net/publication/227615609_Style_Transfer_Functions_for_Illustrative_Volume_Rendering, fetched 2026-10-27 (unofficial)
