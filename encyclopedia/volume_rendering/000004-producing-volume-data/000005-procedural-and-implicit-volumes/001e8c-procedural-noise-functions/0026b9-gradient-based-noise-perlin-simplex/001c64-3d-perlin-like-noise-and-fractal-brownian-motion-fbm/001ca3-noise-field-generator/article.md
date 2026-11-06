# Noise Field Generator

Also called: Layered Simplex Noise with Gradient, SimplexNoise3D

## Overview
The Noise Field Generator produces a 3D scalar field for volumetric rendering by summing multiple octaves of gradient-based noise (typically Simplex or Perlin) with varying frequencies and amplitudes, often combined with a linear vertical gradient [S2, unofficial][S3][S4, unofficial]. It solves the problem of generating realistic, multi-scale organic structures (such as clouds or terrain) in procedural volumes [S3][S5, unofficial]. It is distinguished from [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) by its specific implementation of layered Simplex noise with vertical gradient offsets for volumetric density, and from [Global Cloud Cover with Curl Noise](1711-global-cloud-cover-with-curl-noise.md) by its reliance on scalar summation rather than divergence-free vector fields [S2, unofficial][S3].

## How it works
**Inputs:**
- `Dims`: 3D integer vector defining the voxel grid dimensions $(D_x, D_y, D_z)$ [S2, unofficial].
- `Scale`: Spatial scaling factor for the noise field [S2, unofficial].
- `Time`: Optional float for animating the noise field [S2, S4, unofficial].
- `Octaves`: Integer count of noise layers (typically 4–6) [S3][S4, S5, unofficial].
- `Gain`: Amplitude multiplier per octave (typically 0.5) [S3][S5, unofficial].
- `Lacunarity`: Frequency multiplier per octave (typically 2.0 or varying 2.01–2.03) [S3][S4, S5, unofficial].

**Outputs:**
- `Voxels`: A flat 1D array or structured buffer of floats, indexed by $id.x + D_x \cdot (id.y + D_y \cdot id.z)$, representing the scalar density field [S1, S2, unofficial].

**Data Structures:**
- The output is a contiguous memory buffer of floating-point values. In GPU implementations, this is often an `RWStructuredBuffer<float>` [S2, unofficial].
- Thread groups are typically sized $[8, 8, 8]$ for parallel voxel generation [S2, unofficial].

**Algorithm:**
The generator iterates over every voxel in the grid. For each voxel, it computes a centered world position, initializes a value accumulator with a vertical gradient, and then sums the contributions of $N$ octaves of noise. Each octave scales the input coordinates by the lacunarity and multiplies the noise output by the gain. To prevent axis-aligned artifacts, the coordinate space is rotated or the lacunarity is varied between octaves [S3].

**Formulas:**
1.  **Centered Position:**
    $$ p = \text{Scale} \cdot (id + 0.5 - \frac{\text{Dims}}{2}) $$
    where $id$ is the integer voxel coordinate [S2, unofficial].

2.  **Vertical Gradient Initialization:**
    $$ \text{value} = -p.y $$
    This biases the noise field vertically, useful for cloud layers [S2, unofficial].

3.  **FBM Accumulation:**
    $$ \text{value} \leftarrow \text{value} + \sum_{i=0}^{\text{Octaves}-1} \text{Gain}^i \cdot \text{Noise}(p_i) $$
    where $p_i$ is the transformed coordinate for octave $i$.

**Pseudocode:**
The following implementation combines the gradient initialization from [S2, unofficial] with the rotation-based FBM from [S3] and the time-offset variation from [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def noise_3d(x, y, z):
    """
    Placeholder for a gradient-based noise function (e.g., Simplex or Perlin).
    Returns a value in [-1, 1] or [0, 1] depending on implementation.
    For this example, we assume a standard noise returning [-1, 1].
    """
    # Implementation details of the underlying hash/interpolation are in [S3]
    pass

def generate_noise_field(Dims, Scale, Time, Octaves, Gain, UseRotation=True):
    """
    Generates a 3D scalar field using layered noise with vertical gradient.
    
    Args:
        Dims: tuple (Dx, Dy, Dz)
        Scale: float
        Time: float
        Octaves: int
        Gain: float
        UseRotation: bool, whether to use coordinate rotation to break artifacts
        
    Returns:
        list: Flat list of float values
    """
    Dx, Dy, Dz = Dims
    total_voxels = Dx * Dy * Dz
    voxels = [0.0] * total_voxels
    
    # Precompute rotation matrix components if needed
    # mat2 m = mat2(1.6, 1.2, -1.2, 1.6) from [S3]
    # det(m) = 1.6*1.6 - (1.2*-1.2) = 2.56 + 1.44 = 4.0? 
    # Note: [S3] states |m|=2.0, likely referring to scaling factor or determinant sqrt.
    # We use the matrix values directly.
    m_a, m_b, m_c, m_d = 1.6, 1.2, -1.2, 1.6
    
    for id_z in range(Dz):
        for id_y in range(Dy):
            for id_x in range(Dx):
                # 1. Compute centered world position [S2, unofficial]
                p_x = Scale * (id_x + 0.5 - Dx / 2.0)
                p_y = Scale * (id_y + 0.5 - Dy / 2.0)
                p_z = Scale * (id_z + 0.5 - Dz / 2.0)
                
                # 2. Initialize value with vertical gradient [S2, unofficial]
                value = -p_y
                
                # 3. Add noise octaves
                # Variant A: Simplex with time offset and fixed scaling [S2, S4, unofficial]
                # This variant is used in [S2, unofficial] for SimplexNoise3D
                # Note: [S2, unofficial] adds two specific octaves. We generalize to N octaves.
                
                # Current coordinate for noise sampling
                curr_p_x = p_x
                curr_p_y = p_y
                curr_p_z = p_z
                
                # Time offset vector [S4, unofficial]
                time_offset_x = Time * 0.5 * 1.0
                time_offset_y = Time * 0.5 * -0.2
                time_offset_z = Time * 0.5 * -1.0
                
                # Initial scale and factor [S4, unofficial]
                current_scale = 0.5
                current_factor = 2.02
                
                for i in range(Octaves):
                    # Apply time offset [S4, unofficial]
                    sample_x = curr_p_x + time_offset_x
                    sample_y = curr_p_y + time_offset_y
                    sample_z = curr_p_z + time_offset_z
                    
                    # Sample noise. [S2, unofficial] uses SimplexNoise. [S3] uses generic noise.
                    # We assume noise returns [-1, 1]. If [0, 1], adjust accordingly.
                    n = noise_3d(sample_x, sample_y, sample_z)
                    
                    # Add to value. [S2, unofficial] multiplies noise by 0.5. 
                    # [S4, unofficial] uses current_scale. We follow [S4, unofficial] structure for FBM.
                    value += current_scale * n
                    
                    # Update coordinates for next octave
                    if UseRotation:
                        # Rotate and scale [S3]
                        new_x = m_a * curr_p_x + m_b * curr_p_y
                        new_y = m_c * curr_p_x + m_d * curr_p_y
                        curr_p_x = new_x * current_factor
                        curr_p_y = new_y * current_factor
                        curr_p_z = curr_p_z * current_factor # Z also scales
                    else:
                        # Standard scaling [S1, S4, unofficial]
                        curr_p_x *= current_factor
                        curr_p_y *= current_factor
                        curr_p_z *= current_factor
                        
                    # Update amplitude and lacunarity
                    current_scale *= Gain
                    if UseRotation:
                        # [S3] uses constant lacunarity via matrix scale, 
                        # but [S4, unofficial] varies lacunarity. We vary it if not using rotation.
                        if not UseRotation:
                            current_factor += 0.21 # Varying lacunarity [S4, unofficial]
                    else:
                        current_factor *= 2.0 # Standard lacunarity [S1, unofficial]

                # 4. Write to output buffer [S2, unofficial]
                idx = id_x + Dx * (id_y + Dy * id_z)
                voxels[idx] = value
                
    return voxels
```

**Edge Cases and Pitfalls:**
- **Axis-Aligned Artifacts:** Without inter-octave rotation or varying lacunarity, the noise exhibits visible grid-aligned patterns [S3]. Using a rotation matrix with determinant 2.0 (e.g., `mat2(1.6, 1.2, -1.2, 1.6)`) or varying lacunarity (2.02, 2.03) breaks self-similarity [S3].
- **Precision:** Hash functions using `sin` can be precision-sensitive on some platforms; sin-free hashes (using fract/dot mixing) are more stable [S3].
- **Boundary Conditions:** The centered position calculation $id + 0.5 - Dims/2$ ensures the field is centered at the origin, which is critical for consistent lighting and gradient alignment [S2, unofficial].

## Subtopics

### [FBM Combinations](3962-fbm-combinations.md)
FBM Combinations integrates Fractal Brownian Motion noise fields with rendering and geometric techniques to produce complex volumetric and surface effects [3962-fbm-combinations.md]. It is distinguished from [Terrain Heightmap Generation via Noise](4214-terrain-heightmap-generation-via-noise.md) by its focus on volumetric density and lighting integration rather than pure heightfield displacement, and from [Domain Warped FBM](4493-domain-warped-fbm.md) by its reliance on standard layered sampling rather than recursive coordinate distortion.

### [Terrain Heightmap Generation via Noise](4214-terrain-heightmap-generation-via-noise.md)
This method generates a terrain heightmap by converting 2D world coordinates into a single height value using scaled fractal noise, remapped to a specific vertical range. It differs from its parent [Noise Field Generator](1302-noise-field-generator.md) by operating directly on 2D world coordinates rather than sampling a 3D scalar field.

### [Domain Warped FBM](4493-domain-warped-fbm.md)
Domain Warped FBM is a procedural noise technique where the output of a noise function is fed back as a coordinate offset to distort the sampling domain, creating organic, fluid-like patterns. It differs from standard FBM by introducing non-linear spatial distortion that mimics turbulent flow, whereas standard FBM simply layers noise at different scales.

### Absolute Noise Sum
A fractal noise technique similar to fBm but uses the absolute value of the noise function at each octave, resulting in a range of 0.0 to 1.0 and creating 'cloudy' or 'rough' textures [S6, unofficial]. It sums `abs(inoise(p*freq))*amp` for each octave, with an initial amplitude of 1.0 (vs 0.5 for fBm) [S6, unofficial].

### Ocean Wave Octave
A specialized FBM variant for water surfaces. It computes wave components using `sea_octave`, which mixes `abs(sin(uv))` and `abs(cos(uv))` to create peaked crests, and sums bidirectional propagation shifted by `SEA_TIME` [S3].

### Bounded Perlin Noise Texture
A texture class that maps 3D points to RGB colors using Perlin noise, scaled and bounded within a specific region defined by `min_bound_` and `max_bound_` [S7, unofficial]. It stores color, scale, and a Perlin instance [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V \cdot O)$, where $V$ is the number of voxels and $O$ is the number of octaves. Each voxel requires $O$ noise evaluations [S5, unofficial].
- **Memory Complexity:** $O(V)$ for the output scalar field. Intermediate calculations are $O(1)$ per voxel [S2, unofficial].
- **Performance:** FBM is more computationally intensive than single-layer noise because it samples many waves, but produces more realistic results [S5, unofficial]. Performance can be optimized by reducing octave count for distant objects or using texture sampling instead of mathematical hash functions [S3].
- **Strengths:** Generates natural-looking, multi-scale detail suitable for clouds, terrain, and organic volumes [S3][S5, unofficial].
- **Limitations:** High computational cost for high-resolution volumes with many octaves. Axis-aligned artifacts if not properly randomized [S3].

## Background
The Noise Field Generator is based on Fractal Brownian Motion (FBM), a technique for generating pseudo-random fractal noise by summing multiple octaves of a base noise function [S3]. The specific implementation using layered Simplex noise with vertical gradients is used in GPU-based marching cubes and volumetric raymarching for real-time cloud and terrain generation [S2, S4, unofficial]. The use of rotation matrices to break axis-aligned artifacts was popularized in shader development communities [S3].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] keijiro/ComputeMarchingCubes, https://github.com/keijiro/ComputeMarchingCubes, fetched 2026-10-26 (unofficial)
[S3] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S4] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S5] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S6] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S7] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
