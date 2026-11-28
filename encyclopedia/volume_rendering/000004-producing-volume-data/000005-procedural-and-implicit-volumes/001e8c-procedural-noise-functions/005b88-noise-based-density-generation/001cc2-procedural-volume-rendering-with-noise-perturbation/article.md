# Procedural Volume Rendering with Noise Perturbation

Also called: Texture Coordinate Perturbation, Noise-warped volume rendering, Procedural Volumetric Perturbation, Volume Perturbation via Procedural Noise, Volume Perturbation via Texture Coordinate Displacement

## Overview
Procedural Volume Rendering with Noise Perturbation enhances coarse, low-resolution volumetric data by superimposing high-frequency details using a small, tiled procedural noise volume [S1, unofficial][S2][S3]. It solves the problem of missing microstructure (e.g., wispy clouds, fur, smoke) without increasing the underlying voxel resolution or computational cost of the base volume [S2][S3]. It sets itself apart from sibling methods like [Worley Noise](1726-worley-noise.md) or [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) by applying noise not as a direct density scalar, but as a displacement field or optical modifier, effectively warping the sampling domain or material properties [S2].

## How it works
**Inputs:**
1. **Macrostructure Volume:** A low-resolution 3D texture (voxel dataset) representing the coarse density or scalar field [S2][S3].
2. **Perturbation Volume:** A small 3D texture (typically $32 \times 32 \times 32$) initialized with random noise [S2].
3. **Transfer Function:** Maps scalar density to optical properties (color, opacity) [S2][S3].
4. **Parameters:**
   * `kNoiseDim`: Integer dimension of the cubic noise volume (typically 32) [S1, unofficial][S2].
   * `noiseScale`: Scaling factor for the perturbation amplitude [S1, unofficial].
   * `coordPerturb`: Boolean/weight enabling coordinate perturbation [S1, unofficial].
   * `propPerturb`: Boolean/weight enabling optical property perturbation [S1, unofficial].
   * `time`: Animation time offset [S1, unofficial].
   * `flowSpeed`: Speed of animation [S1, unofficial].

**Outputs:**
* A rendered image with high-frequency procedural details superimposed on the coarse volume [S3].

**Data Structures:**
* **Perturbation Volume:**
  * Format: 3D texture, 8-bit unsigned normalized integer per channel (RGBA8).
  * Dimensions: $kNoiseDim \times kNoiseDim \times kNoiseDim$ (e.g., $32^3$) [S1, unofficial][S2].
  * Content: Each texel stores four random 8-bit numbers [S2][S3].
  * Filtering: Slightly blurred to hide artifacts from trilinear interpolation [S2][S3].
  * Addressing: Repeat (wrap) mode to allow tiling [S1, unofficial][S2].
* **Push Constants (Vulkan Example):**
  * `mvp`: $4 \times 4$ matrix (Model-View-Projection) [S1, unofficial].
  * `params`: `vec4` containing `[time, noiseScale, coordPerturb, propPerturb]` [S1, unofficial].
  * `params2`: `vec4` containing `[opacityCorrection, flowSpeed, ...]` [S1, unofficial].

**Steps:**

1. **Generate Perturbation Volume:**
   * Create a 3D texture of size $kNoiseDim^3$ [S1, unofficial].
   * Initialize each voxel with four random 8-bit values (RGBA) [S2][S3].
   * Apply a slight blur to the volume to mitigate trilinear interpolation artifacts [S2][S3].

2. **Compute Perturbation Field (Pre-pass or Shader):**
   * Due to hardware limits on simultaneous textures and dependent texture reads, an additional pass is often required before the main volume shading [S2].
   * For each pixel/fragment, sample multiple copies of the perturbation volume at different scales [S2][S3].
   * Let $N_i$ be the noise sample at scale $i$ [S2][S3].
   * Compute the weighted sum of noise components to form a displacement vector $\mathbf{D}$:
     $$ \mathbf{D} = \sum_i w_i \cdot N_i $$
     where $w_i$ are weights for each scale [S2][S3].
   * **Low Frequency:** Achieved by applying the perturbation texture once with large amplitude [S2].
   * **High Frequency:** Achieved by applying the perturbation texture again with low amplitude, relying on texture repetition (wrapping) to create fine detail [S2].

3. **Apply Perturbation (Two Approaches):**

   * **Approach 1: Coordinate Perturbation (Geometry/Density Warping)** [S2]
     1. Take the original texture coordinates $\mathbf{T}$ for the volume data lookup [S2][S3].
     2. Use three components of the noise vector $\mathbf{D}$ (e.g., RGB) as a displacement vector [S2][S3].
     3. Compute perturbed coordinates:
        $$ \mathbf{T}_{perturbed} = \mathbf{T} + \mathbf{D} \cdot \text{scale} $$
     4. Perform a dependent texture read on the **Macrostructure Volume** using $\mathbf{T}_{perturbed}$ [S2][S3].
     5. Render the result to a pixel buffer (render-to-texture) to be used in subsequent shading steps [S2].
     6. *Note:* This avoids recomputing gradients on a perturbed grid, which is required for surface shading but not for volume raycasting/shading [S2].

   * **Approach 2: Optical Property Perturbation** [S2]
     1. Evaluate the transfer function on the original volume data to get base optical properties (color, opacity) [S2][S3].
     2. Use all four per-pixel noise components (RGBA) to modify these optical properties [S2][S3].
     3. The user selects which specific optical properties (e.g., opacity, color channels) are modified [S2].
     4. This creates inhomogeneities and subtle iridescence effects [S2].

4. **Animate:**
   * Add a varying offset vector to the noise texture coordinates in the fragment shader [S3].
   * Update the offset each frame using `time` and `flowSpeed` [S1, unofficial][S2].
   * Different offsets can be added to each noise texture copy to enhance motion complexity [S2].

**Pseudocode (Fragment Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_3d_texture(tex, coord):
    """Simulates 3D texture sampling with repeat addressing and trilinear filtering."""
    # Implementation depends on GPU API; returns RGBA tuple
    pass

def sample_1d_texture(tex, value):
    """Simulates 1D texture lookup for transfer function."""
    pass

def compute_perturbation_field(v_tex_coord, u_noise_volume, time, flow_speed, noise_scale):
    """Computes the weighted displacement vector from multi-scale noise samples."""
    perturbation = [0.0, 0.0, 0.0]
    
    # Scale 1: Large amplitude, low frequency
    coord1 = [v_tex_coord[0] * 0.5 + time * flow_speed, v_tex_coord[1], v_tex_coord[2]]
    noise1 = sample_3d_texture(u_noise_volume, coord1)
    perturbation[0] += noise1[0] * 2.0
    perturbation[1] += noise1[1] * 2.0
    perturbation[2] += noise1[2] * 2.0
    
    # Scale 2: Small amplitude, high frequency (via wrapping)
    coord2 = [v_tex_coord[0] * 2.0 + time * flow_speed * 0.5, v_tex_coord[1], v_tex_coord[2]]
    noise2 = sample_3d_texture(u_noise_volume, coord2)
    perturbation[0] += noise2[0] * 0.5
    perturbation[1] += noise2[1] * 0.5
    perturbation[2] += noise2[2] * 0.5
    
    return perturbation

def apply_perturbation(v_tex_coord, u_macro_volume, u_transfer_func, u_params, perturbation):
    """Applies either coordinate or optical perturbation based on flags."""
    coord_perturb = u_params[2]
    prop_perturb = u_params[3]
    noise_scale = u_params[1]
    
    if coord_perturb > 0.5:
        # Coordinate Perturbation: warp lookup coordinate
        perturbed_coord = [v_tex_coord[i] + perturbation[i] * noise_scale for i in range(3)]
        density = sample_3d_texture(u_macro_volume, perturbed_coord)[0]
        tf_result = sample_1d_texture(u_transfer_func, density)
        return tf_result
    else:
        # Optical Property Perturbation: modify optical properties after transfer function
        density = sample_3d_texture(u_macro_volume, v_tex_coord)[0]
        tf_result = sample_1d_texture(u_transfer_func, density)
        
        # Use noise components to modulate color and opacity
        noise_mod = sample_3d_texture(u_noise_volume, v_tex_coord)[0][0]
        color = [tf_result[i] * (1.0 + noise_mod * 0.2) for i in range(3)]
        opacity = tf_result[3] * (1.0 + noise_mod * 0.1)
        return [color[0], color[1], color[2], opacity]
```

**Edge Cases and Pitfalls:**
* **Trilinear Interpolation Artifacts:** The perturbation volume must be blurred slightly to prevent visible artifacts caused by linear interpolation of the random noise values [S2][S3].
* **Hardware Limits:** Modern GPUs limit the number of simultaneously applied textures and sequential dependent texture reads. This necessitates an additional render-to-texture pass to store the perturbed data before final shading [S2].
* **Gradient Recomputation:** If using surface shading on perturbed volumes, gradients must be recomputed based on the perturbed grid. This method avoids that cost by perturbing the data access location directly [S2].

## Subtopics

### [Fog Animation](4611-fog-animation.md)
Fog Animation generates the appearance of moving gaseous volumes by perturbing the spatial coordinates of the evaluation point before sampling a static turbulence function, effectively moving the point through a "solid space" texture. Unlike methods that animate the noise function itself, this approach creates motion by shifting the sampling domain, producing upward-swirling gas movement suitable for real-time fog and smoke. It builds on Procedural Volume Rendering with Noise Perturbation [1370-procedural-volume-rendering-with-noise-perturbation.md] by applying coordinate perturbation specifically to transparency-based fog effects.

## Operating characteristics
* **Time Complexity:** Adds a constant overhead per fragment for sampling the perturbation volume at multiple scales. The additional render-to-texture pass adds one full-screen pass per frame [S2].
* **Memory Complexity:** Requires storage for the small perturbation volume ($32^3 \times 4$ bytes = 128 KB) and potentially an intermediate render target for the perturbed data [S2].
* **Performance:** On hardware like NVIDIA GeForce 3 and ATI Radeon 8500, frame rates were only 50-60% slower than volume rendering with no shading, leveraging OpenGL render-to-texture extensions to avoid costly copy operations [S2].
* **Detail Equivalence:** The visual detail produced is equivalent to an unperturbed voxel dataset of at least 100 times the resolution [S2].
* **Strengths:** Efficiently adds high-frequency detail (clouds, smoke, fur) to coarse volumes without increasing voxel count or simulation cost [S2][S3].
* **Limitations:** Requires additional rendering passes due to texture read limits; artifacts can occur if the perturbation volume is not properly filtered [S2].

## Background
The technique was introduced by Kniss et al. in 2002 as part of "Interactive Translucent Volume Rendering and Procedural Modeling" [S2]. It adapts Ebert's lattice-based noise approach for clouds [S2] and Peachey's vector-based noise simulation [S2] to interactive volume rendering. It is detailed in GPU Gems Chapter 39 [S3] and implemented in various open-source projects [S1, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
