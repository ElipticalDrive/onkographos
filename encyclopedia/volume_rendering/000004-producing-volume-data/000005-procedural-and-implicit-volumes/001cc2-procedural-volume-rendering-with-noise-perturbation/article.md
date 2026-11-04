# Procedural Volume Rendering with Noise Perturbation

Also called: Texture Coordinate Perturbation, Noise-warped volume rendering, Procedural Volumetric Perturbation, Volume Perturbation via Procedural Noise, Volume Perturbation via Texture Coordinate Displacement

## Overview
Procedural Volume Rendering with Noise Perturbation enhances coarse, low-resolution volumetric data by adding high-frequency microstructure details (such as wispy cloud filaments, fur, or smoke turbulence) at render time, avoiding the memory and computational cost of increasing the underlying voxel resolution [S2, S3]. It sets itself apart from sibling methods like [Implicit Field Composition](2652-implicit-field-composition.md) by perturbing the sampling domain or optical properties of an existing discrete volume rather than defining the geometry via a continuous mathematical field, and from [Procedural Noise Functions](1799-procedural-noise-functions.md) by applying noise as a displacement or modulation vector to texture lookups rather than generating the scalar field directly [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A coarse 3D volume dataset (macrostructure), a small 3D perturbation noise volume, and rendering parameters (time, scales, weights).
*   **Output:** A rendered image with high-frequency detail superimposed on the coarse volume.

**Data Structures:**
1.  **Macrostructure Volume:** The base volume data (e.g., density field). Resolution is typically coarse (e.g., $64^3$ voxels) [S2].
2.  **Perturbation Noise Volume:** A small 3D texture.
    *   **Dimensions:** Typically $32 \times 32 \times 32$ [S2, S3].
    *   **Format:** 8-bit RGBA (e.g., `vk::Format::eR8G8B8A8Unorm` or OpenGL equivalent) [S1, unofficial][S2].
    *   **Initialization:** Each texel is initialized with four random 8-bit numbers [S2, S3].
    *   **Filtering:** Slightly blurred to hide artifacts caused by trilinear interpolation [S2, S3].
    *   **Addressing:** Set to repeat (wrap) to allow tiling [S1, unofficial][S2].
3.  **Parameters:**
    *   `time`: Animation time offset [S1, unofficial][S2].
    *   `noiseScale`: Scaling factor for the noise volume coordinates [S1, unofficial].
    *   `coordPerturb`: Amplitude for coordinate perturbation [S1, unofficial].
    *   `propPerturb`: Amplitude for property perturbation [S1, unofficial].
    *   `opacityCorrection`: Correction factor for opacity when slice count changes [S1, unofficial].
    *   `flowSpeed`: Speed of animation [S1, unofficial].

**Algorithm Steps:**
The method requires an additional rendering pass (or fragment shader logic) before the main volume shading/compositing pass due to hardware limitations on simultaneous texture bindings and dependent texture reads [S2].

1.  **Generate Perturbation Field:**
    *   Create multiple copies of the perturbation noise texture, each scaled by a different factor (e.g., low frequency/large amplitude and high frequency/small amplitude) [S2].
    *   For each copy, sample the noise texture using coordinates perturbed by time and scale.
    *   Compute the per-pixel perturbation vector $\mathbf{P}$ as the weighted sum of the individual noise components from all scales [S3].
    *   *Note:* For coordinate perturbation, use 3 components of the noise (RGB) as the vector $\mathbf{P}$. For optical perturbation, use 4 components (RGBA) [S2, S3].

2.  **Apply Perturbation (Two Approaches):**
    *   **Approach A: Coordinate Perturbation (Geometry/Density Warping)**
        1.  Calculate the original texture coordinates $\mathbf{C}_{orig}$ for the volume data lookup [S2].
        2.  Compute perturbed coordinates: $\mathbf{C}_{pert} = \mathbf{C}_{orig} + \mathbf{P} \cdot \text{scale}$ [S2, S3].
        3.  Perform a dependent texture read on the macrostructure volume using $\mathbf{C}_{pert}$ [S2].
        4.  Render the resulting perturbed data to a pixel buffer (render target) [S2].
        5.  Use this pixel buffer as the input for the subsequent volume shading/compositing pass [S2].
    *   **Approach B: Optical Property Perturbation**
        1.  Sample the macrostructure volume at $\mathbf{C}_{orig}$ to get base density/value [S2].
        2.  Evaluate the transfer function to get base optical properties (color, opacity) [S2].
        3.  Use the 4 noise components from $\mathbf{P}$ to modulate the optical properties (e.g., color or opacity) [S2, S3].
        4.  The user selects which specific optical properties are modified [S2].

3.  **Animation:**
    *   Update the noise texture coordinates by adding a time-based offset vector $\mathbf{T}(t)$ each frame [S2, S3].
    *   $\mathbf{C}_{noise}(t) = \mathbf{C}_{base} + \mathbf{T}(t)$ [S3].

**Pseudocode (Coordinate Perturbation Approach):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader / Geometry Setup
# Passes original texture coordinates C_orig to fragment shader

# Fragment Shader
uniform sampler3D macroVolume;
uniform sampler3D noiseVolume;
uniform vec3 timeOffset;
uniform float coordPerturbScale;
uniform vec3 noiseScales[4]; # Scales for multiple noise copies
uniform float noiseWeights[4];

void main() {
    vec3 C_orig = gl_TexCoord[0].xyz; // Original volume coordinates
    
    // Step 1: Compute perturbation vector P
    vec3 P = vec3(0.0);
    for (int i = 0; i < 4; i++) {
        // Sample noise at scaled and time-offset coordinates
        vec3 noiseCoord = C_orig * noiseScales[i] + timeOffset;
        vec4 noiseVal = texture3D(noiseVolume, noiseCoord);
        
        // Use RGB components for displacement vector
        P += noiseVal.rgb * noiseWeights[i];
    }
    
    // Step 2: Apply perturbation
    vec3 C_pert = C_orig + P * coordPerturbScale;
    
    // Step 3: Dependent texture read
    // Note: In OpenGL, this might require a separate render-to-texture pass
    // if dependent reads are limited. Here we assume direct access for clarity.
    float density = texture3D(macroVolume, C_pert).r;
    
    // Step 4: Standard volume shading/compositing using 'density'
    // ... (Transfer function, lighting, etc.)
    gl_FragColor = vec4(density, density, density, 1.0);
}
```

**Edge Cases and Pitfalls:**
*   **Trilinear Interpolation Artifacts:** The blurring of the noise volume is critical to prevent visible grid artifacts when sampling the perturbation volume [S2, S3].
*   **Hardware Limits:** Limited number of simultaneous texture units and dependent texture reads may necessitate an intermediate render-to-texture pass to store the perturbed volume data before the final shading pass [S2].
*   **Gradient Recomputation:** Unlike surface shading on perturbed volumes, this method avoids the need to recompute gradients based on the perturbed grid, simplifying lighting calculations [S2].

## Subtopics

### [Fog Animation](4611-fog-animation.md)
Fog Animation generates the appearance of moving gaseous volumes by perturbing the spatial coordinates of the evaluation point before sampling a static turbulence function, effectively moving the point through a "solid space" texture. Unlike methods that animate the noise function itself, this approach creates motion by shifting the sampling domain, producing upward-swirling gas movement suitable for real-time fog and smoke. It is a specific application of Procedural Volume Rendering with Noise Perturbation [1370-procedural-volume-rendering-with-noise-perturbation.md] tailored for transparency-based fog effects.

## Operating characteristics
*   **Time Complexity:** Adds a constant overhead per fragment for sampling the noise volume(s) and computing the perturbation vector. The number of noise samples depends on the number of scales used (typically 2-4) [S2].
*   **Memory Complexity:** Requires storage for the coarse volume and the small perturbation noise volume ($32^3$ texels is negligible compared to high-res volumes) [S2].
*   **Performance:** On hardware like NVIDIA GeForce 3 and ATI Radeon 8500, frame rates were only 50-60% slower than volume rendering with no shading, demonstrating high efficiency [S2].
*   **Strengths:** Produces detail equivalent to an unperturbed voxel dataset of at least 100 times the resolution [S2]. Efficiently adds high-frequency details (clouds, fur, smoke) without increasing voxel count [S2, S3]. Avoids gradient recomputation issues associated with surface shading on perturbed grids [S2].
*   **Limitations:** Requires additional rendering passes or fragment shader complexity due to texture read limits [S2]. The visual quality is heavily dependent on the underlying shading model; without realistic shading, results may look like deformed blobs [S2].
*   **When to Use:** Real-time rendering of clouds, smoke, fire, fur, or any volumetric effect where high-frequency detail is desired but memory/bandwidth for high-resolution voxels is limited [S1, unofficial][S2].
*   **When Not to Use:** When exact physical simulation of density fields is required, or when the perturbation artifacts (if blurring is insufficient) become distracting.

## Background
The technique was introduced by Kniss et al. in 2002 as part of "Interactive Translucent Volume Rendering and Procedural Modeling" [S2]. It builds on earlier work by Ebert (lattice-based noise for clouds) and Peachey (vector-based noise simulation) [S2]. It was further detailed in GPU Gems Chapter 39 (Kniss et al., 2003) [S3]. The method was implemented on early GPUs (GeForce 3, Radeon 8500) leveraging OpenGL extensions for render-to-texture [S2].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Translucent Volume Rendering and Procedural Modeling, https://courses.cs.duke.edu/spring03/cps296.8/papers/kniss_volumeshading02.pdf, fetched 2026-10-26
[S3] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
