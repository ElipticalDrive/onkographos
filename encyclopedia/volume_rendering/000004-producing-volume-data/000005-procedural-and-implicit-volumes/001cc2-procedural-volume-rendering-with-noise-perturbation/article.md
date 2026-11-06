# Procedural Volume Rendering with Noise Perturbation

Also called: Texture Coordinate Perturbation, Noise-warped volume rendering, Procedural Volumetric Perturbation, Volume Perturbation via Procedural Noise, Volume Perturbation via Texture Coordinate Displacement

## Overview
Procedural Volume Rendering with Noise Perturbation enhances coarse, low-resolution volumetric data by superimposing high-frequency details using procedural noise, creating wispy, cloud-like filaments suitable for real-time clouds, smoke, and fire [S1, unofficial][S2, S3]. It sets itself apart from sibling methods like [Implicit Surface Primitives](1800-implicit-surface-primitives.md) or [Distance to Hypertextures](1555-distance-to-hypertextures.md) by avoiding the computational cost of increasing voxel resolution or recomputing geometric gradients; instead, it warps the sampling domain or modulates optical properties at render time [S2]. It differs from [Procedural Noise Functions](1799-procedural-noise-functions.md) by specifically targeting volume access coordinates or transfer function outputs rather than generating the base field itself.

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse volumetric dataset (macrostructure) and a small 3D perturbation volume (microstructure).
*   **Output:** A rendered image with high-frequency detail that appears equivalent to a much higher resolution dataset [S2].

**Data Structures:**
1.  **Macrostructure Volume:** The base volume data (e.g., density field).
2.  **Perturbation Volume:** A small 3D texture.
    *   **Dimensions:** Typically $32 \times 32 \times 32$ [S2, S3].
    *   **Format:** RGBA, with each component being an 8-bit unsigned integer (e.g., `vk::Format::eR8G8B8A8Unorm`) [S1, unofficial][S2, S3].
    *   **Initialization:** Each voxel is initialized with four random 8-bit numbers [S2, S3].
    *   **Filtering:** The volume is blurred slightly to hide artifacts caused by trilinear interpolation [S2, S3].
    *   **Addressing:** Set to repeat (wrap) mode to allow tiling [S1, unofficial][S2].

**Parameters:**
*   `kNoiseDim`: Dimension of the cubic noise volume (e.g., 32) [S1, unofficial].
*   `noiseScale`: Controls the amplitude of the perturbation [S1, unofficial].
*   `coordPerturb`: Boolean/flag to enable coordinate perturbation [S1, unofficial].
*   `propPerturb`: Boolean/flag to enable optical property perturbation [S1, unofficial].
*   `opacityCorrection`: Scaling factor for opacity, often calculated as $N_{ref} / N_{curr}$ where $N_{ref}$ is a reference slice count and $N_{curr}$ is the current slice count [S1, unofficial].
*   `flowSpeed`: Speed of animation [S1, unofficial].
*   **Animation Offset:** A varying vector added to noise texture coordinates per frame [S2, S3].

**Steps:**
The method requires an additional rendering pass before the main volume shading steps due to hardware limitations on simultaneous texture bindings and dependent texture reads [S2].

1.  **Generate Perturbation Field:**
    *   Create multiple copies of the perturbation volume texture.
    *   Apply these copies to the slice at different scales [S2, S3].
    *   Weight and sum the noise components per pixel to create a displacement vector field (for coordinate perturbation) or a modulation vector (for property perturbation) [S2, S3].
    *   *Note:* Low-frequency perturbation is achieved by applying the texture once with large amplitude. High-frequency perturbation is achieved by applying the texture again with low amplitude and allowing the texture to repeat [S2].

2.  **Apply Perturbation (Choose One Approach):**

    **Approach 1: Coordinate Perturbation (Geometry/Density Warping)**
    1.  Use three components (e.g., RGB) of the computed noise vector as a displacement vector [S2, S3].
    2.  Add this displacement vector to the original texture coordinates of the volume data lookup [S2, S3].
    3.  Read the volume data using the perturbed coordinates via dependent texture reads [S2].
    4.  Render the perturbed data to a pixel buffer (render-to-texture) [S2].
    5.  Use this perturbed buffer as the input for the subsequent volume shading/compositing pass [S2].

    **Approach 2: Optical Property Perturbation**
    1.  Evaluate the transfer function on the original volume data to get base optical properties (color, opacity) [S2, S3].
    2.  Use the four per-pixel noise components (RGBA) to modify these optical properties [S2, S3].
    3.  The user can select which specific optical properties (e.g., color channels, opacity) are modified [S2].
    4.  This creates inhomogeneities and subtle iridescence effects [S2].

3.  **Animate:**
    *   Add a time-varying offset vector to the noise texture coordinates in the fragment shader [S2, S3].
    *   Update the offset each frame to scroll the noise over time [S1, unofficial][S2].

**Pseudocode (Coordinate Perturbation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Geometry Setup
# Renders a slice or ray-cast geometry. Passes original UVs (tex_coord) to fragment shader.

# Fragment Shader
uniform sampler3D volumeData;       // Coarse macrostructure
uniform sampler3D noiseVolume;      // 32x32x32 perturbation volume
uniform vec3 noiseOffset;           // Animation offset (time-based)
uniform vec3 noiseScale;            // Amplitude of perturbation

void main() {
    // 1. Sample noise volume at perturbed coordinates
    // The noise volume is small and tiled.
    vec3 noiseUV = tex_coord + noiseOffset;
    vec4 noiseVal = texture(noiseVolume, noiseUV);
    
    // 2. Construct displacement vector from RGB components
    // Map 8-bit values [0,1] to a centered range or direct offset
    vec3 displacement = (noiseVal.rgb - 0.5) * noiseScale;
    
    // 3. Perturb the lookup coordinates for the volume data
    vec3 perturbedUV = tex_coord + displacement;
    
    // 4. Sample the coarse volume with perturbed coordinates
    // This is a dependent texture read
    float density = texture(volumeData, perturbedUV).r;
    
    // 5. Continue with standard volume shading/compositing using 'density'
    // ...
}
```

**Pseudocode (Optical Perturbation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader
uniform sampler3D volumeData;
uniform sampler3D noiseVolume;
uniform vec3 noiseOffset;
uniform vec3 noiseScale;

void main() {
    // 1. Sample noise
    vec3 noiseUV = tex_coord + noiseOffset;
    vec4 noiseVal = texture(noiseVolume, noiseUV);
    
    // 2. Sample base volume
    float density = texture(volumeData, tex_coord).r;
    
    // 3. Evaluate Transfer Function (TF) to get base color/opacity
    vec4 baseColor = evaluateTransferFunction(density);
    
    // 4. Modulate optical properties using noise
    // Example: Modify color channels or opacity
    vec4 perturbedColor = baseColor;
    perturbedColor.rgb *= (1.0 + (noiseVal.rgb - 0.5) * noiseScale);
    perturbedColor.a *= (1.0 + (noiseVal.a - 0.5) * noiseScale);
    
    // 5. Composite perturbedColor
    // ...
}
```

**Edge Cases and Pitfalls:**
*   **Trilinear Interpolation Artifacts:** The small size of the perturbation volume can cause visible artifacts when sampled with trilinear filtering. Mitigated by blurring the perturbation volume slightly [S2, S3].
*   **Hardware Limits:** Limitations on the number of simultaneously applied textures and sequential dependent texture reads require the additional pre-pass (rendering to a pixel buffer) for coordinate perturbation [S2].
*   **Gradient Recomputation:** Unlike surface shading on perturbed volumes, this method avoids the need to recompute gradients based on the perturbed grid, which is computationally expensive [S2].

## Subtopics

### [Fog Animation](4611-fog-animation.md)
Fog Animation generates the appearance of moving gaseous volumes by perturbing the spatial coordinates of the evaluation point before sampling a static turbulence function, effectively moving the point through a "solid space" texture. Unlike methods that animate the noise function itself, this approach creates motion by shifting the sampling domain, producing upward-swirling gas movement suitable for real-time fog and smoke. It is a specific application of [Procedural Volume Rendering with Noise Perturbation](1370-procedural-volume-rendering-with-noise-perturbation.md) tailored for transparency-based fog effects.

## Operating characteristics
*   **Time Complexity:** Adds a constant overhead of one additional rendering pass (for coordinate perturbation) or minor shader computation (for optical perturbation) per frame [S2].
*   **Memory Complexity:** Requires storage for the small perturbation volume ($32^3$ texels) and potentially an intermediate render target buffer [S2].
*   **Performance:** On hardware like NVIDIA GeForce 3 and ATI Radeon 8500, frame rates were only 50-60% slower than volume rendering with no shading, leveraging OpenGL render-to-texture extensions to avoid costly copy operations [S2].
*   **Strengths:** Produces detail equivalent to an unperturbed voxel dataset of at least 100 times the resolution [S2]. Efficiently adds high-frequency details (clouds, smoke, fur) to coarse models without increasing voxel count [S1, unofficial][S2]. Avoids gradient recomputation costs [S2].
*   **Limitations:** Requires additional passes due to texture read limits [S2]. The visual quality is heavily dependent on the underlying shading model; without realistic shading, clouds may look like deformed blobs [S2].
*   **When to Use:** Real-time rendering of clouds, smoke, fire, fur, or moss where high resolution is prohibitive [S1, unofficial][S2].
*   **When Not to Use:** When exact physical accuracy of the density field is required, or when hardware does not support dependent texture reads or render-to-texture efficiently.

## Background
Originating from work by Kniss et al. (2002/2003) and Ebert et al. (2002), this technique adapts procedural noise simulation for interactive volume rendering [S2, S3]. It builds on Ebert's approach of using coarse techniques for macrostructure and procedural noise for microstructure [S2]. It is closely related to Peachey's vector-based noise simulation technique [S2]. It is documented in *GPU Gems* Chapter 39 and the paper *Interactive Translucent Volume Rendering and Procedural Modeling* [S2, S3].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
