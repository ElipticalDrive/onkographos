# Noise Field Generator

Also called: Layered Simplex Noise with Gradient, SimplexNoise3D

## Overview
The Noise Field Generator creates a 3D scalar field for volumetric rendering by summing multiple octaves of layered Simplex noise, offset by a linear vertical gradient. It solves the problem of generating organic, multi-scale density distributions (such as soft clouds or terrain) without requiring pre-baked texture data. It distinguishes itself from its parent, [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md), by explicitly incorporating a vertical gradient bias and specific Simplex noise variants into the layering process, rather than relying solely on generic hash-based Perlin noise.

## How it works
**Inputs:**
- `Dims`: 3D integer vector defining the voxel grid dimensions $(D_x, D_y, D_z)$.
- `Scale`: Scalar spatial frequency multiplier for the noise field.
- `Time`: Scalar time offset for animated noise.
- `Voxels`: Output buffer (e.g., `RWStructuredBuffer<float>`) of size $D_x \times D_y \times D_z$.

**Data Structures:**
- The output is a flat 1D array representing the 3D grid. The index for a voxel at $(x, y, z)$ is calculated as:
  $$ \text{index} = x + D_x \times (y + D_y \times z) $$
- Thread groups are processed in blocks of $[8, 8, 8]$ threads.

**Algorithm:**
1.  **Thread Identification:** Each thread computes its global 3D coordinate `id` $(x, y, z)$ based on its group ID and local thread ID.
2.  **Position Calculation:** Compute the centered world position `p` for the voxel:
    $$ p = \text{Scale} \times \left( \text{id} + 0.5 - \frac{\text{Dims}}{2} \right) $$
    This centers the noise field around the origin $(0,0,0)$.
3.  **Gradient Initialization:** Initialize the scalar `value` with a vertical gradient. The gradient typically biases the noise towards the negative Y-axis (or positive, depending on coordinate system convention), often used to simulate gravity or density stratification in clouds:
    $$ \text{value} = -p.y $$
4.  **Octave Summation:** Add multiple octaves of Simplex noise. Each octave uses a different frequency scaling, spatial offset, and amplitude.
    -   **First Octave:**
        $$ \text{value} \leftarrow \text{value} + \text{SimplexNoise}(p \times 0.83 + \mathbf{v}_1 \times \text{Time}) \times 0.5 $$
        Where $\mathbf{v}_1 = (-0.37, -0.12, 0)$.
    -   **Second Octave:**
        $$ \text{value} \leftarrow \text{value} + \text{SimplexNoise}(p \times 0.72 + \mathbf{v}_2 \times \text{Time}) \times 0.5 $$
        Where $\mathbf{v}_2 = (0.21, 0.07, 0)$.
    -   *Note:* Sources indicate a 2-octave structure for this specific "SimplexNoise3D" variant [S2, unofficial]. Other FBM implementations may loop for more octaves [S3][S4, unofficial].
5.  **Output:** Write the computed `value` to the `Voxels` buffer at the calculated index.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel
def noise_field_generator_kernel(id: int3, Dims: int3, Scale: float, Time: float, Voxels: array):
    # 1. Calculate centered world position
    # id is the global thread ID (x, y, z)
    p_x = Scale * (id.x + 0.5 - Dims.x / 2.0)
    p_y = Scale * (id.y + 0.5 - Dims.y / 2.0)
    p_z = Scale * (id.z + 0.5 - Dims.z / 2.0)
    p = (p_x, p_y, p_z)
    
    # 2. Initialize with vertical gradient
    value = -p.y
    
    # 3. Add First Octave of Simplex Noise
    # Offset vector v1 = (-0.37, -0.12, 0)
    offset1 = (-0.37 * Time, -0.12 * Time, 0.0)
    freq1 = 0.83
    sample1 = simplex_noise_3d((p.x * freq1 + offset1.x, 
                                p.y * freq1 + offset1.y, 
                                p.z * freq1 + offset1.z))
    value += sample1 * 0.5
    
    # 4. Add Second Octave of Simplex Noise
    # Offset vector v2 = (0.21, 0.07, 0)
    offset2 = (0.21 * Time, 0.07 * Time, 0.0)
    freq2 = 0.72
    sample2 = simplex_noise_3d((p.x * freq2 + offset2.x, 
                                p.y * freq2 + offset2.y, 
                                p.z * freq2 + offset2.z))
    value += sample2 * 0.5
    
    # 5. Write to output buffer
    # Flat index calculation: x + Dx * (y + Dy * z)
    index = id.x + Dims.x * (id.y + Dims.y * id.z)
    Voxels[index] = value

# Helper: Simplex Noise 3D
# Returns a value in range [-1, 1] or [0, 1] depending on implementation.
# Standard Simplex Noise uses gradient vectors at grid corners and trilinear interpolation.
def simplex_noise_3d(p: tuple):
    # Implementation details of Simplex Noise 3D are assumed to be available 
    # from the parent article [1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md].
    # It involves:
    # 1. Finding the simplex cell containing p.
    # 2. Computing contributions from the 4 corners of the cell.
    # 3. Summing contributions weighted by distance factors.
    pass
```

**Parameters:**
- `Scale`: Spatial scaling factor. Typical values depend on the desired world units; [S2, unofficial] uses a generic `Scale` variable.
- `Time`: Animation speed. Multiplies the spatial offset vectors.
- `Octaves`: The number of noise layers. The specific "SimplexNoise3D" variant in [S2, unofficial] uses 2 octaves. General FBM implementations use 4–6 octaves [S3][S4, unofficial].
- `Gain`: Amplitude multiplier per octave. In [S2, unofficial], both octaves use a gain of 0.5. In standard FBM [S3], Gain is typically 0.5.
- `Lacunarity`: Frequency multiplier per octave. In [S2, unofficial], frequencies are hardcoded (0.83, 0.72). In standard FBM [S3], Lacunarity is typically ~2.0.

**Edge Cases and Pitfalls:**
- **Precision:** Using `float` for position calculations may lead to precision loss at large world coordinates. Centering the grid around the origin (as done in Step 2) mitigates this.
- **Boundary Artifacts:** Without inter-octave rotation or varying lacunarity, axis-aligned artifacts may appear [S3]. The specific Simplex variant in [S2, unofficial] avoids this by using distinct spatial offsets for each octave.
- **Gradient Direction:** The vertical gradient `-p.y` assumes Y is up. If Z is up, the gradient should be `-p.z`.

## Subtopics

### [FBM Combinations](3962-fbm-combinations.md)
Integrates Fractal Brownian Motion noise fields with rendering techniques such as raymarching, lighting models, and color mapping. It focuses on deriving density, normals, and color from layered noise samples for use in raymarched scenes, building on the raw noise generation of the Noise Field Generator by applying intersection and shading logic.

### [Terrain Heightmap Generation via Noise](4214-terrain-heightmap-generation-via-noise.md)
Generates a 2D terrain heightmap by sampling a 3D scalar noise field at a fixed vertical coordinate (e.g., $z=0$). It projects the layered noise onto the X-Y plane to produce continuous elevation data, differing from the full 3D volume generation by ignoring the vertical gradient offset and returning a single scalar height per X/Y coordinate.

### [Domain Warped FBM](4493-domain-warped-fbm.md)
A technique where the output of a noise function is fed back as a coordinate offset to distort the sampling domain, creating organic, fluid-like patterns. It differs from standard FBM by introducing non-linear spatial distortion that mimics turbulent flow, using the noise amplitude itself to drive the displacement.

### [Turbulence](17000-turbulence.md)
A noise basis function that creates the visual impression of Brownian motion by summing the absolute values of noise at decreasing frequencies. It differs from standard FBM by applying the absolute value operator to each octave, introducing discontinuities in the gradient and creating a "carved" or fragmented appearance.

### Absolute Noise Sum
A fractal noise variant that sums the absolute values of Perlin noise octaves. It results in a range of 0.0 to 1.0 and creates "cloudy" or "rough" textures. It differs from standard FBM by using an initial amplitude of 1.0 and the `abs()` function on each octave sample [S6, unofficial].

### Ocean Wave Octave
A specialized noise function for simulating ocean waves. It computes wave components by adding noise to UV coordinates, calculating sine/cosine components (`wv`, `swv`), and mixing them with a "choppy" factor. It supports bidirectional propagation by summing results for time-shifted UVs [S3].

### Bounded Perlin Noise Texture
A texture class that maps 3D points to RGB colors using Perlin noise, scaled and bounded within a specific region. It stores color, min/max bounds, scale, and a Perlin noise instance, blending the noise result with a base color [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \times O)$, where $N$ is the number of voxels and $O$ is the number of octaves. Each voxel requires $O$ noise evaluations.
- **Memory Complexity:** $O(N)$ for the output voxel grid.
- **Performance:** More computationally intensive than single-layer noise due to multiple noise samples per voxel [S5, unofficial]. Performance can be optimized by reducing the octave count for distant objects or using LOD [S3].
- **Strengths:** Produces realistic, organic, multi-scale details suitable for clouds and terrain. The vertical gradient adds physical plausibility to density distributions.
- **Limitations:** Requires significant computation for high-resolution volumes. The specific Simplex variant in [S2, unofficial] is limited to 2 octaves, which may lack fine detail compared to higher-octave FBM.

## Background
The Noise Field Generator is a specific implementation of procedural noise generation for volumetric data, often used in real-time graphics and GPU-based terrain/cloud generation. It builds on the principles of Fractal Brownian Motion (FBM) and Simplex Noise. The specific variant with vertical gradient and layered Simplex noise is documented in GPU-based marching cubes and volumetric raymarching tutorials [S2, S4, unofficial]. The concept of FBM was popularized by Ken Perlin and others for procedural texture synthesis [S3][S5, unofficial].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] keijiro/ComputeMarchingCubes, https://github.com/keijiro/ComputeMarchingCubes, fetched 2026-10-26 (unofficial)
[S3] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S4] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S5] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S6] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S7] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
