# Gradient-Isosurface Texture Packing


## Overview
Gradient-Isosurface Texture Packing is a data layout strategy for 3D volume textures that stores pre-computed gradient vectors in the RGB components and the original scalar isovalue in the Alpha component [S1, unofficial]. This allows a single texture to provide both the data for transfer function indexing and the normals for shading, avoiding the need for a second texture copy or expensive runtime gradient calculation [S1, unofficial]. It differs from [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md) by embedding shading data directly into the volume representation, and from [Röttger et al. Rendering](938-r-ttger-et-al-rendering.md) by relying on hardware interpolation of pre-computed gradients rather than software-based ray casting optimizations [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset of scalar values (density/isovalue).
*   **Output:** A 3D volume texture where each voxel contains a 4-component vector.

**Data Structures:**
The texture stores a `HalfVector4` (or equivalent 16-bit float per channel) per voxel [S1, unofficial].
*   **R, G, B:** The three components of the pre-computed gradient vector (normal) [S1, unofficial][S2].
*   **A:** The original scalar isovalue/density at that voxel position [S1, unofficial][S2].

**Steps:**
1.  **Pre-compute Gradients:** Calculate the gradient vector for each voxel in the source volume data. The sources cite two methods:
    *   **Central Differences:** A simple finite difference scheme [S1, unofficial].
    *   **Sobel Filters:** High-quality Sobel gradients computed serially offline for better accuracy [S2].
2.  **Pack Data:** Transform the calculated gradient vectors and original scalar values into a 4-component array. The X, Y, and Z components of the gradient are assigned to the RGB channels, and the original isovalue is assigned to the Alpha channel [S1, unofficial].
3.  **Upload:** Upload the packed data to the 3D volume texture on the GPU [S1, unofficial][S2].
4.  **Sampling:** During rendering, sample the 3D texture. The hardware interpolates both the density (Alpha) and the gradient (RGB) simultaneously [S2].
5.  **Shading:** Use the interpolated gradient (RGB) to compute lighting (e.g., Phong shading) and the interpolated density (A) to index the transfer function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing phase (CPU)
def pack_volume_data(volume_data: Array3D[float]) -> Array3D[HalfVector4]:
    """
    Pre-computes gradients and packs them into a 3D texture format.
    volume_data: 3D array of scalar density values.
    Returns: 3D array of HalfVector4 (RGB=Gradient, A=Density).
    """
    height, width, depth = volume_data.shape
    packed_data = Array3D[HalfVector4](height, width, depth)
    
    # Compute gradients using Sobel or Central Differences
    # Note: Exact Sobel kernel implementation omitted for brevity; 
    # assume compute_gradient() returns normalized (gx, gy, gz)
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                # Compute gradient at (x, y, z)
                gx, gy, gz = compute_gradient(volume_data, x, y, z)
                
                # Pack into HalfVector4
                # R, G, B store gradient components
                # A stores original density
                packed_data[z, y, x] = HalfVector4(gx, gy, gz, volume_data[z, y, x])
                
    return packed_data

# Rendering phase (GPU Shader)
def sample_volume(pos: Vector3) -> Tuple[Color, float]:
    """
    Samples the packed 3D texture.
    pos: Normalized 3D coordinates [0, 1].
    Returns: (Color, Opacity)
    """
    # Sample the packed texture
    # The hardware interpolates RGB (gradient) and A (density)
    texel = tex3Dlod(VolumeTexture, pos)
    
    gradient = Vector3(texel.r, texel.g, texel.b)
    density = texel.a
    
    # Normalize gradient if necessary (hardware interpolation may not preserve length)
    # [G] Ensure gradient is normalized for correct lighting calculation
    if length(gradient) > 0.0:
        gradient = normalize(gradient)
        
    # Compute lighting using the interpolated gradient
    color = compute_phong_shading(gradient, light_direction, view_direction)
    
    # Use density to index transfer function or determine opacity
    opacity = transfer_function(density)
    
    return color, opacity
```

**Parameters and Edge Cases:**
*   **Gradient Normalization:** The sources note that simple averaging of gradients can result in zero vectors if not handled [S1, unofficial]. The pseudocode includes a normalization step [G].
*   **Precision:** The method uses `HalfVector4` (16-bit floats) to store data, balancing memory usage and precision [S1, unofficial].
*   **Boundary Conditions:** Gradient computation at volume boundaries requires handling out-of-bounds accesses, typically by clamping or mirroring [S1, unofficial].

## Subtopics

### Pre-computed Gradient Resampling
A technique where volume gradients are pre-computed and stored in texture memory alongside density data, allowing the texture mapping hardware to interpolate both density and gradients during resampling [S2]. Unlike Sommer et al.'s method, which computes gradients from a second copy of the volume in main memory (slower for translucent projections), this method uses hardware interpolation for throughput [S2]. It involves computing high-quality Sobel gradients serially offline, loading the four 8-bit values (three gradient components and raw density) into an RGB type texture map, and interpolating them using 3D texture mapping hardware [S2]. This approach produces accurate shading for arbitrary and dynamically changing directional lights, viewing parameters, and transfer functions [S2].

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(N)$ where $N$ is the number of voxels, performed offline [S2]. Runtime complexity is dominated by texture sampling and shading, which is efficient due to hardware interpolation [S2].
*   **Memory Complexity:** Requires storing 4 components per voxel (RGB + A) instead of 1 [S1, unofficial]. This increases memory usage by 4x compared to a single-channel density texture, but avoids the need for a second texture copy [S1, unofficial].
*   **Performance:** On a 200MHz CPU, computing gradients for a $256^3$ volume takes 12 seconds [S2]. The method allows for high-quality Phong shading with hardware acceleration, enabling real-time or near-real-time rendering depending on resolution and view parameters [S2].
*   **Strengths:** Eliminates the need for runtime gradient calculation (6 extra texture fetches) [S1, unofficial]. Supports arbitrary lighting and view changes without reloading textures [S2].
*   **Limitations:** Increases memory bandwidth usage due to 4-component storage [S1, unofficial]. Pre-computation is offline and static; dynamic volume updates require re-computation [S2].

## Background
The method originates from early hardware-accelerated volume rendering research. Cullip and Neumann described a method utilizing the PixelFlow machine that pre-computed gradient components and used texture mapping to interpolate density and gradients [S2]. This was one of the first proposed solutions to implement full Phong lighting functionality in hardware, though it was never fully realized due to hardware constraints (requiring 43 processor cards) [S2]. Sommer et al. later proposed resampling texture volumes in planes parallel to image pixels and mentioned the potential to interpolate gradients with hardware [S2]. The specific "Gradient-Isosurface Texture Packing" terminology and implementation details are found in unofficial tutorials and blog posts [S1, unofficial].

## Sources
[S1] Volume Rendering 102: Transfer Functions, http://graphicsrunner.blogspot.com/2009/01/volume-rendering-102-transfer-functions.html, fetched 2026-10-28 (unofficial)
[S2] High-Quality Volume Rendering Using Texture Mapping ..., https://cfcs.pku.edu.cn/baoquan/docs/20180622110034067331.pdf, fetched 2026-10-27
