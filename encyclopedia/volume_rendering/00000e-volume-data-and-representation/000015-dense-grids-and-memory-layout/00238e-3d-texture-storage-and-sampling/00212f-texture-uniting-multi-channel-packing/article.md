# Texture Uniting (Multi-Channel Packing)


## Overview
Texture Uniting is a technique to combine four separate 3D textures (e.g., noise, detail, light, mask) into a single RGBA 3D texture to optimize memory bandwidth and sampling [S1, unofficial]. It sets itself apart from sibling methods like [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md) by focusing on data consolidation for sampling efficiency rather than geometric approximation. Unlike [Volume Texture Creation](1857-volume-texture-creation.md) which may handle single-channel data, this method explicitly manages multi-channel alignment and resolution differences using linear clamping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Four source 3D textures: `noise_texture`, `detail_texture`, `light_texture`, `mask_texture`. Each has an integer dimension ($N_{noise}, N_{detail}, N_{light}, N_{mask}$).
*   **Outputs:** A single unified RGBA 3D texture (`united_texture`) with dimension $N_{united}$ (assumed cubic).

**Data Structures:**
*   `united_texture`: 3D texture of format RGBA. Each texel stores four floating-point values $(x, y, z, w)$.
*   Source textures: 3D textures of format R (single channel).

**Parameters:**
*   $N_{united}$: Integer dimension of the output unified texture.
*   $N_{noise}, N_{detail}, N_{light}, N_{mask}$: Integer dimensions of the source textures.

**Steps:**
1.  Initialize `united_texture` with dimensions $N_{united} \times N_{united} \times N_{united}$.
2.  Iterate through every voxel coordinate $(i, j, k)$ in the unified texture, where $0 \le i, j, k < N_{united}$.
3.  Compute normalized coordinates for sampling:
    $$ u = \frac{i}{N_{united}}, \quad v = \frac{j}{N_{united}}, \quad w = \frac{k}{N_{united}} $$
4.  Sample the noise channel from `noise_texture` using linear clamping sampling at normalized coordinates $(u, v, w)$:
    $$ x = \text{SampleLevel}(\text{sampler\_linear\_clamp}, (u, v, w), 0).r $$
5.  Sample the detail channel from `detail_texture` using linear clamping sampling at normalized coordinates $(u, v, w)$:
    $$ y = \text{SampleLevel}(\text{sampler\_linear\_clamp}, (u, v, w), 0).r $$
6.  Sample the light channel from `light_texture` using linear clamping sampling at normalized coordinates $(u, v, w)$:
    $$ z = \text{SampleLevel}(\text{sampler\_linear\_clamp}, (u, v, w), 0).r $$
7.  Sample the mask channel from `mask_texture` using linear clamping sampling at normalized coordinates $(u, v, w)$ and normalize the result:
    $$ w_{val} = \frac{\text{SampleLevel}(\text{sampler\_linear\_clamp}, (u, v, w), 0).r}{32.0} $$
8.  Write the packed vector to the unified texture:
    $$ \text{united_texture}[i, j, k] = (x, y, z, w_{val}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unite_textures(noise_tex, detail_tex, light_tex, mask_tex, united_size):
    """
    Combines four 3D scalar textures into one RGBA 3D texture.
    
    Args:
        noise_tex: 3D texture object, dimension N_noise
        detail_tex: 3D texture object, dimension N_detail
        light_tex: 3D texture object, dimension N_light
        mask_tex: 3D texture object, dimension N_mask
        united_size: int, dimension of the output texture
        
    Returns:
        united_tex: 3D texture object, dimension united_size x united_size x united_size, format RGBA
    """
    # Initialize output texture
    united_tex = create_3d_texture(width=united_size, height=united_size, depth=united_size, format=RGBA)
    
    # Pre-calculate inverse sizes for normalized coordinate conversion
    inv_united = 1.0 / united_size
    
    for i in range(united_size):
        for j in range(united_size):
            for k in range(united_size):
                # Compute normalized coordinates [0, 1]
                u = i * inv_united
                v = j * inv_united
                w = k * inv_united
                
                # Sample noise channel (R)
                # Uses linear clamping to handle resolution differences between source textures
                x = sample_linear_clamp(noise_tex, u, v, w)
                
                # Sample detail channel (R)
                y = sample_linear_clamp(detail_tex, u, v, w)
                
                # Sample light channel (R)
                z = sample_linear_clamp(light_tex, u, v, w)
                
                # Sample mask channel (R) and normalize
                mask_val = sample_linear_clamp(mask_tex, u, v, w)
                w_val = mask_val / 32.0
                
                # Write to unified texture
                united_tex.set_pixel(i, j, k, float4(x, y, z, w_val))
                
    return united_tex

def sample_linear_clamp(tex, u, v, w):
    """
    Samples a 3D texture at normalized coordinates [0,1] using linear filtering and clamp wrapping.
    This mimics HLSL SampleLevel with sampler_linear_clamp.
    """
    # Convert normalized coordinates to texture space coordinates
    # Assuming tex has dimensions D x D x D
    dim = tex.get_dimension()
    tx = u * dim
    ty = v * dim
    tz = w * dim
    
    # Perform trilinear interpolation
    # Note: In a real implementation, this involves fetching 8 neighboring texels and interpolating.
    # For the purpose of this algorithm description, we assume a standard hardware or library function
    # that performs this operation given normalized coordinates and a linear clamp sampler.
    return tex.sample_linear_clamp(tx, ty, tz)
```

**Edge Cases and Pitfalls:**
*   **Resolution Mismatch:** The method uses normalized coordinates and linear clamping to handle cases where source textures have different resolutions ($N_{noise} \neq N_{detail}$, etc.) [S1, unofficial]. This allows for bilinear filtering across resolution boundaries [S1, unofficial].
*   **Mask Normalization:** The mask channel is divided by 32.0. If the source mask values are not in the expected range, this normalization may produce incorrect weights [S1, unofficial].
*   **Precision:** Arithmetic operations are performed in 32-bit floating point, but results are truncated when written to the destination texture if the texture format uses lower precision (e.g., 16-bit) [S3, unofficial].

## Subtopics

### PBR Material Channel Packing
Packs PBR material properties into standard texture formats to optimize storage and sampling [S2, unofficial]. BaseColor and Alpha are packed into RGBA, while Metallic and Roughness are packed into RGB (R=0, G=Roughness, B=Metallic) [S2, unofficial]. This is chosen when rendering PBR materials with limited texture slots. Pitfalls include ensuring `alphaMode` is 'OPAQUE' and `doubleSided` is set correctly if not remeshing [S2, unofficial].

### Texture Packing Strategy
Avoids packing multiple cell attributes into a single texture if it increases memory bandwidth usage unnecessarily [S3, unofficial]. Frequently accessed attributes (like inside-outside for pressure solve) should be kept in their own scalar textures to minimize bandwidth [S3, unofficial]. A key pitfall is that packing inside-outside and velocity into a single RGBA texture increases memory transfer by 4x, which is inefficient if velocity is unused during pressure solving [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{united}^3)$, where $N_{united}$ is the dimension of the unified texture. Each voxel requires four samples from source textures.
*   **Memory Complexity:** $O(N_{united}^3 \times 4 \times \text{bytes\_per\_channel})$. The unified texture uses 4x the memory of a single-channel texture of the same resolution, but reduces the number of texture objects and potentially improves cache locality during sampling.
*   **Performance:** Optimizes memory bandwidth by reducing the number of texture fetches per pixel during rendering from four separate textures to one [S1, unofficial]. However, if the source textures have vastly different resolutions, the linear clamping sampling may introduce artifacts or require more complex filtering [S1, unofficial].
*   **Strengths:** Reduces texture object overhead, simplifies shader code (single texture lookup), and can improve cache efficiency if the unified texture is accessed frequently [S1, unofficial].
*   **Limitations:** Increases memory usage for the unified texture compared to individual scalar textures if not carefully managed. Not suitable if attributes have vastly different access patterns or if bandwidth is not the primary bottleneck [S3, unofficial].
*   **When to use:** When rendering requires frequent access to multiple scalar fields (e.g., density, temperature, velocity components) and memory bandwidth is a constraint.
*   **When not to use:** When attributes have very different access frequencies or when memory usage is more critical than bandwidth [S3, unofficial].

## Background
Originated in the context of volumetric smoke rendering and fluid simulation, specifically in the implementation of `VolumetricSmoke` [S1, unofficial]. The technique is also discussed in the context of real-time fluid simulation and rendering, where memory bandwidth optimization is critical [S3, unofficial]. It is a practical adaptation of multi-channel packing techniques used in computer graphics to optimize texture sampling performance.

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
[S2] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S3] PDF Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
