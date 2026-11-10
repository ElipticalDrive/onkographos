# 3D Perlin-Worley Noise Texture Generation

Also called: Low-Frequency Cloud Shape Generation, Multi-Octave Worley, Noise-based Density Sampling, Perlin-Worley Hybrid, Volumetric Density Map Generation, sampleLowFrequency

## Overview
This method generates 3D volumetric density maps for procedural cloud rendering by combining low-frequency Perlin noise for broad cloud mass with inverted Worley (cellular) noise to carve hollows, producing a puffy cumulus appearance [S3, unofficial]. It differs from pure Perlin noise methods by introducing cellular structures that define cloud edges and internal voids, and from pure Worley methods by retaining the organic, continuous mass of gradient-based noise [S3, unofficial]. It builds on [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) by using fBm principles to sum multiple noise octaves.

## How it works
**Inputs:**
- Input position $P$ (3D vector).
- Texture size $T$ (3D integer vector, e.g., $256^3$).
- Global invocation ID $G$ (3D integer vector from compute shader).
- Cloud coverage parameter (hardcoded to 0.6 in source implementations) [S2, unofficial].

**Outputs:**
- A 3D texture (e.g., `rgba8`) storing density values. The Red channel typically stores the base cloud shape, while Green, Blue, and Alpha channels may store multi-octave Worley noise for detail erosion [S3, unofficial].

**Data Structures:**
- `u_NoiseTexture`: A `writeonly uniform image3D` bound at binding point 0. Layout is `rgba8`, meaning 8-bit unsigned integer per channel [S3, unofficial].
- `cloudBaseShapeSampler`: A 3D texture sampler used to sample pre-computed or generated low-frequency noise fields [S2, unofficial].

**Steps:**
1.  **Coordinate Calculation:** Compute fractional coordinates for the current voxel by dividing the global invocation ID by the texture size: $C = G / T$. Add an offset and scale the position if required by the specific noise implementation [S1, S3, unofficial].
2.  **Sample Low-Frequency Base Shape:** Sample the `cloudBaseShapeSampler` at the fractional coordinates to retrieve a `vec4` of low-frequency noise values: $L = \text{texture}(cloudBaseShapeSampler, C)$ [S2, unofficial].
3.  **Compute Low-Frequency FBM:** Construct a fractal sum from the Green, Blue, and Alpha channels of $L$ using specific weights:
    $$ L_{FBM} = (L.g \times 0.625) + (L.b \times 0.25) + (L.a \times 0.125) $$
    Clamp the result to $[0.0, 1.0]$ [S2, unofficial].
4.  **Generate Base Cloud Density:** Remap the Red channel of $L$ ($L.r$) using the computed $L_{FBM}$ to create the base cloud density. The source uses a function `remapClamped` with parameters derived from $L_{FBM}$:
    $$ D_{base} = \text{remapClamped}(L.r, L_{FBM} - 0.9, 1.0, 0.0, 1.0) $$
    This step effectively thresholds and scales the noise to define where clouds exist [S2, unofficial].
5.  **Generate Worley Noise (Inverted):** Sample Worley noise at frequency 1.0 and amplitude 1.0. Invert the result (subtract from 1.0 or negate, depending on implementation) and scale to the range $[-1, 1]$. Store this in a variable $W$ [S1, unofficial].
6.  **Generate Multi-Octave Perlin Noise:** Sample Perlin noise at 6 different frequencies and amplitudes to create a fractal sum. The specific octaves are:
    - Frequency 1, Amplitude 8
    - Frequency 2, Amplitude 4
    - Frequency 4, Amplitude 2
    - Frequency 8, Amplitude 1
    - Frequency 16, Amplitude 0.5
    - Frequency 32, Amplitude 0.25
    Sum these values to get $P_{sum}$ [S1, unofficial].
7.  **Combine and Finalize Density:** Sum the Perlin noise layers ($P_{sum}$) and the inverted Worley noise ($W$). Subtract a constant offset of 0.2 and clamp the result to a minimum of 0:
    $$ D = \max(0, P_{sum} + W - 0.2) $$
    This formula represents the core density calculation [S1, unofficial].
8.  **Write Output:** Store the computed density (and potentially the multi-octave Worley channels for erosion) into the `u_NoiseTexture` using `imageStore()` at the index corresponding to $G$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume G is the global invocation ID (ivec3)
# Assume T is the texture size (ivec3)
# Assume noise functions: perlin_noise(pos, freq, amp), worley_noise(pos, freq, amp)

def generate_perlin_worley_noise(G, T):
    # 1. Calculate fractional coordinates
    C = G / T
    
    # 2. Sample low-frequency base shape (simulated texture lookup)
    # In a real implementation, this reads from a pre-generated 3D texture
    L = sample_cloud_base_shape(C) # Returns vec4 (r, g, b, a)
    
    # 3. Compute Low-Frequency FBM
    low_frequency_fbm = (L.g * 0.625) + (L.b * 0.25) + (L.a * 0.125)
    low_frequency_fbm = clamp(low_frequency_fbm, 0.0, 1.0)
    
    # 4. Define Base Cloud
    # remapClamped(value, min_val, max_val, out_min, out_max)
    # Source: remapClamped(L.r, low_frequency_fbm - 0.9, 1.0, 0.0, 1.0)
    base_cloud = remap_clamped(L.r, low_frequency_fbm - 0.9, 1.0, 0.0, 1.0)
    
    # 5. Generate Inverted Worley Noise (Frequency 1.0, Amplitude 1.0)
    worley_val = worley_noise(C, 1.0, 1.0)
    worley_inverted = invert_and_scale(worley_val) # Scale to [-1, 1]
    
    # 6. Generate Multi-Octave Perlin Noise
    # Octaves: (freq, amp)
    octaves = [
        (1, 8), (2, 4), (4, 2), (8, 1), (16, 0.5), (32, 0.25)
    ]
    perlin_sum = 0.0
    for freq, amp in octaves:
        perlin_sum += perlin_noise(C, freq, amp)
        
    # 7. Combine and Finalize Density
    # Formula: D = max(0, sum(P_i) + W - 0.2)
    density = max(0.0, perlin_sum + worley_inverted - 0.2)
    
    # 8. Return density and potentially multi-octave worley for erosion
    # Note: The source mentions storing 3 octaves of Worley in G, B, A channels
    # for a secondary detail texture. This pseudocode focuses on the primary density.
    return density

def remap_clamped(value, min_val, max_val, out_min, out_max):
    # Linear remapping with clamping
    if max_val == min_val:
        return out_min
    t = clamp((value - min_val) / (max_val - min_val), 0.0, 1.0)
    return out_min + t * (out_max - out_min)

def invert_and_scale(worley_val):
    # Assuming worley_noise returns [0, 1]
    inverted = 1.0 - worley_val
    # Scale to [-1, 1]
    return inverted * 2.0 - 1.0
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `rgba8` for the output texture limits density resolution to 256 levels per channel. This may cause banding in smooth gradients. [G] Use `rgba16f` or `rgba32f` if higher precision is required for raymarching.
- **Boundary Conditions:** The noise functions must handle periodic or seamless wrapping if the cloud layer is expected to tile or wrap around a sphere. [G]
- **Degenerate Input:** If the camera is inside the cloud layer, the ray start point must be clamped to the camera position to avoid sampling errors at the entry point [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the 3D texture. Each voxel requires 6 Perlin noise evaluations and 1 Worley noise evaluation, plus texture sampling for the base shape.
- **Memory Complexity:** $O(N)$ for the 3D texture storage. For a $256^3$ texture with `rgba8`, this is $256^3 \times 4$ bytes $\approx 64$ MB [S3, unofficial].
- **Measured Performance:** Not given in the sources. [S1, S2, S3, unofficial]
- **Strengths:** Produces visually appealing, puffy cumulus clouds with natural-looking hollows and edges due to the Worley component. The separation of base shape and detail erosion allows for flexible tuning.
- **Limitations:** The fixed octaves and weights are hardcoded, reducing flexibility. The `rgba8` output may limit dynamic range. The method is computationally expensive due to multiple noise evaluations per voxel.
- **When to use:** Ideal for real-time volumetric cloud rendering where visual quality is prioritized over absolute physical accuracy.
- **When not to use:** When high-precision density maps are required or when the scene geometry does not support the spherical cloud layer model.

## Background
The technique originates from implementations of volumetric clouds in OpenGL, specifically described in "Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL" [S3, unofficial]. It combines two distinct noise types: Perlin noise for organic, continuous mass and Worley noise for cellular, void-like structures. The method was popularized in community-driven WebGL and OpenGL tutorials [S1, S2, S3, unofficial]. The specific parameters (octaves, weights) are derived from practical implementations aimed at achieving a "puffy cumulus appearance" [S3, unofficial].

## Sources
[S1] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
