# Procedural Volume Rendering with Noise Perturbation

Also called: Texture Coordinate Perturbation, Noise-warped volume rendering, Procedural Volumetric Perturbation, Volume Perturbation via Procedural Noise, Volume Perturbation via Texture Coordinate Displacement

## Overview
Procedural Volume Rendering with Noise Perturbation enhances coarse, low-resolution volumetric data by superimposing high-frequency details using a small, tiled procedural noise volume [S1, unofficial][S2][S3]. It solves the problem of missing microstructure (e.g., wispy clouds, fur, smoke) without increasing the underlying voxel resolution or computational cost of the base volume [S2][S3]. It sets itself apart from sibling methods like [Worley Noise](1726-worley-noise.md) or [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) by applying noise not as a direct density scalar, but as a displacement field or optical modifier, effectively warping the sampling domain or material properties [S2].

## How it works
**Inputs:**
1.  **Macrostructure Volume:** A low-resolution 3D texture (voxel dataset) representing the coarse density or scalar field.
2.  **Perturbation Volume:** A small 3D texture (typically $32 \times 32 \times 32$) initialized with random noise.
3.  **Transfer Function:** Maps scalar density to optical properties (color, opacity).
4.  **Parameters:**
    *   `kNoiseDim`: Integer dimension of the cubic noise volume (typically 32) [S1, unofficial][S2].
    *   `noiseScale`: Scaling factor for the perturbation amplitude [S1, unofficial].
    *   `coordPerturb`: Boolean/weight enabling coordinate perturbation [S1, unofficial].
    *   `propPerturb`: Boolean/weight enabling optical property perturbation [S1, unofficial].
    *   `time`: Animation time offset [S1, unofficial].
    *   `flowSpeed`: Speed of animation [S1, unofficial].

**Outputs:**
*   A rendered image with high-frequency procedural details superimposed on the coarse volume.

**Data Structures:**
*   **Perturbation Volume:**
    *   Format: 3D texture, 8-bit unsigned normalized integer per channel (RGBA8).
    *   Dimensions: $kNoiseDim \times kNoiseDim \times kNoiseDim$ (e.g., $32^3$) [S1, unofficial][S2].
    *   Content: Each texel stores four random 8-bit numbers [S2][S3].
    *   Filtering: Slightly blurred to hide artifacts from trilinear interpolation [S2][S3].
    *   Addressing: Repeat (wrap) mode to allow tiling [S1, unofficial][S2].
*   **Push Constants (Vulkan Example):**
    *   `mvp`: $4 \times 4$ matrix (Model-View-Projection) [S1, unofficial].
    *   `params`: `vec4` containing `[time, noiseScale, coordPerturb, propPerturb]` [S1, unofficial].
    *   `params2`: `vec4` containing `[opacityCorrection, flowSpeed, ...]` [S1, unofficial].

**Steps:**

1.  **Generate Perturbation Volume:**
    *   Create a 3D texture of size $kNoiseDim^3$.
    *   Initialize each voxel with four random 8-bit values (RGBA) [S2][S3].
    *   Apply a slight blur to the volume to mitigate trilinear interpolation artifacts [S2][S3].

2.  **Compute Perturbation Field (Pre-pass or Shader):**
    *   Due to hardware limits on simultaneous textures and dependent texture reads, an additional pass is often required before the main volume shading [S2].
    *   For each pixel/fragment, sample multiple copies of the perturbation volume at different scales [S2][S3].
    *   Let $N_i$ be the noise sample at scale $i$.
    *   Compute the weighted sum of noise components to form a displacement vector $\mathbf{D}$:
        $$ \mathbf{D} = \sum_i w_i \cdot N_i $$
        where $w_i$ are weights for each scale [S2][S3].
    *   **Low Frequency:** Achieved by applying the perturbation texture once with large amplitude [S2].
    *   **High Frequency:** Achieved by applying the perturbation texture again with low amplitude, relying on texture repetition (wrapping) to create fine detail [S2].

3.  **Apply Perturbation (Two Approaches):**

    *   **Approach 1: Coordinate Perturbation (Geometry/Density Warping)**
        1.  Take the original texture coordinates $\mathbf{T}$ for the volume data lookup [S2][S3].
        2.  Use three components of the noise vector $\mathbf{D}$ (e.g., RGB) as a displacement vector [S2][S3].
        3.  Compute perturbed coordinates:
            $$ \mathbf{T}_{perturbed} = \mathbf{T} + \mathbf{D} \cdot \text{scale} $$
        4.  Perform a dependent texture read on the **Macrostructure Volume** using $\mathbf{T}_{perturbed}$ [S2][S3].
        5.  Render the result to a pixel buffer (render-to-texture) to be used in subsequent shading steps [S2].
        6.  *Note:* This avoids recomputing gradients on a perturbed grid, which is required for surface shading but not for volume raycasting/shading [S2].

    *   **Approach 2: Optical Property Perturbation**
        1.  Evaluate the transfer function on the original volume data to get base optical properties (color, opacity) [S2][S3].
        2.  Use all four per-pixel noise components (RGBA) to modify these optical properties [S2][S3].
        3.  The user selects which specific optical properties (e.g., opacity, color channels) are modified [S2].
        4.  This creates inhomogeneities and subtle iridescence effects [S2].

4.  **Animate:**
    *   Add a varying offset vector to the noise texture coordinates in the fragment shader [S3].
    *   Update the offset each frame using `time` and `flowSpeed` [S1, unofficial][S2].
    *   Different offsets can be added to each noise texture copy to enhance motion complexity [S2].

**Pseudocode (Fragment Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# u_macro_volume: 3D texture (low-res macrostructure)
# u_noise_volume: 3D texture (32x32x32, blurred, repeat mode)
# u_transfer_func: 1D texture or function
# u_params: vec4(time, noiseScale, coordPerturb, propPerturb)
# u_params2: vec4(opacityCorrection, flowSpeed, ...)
# v_tex_coord: vec3, original texture coordinate for macro volume

# Step 1: Compute Perturbation Vector
# Sample noise at multiple scales and sum them
vec3 perturbation = vec3(0.0);
float scale = u_params.y; # noiseScale
float time = u_params.x;
float flow = u_params2.y; # flowSpeed

# Example: Two-scale perturbation (Low freq high amp, High freq low amp)
# Scale 1: Large amplitude, low frequency
vec3 coord1 = v_tex_coord * 0.5 + vec3(time * flow, 0.0, 0.0);
vec4 noise1 = texture(u_noise_volume, coord1);
perturbation += noise1.rgb * 2.0; # Large amplitude

# Scale 2: Small amplitude, high frequency (via wrapping)
vec3 coord2 = v_tex_coord * 2.0 + vec3(time * flow * 0.5, 0.0, 0.0);
vec4 noise2 = texture(u_noise_volume, coord2);
perturbation += noise2.rgb * 0.5; # Small amplitude

# Step 2: Apply Perturbation
vec4 color = vec4(0.0);
float opacity = 0.0;

if (u_params.z > 0.5) { # Coordinate Perturbation
    # Warp the lookup coordinate
    vec3 perturbed_coord = v_tex_coord + perturbation * scale;
    # Dependent texture read
    float density = texture(u_macro_volume, perturbed_coord).r;
    # Evaluate transfer function
    vec4 tf_result = texture(u_transfer_func, density);
    color = tf_result;
    opacity = tf_result.a;
} else { # Optical Property Perturbation
    # Standard lookup
    float density = texture(u_macro_volume, v_tex_coord).r;
    vec4 tf_result = texture(u_transfer_func, density);
    
    # Modify optical properties with noise
    # e.g., modulate opacity and color with noise components
    float noise_mod = noise1.r; # Use one component as example
    color.rgb = tf_result.rgb * (1.0 + noise_mod * 0.2);
    color.a = tf_result.a * (1.0 + noise_mod * 0.1);
    opacity = color.a;
}

# Step 3: Composite (Standard Volume Shading)
# ... standard front-to-back or back-to-front compositing ...
```

**Edge Cases and Pitfalls:**
*   **Trilinear Interpolation Artifacts:** The perturbation volume must be blurred slightly to prevent visible artifacts caused by linear interpolation of the random noise values [S2][S3].
*   **Hardware Limits:** Modern GPUs limit the number of simultaneously applied textures and sequential dependent texture reads. This necessitates an additional render-to-texture pass to store the perturbed data before final shading [S2].
*   **Gradient Recomputation:** If using surface shading on perturbed volumes, gradients must be recomputed based on the perturbed grid. This method avoids that cost by perturbing the data access location directly [S2].

## Subtopics

### [Fog Animation](4611-fog-animation.md)
Fog Animation generates the appearance of moving gaseous volumes by perturbing the spatial coordinates of the evaluation point before sampling a static turbulence function, effectively moving the point through a "solid space" texture. Unlike methods that animate the noise function itself, this approach creates motion by shifting the sampling domain, producing upward-swirling gas movement suitable for real-time fog and smoke. It builds on Procedural Volume Rendering with Noise Perturbation [1370-procedural-volume-rendering-with-noise-perturbation.md] by applying coordinate perturbation specifically to transparency-based fog effects.

## Operating characteristics
*   **Time Complexity:** Adds a constant overhead per fragment for sampling the perturbation volume at multiple scales. The additional render-to-texture pass adds one full-screen pass per frame [S2].
*   **Memory Complexity:** Requires storage for the small perturbation volume ($32^3 \times 4$ bytes = 128 KB) and potentially an intermediate render target for the perturbed data [S2].
*   **Performance:** On hardware like NVIDIA GeForce 3 and ATI Radeon 8500, frame rates were only 50-60% slower than volume rendering with no shading, leveraging OpenGL render-to-texture extensions to avoid costly copy operations [S2].
*   **Detail Equivalence:** The visual detail produced is equivalent to an unperturbed voxel dataset of at least 100 times the resolution [S2].
*   **Strengths:** Efficiently adds high-frequency detail (clouds, smoke, fur) to coarse volumes without increasing voxel count or simulation cost [S2][S3].
*   **Limitations:** Requires additional rendering passes due to texture read limits; artifacts can occur if the perturbation volume is not properly filtered [S2].

## Background
The technique was introduced by Kniss et al. in 2002 as part of "Interactive Translucent Volume Rendering and Procedural Modeling" [S2]. It adapts Ebert's lattice-based noise approach for clouds [S2] and Peachey's vector-based noise simulation [S2] to interactive volume rendering. It is detailed in GPU Gems Chapter 39 [S3] and implemented in various open-source projects [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
