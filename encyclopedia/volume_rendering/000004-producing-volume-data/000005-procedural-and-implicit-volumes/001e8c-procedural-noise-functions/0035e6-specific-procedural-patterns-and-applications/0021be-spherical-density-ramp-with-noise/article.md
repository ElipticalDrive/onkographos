# Spherical Density Ramp with Noise

Also called: ramp-confined turbulence gas, steam_slab1 confined gas modeling

## Overview
Spherical Density Ramp with Noise is a procedural volume density function that confines a gas to a slab region using a spherical distance ramp and an offset table, simulating steam rising from a source [S1, unofficial]. It differs from siblings like [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) and [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md) by applying a deterministic spherical falloff combined with a discrete random offset table to modulate base turbulence density, avoiding complex flow advection [S1, unofficial].

## How it works
The method computes the density of a gas at a query point by combining a base turbulence density with a spherical confinement ramp and a random offset table [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Point in solid texture space $P_{solid}$, point in world space $P_{world}$, base density $\rho$, parameter array $\mathbf{p}$, volume structure $V$.
*   **Output:** Updated density $\rho$.
*   **Volume Structure $V$:** Contains `center` (3D vector) and `inv_rad_sq` (vector, where `.y` is $1/r^2$) [S1, unofficial].
*   **Parameters $\mathbf{p}$:** $\mathbf{p}[1]$ scales the turbulence input, $\mathbf{p}[2]$ is the exponent for the power function, $\mathbf{p}[3]$ is the randomness multiplier for the offset table [S1, unofficial].

**Data Structures:**
1.  **`pow_table`**: Array of size `POW_TABLE_SIZE`. Precomputed values mapping turbulence intensity to density [S1, unofficial].
2.  **`ramp`**: Array of size `RAMP_SIZE`. Precomputed cosine falloff values [S1, unofficial].
3.  **`offset`**: Array of size `OFFSET_SIZE`. Precomputed pseudo-random values used to add noise to the ramp-off [S1, unofficial].

**Precomputation:**
The tables `pow_table`, `ramp`, and `offset` are computed once per image [S1, unofficial].
*   `pow_table[i]` is calculated as:
    $$ \text{pow\_table}[i] = \left( \frac{i}{\text{POW\_TABLE\_SIZE}-1} \cdot \mathbf{p}[1] \cdot 2.0 \right)^{\mathbf{p}[2]} $$
    for $i$ from `POW_TABLE_SIZE-1` down to 0 [S1, unofficial].
*   `ramp[i]` is calculated as:
    $$ \text{ramp}[i] = \frac{\cos\left(\frac{i}{\text{RAMP\_SIZE}-1} \pi\right) + 1.0}{2.0} $$
    for $i$ from 0 to `RAMP_SIZE-1` [S1, unofficial].

**Algorithm Steps:**
1.  **Base Turbulence:** Evaluate $T = \text{fast\_turbulence}(P_{solid})$. Map to base density using the `pow_table`:
    $$ \rho = \text{pow\_table}\left[ \left\lfloor T \cdot 0.5 \cdot (\text{POW\_TABLE\_SIZE}-1) \right\rfloor \right] $$
    [S1, unofficial].
2.  **Spherical Confinement:**
    a. Calculate the squared distance from the point to the volume center:
       $$ \text{diff} = P_{world} - V.\text{center} $$
       $$ \text{dist\_sq} = \text{dot}(\text{diff}, \text{diff}) $$
       [S1, unofficial].
    b. Compute the initial density scaling factor:
       $$ \text{density\_max} = \text{dist\_sq} \cdot V.\text{inv\_rad\_sq}.y $$
       [S1, unofficial].
    c. Add noise to the scaling factor using an offset table indexed by the sum of the point's coordinates:
       $$ \text{indx} = \left\lfloor (P_{world}.x + P_{world}.y + P_{world}.z) \cdot 100 \right\rfloor \ \& \ (\text{OFFSET\_SIZE} - 1) $$
       $$ \text{density\_max} += \mathbf{p}[3] \cdot \text{offset}[\text{indx}] $$
       [S1, unofficial].
    d. If $\text{density\_max} \ge 0.25$, apply a cosine falloff ramp:
       $$ i = \left\lfloor (\text{density\_max} - 0.25) \cdot \frac{4}{3} \cdot \text{RAMP\_SIZE} \right\rfloor $$
       Clamp $i$ to $[0, \text{RAMP\_SIZE}-1]$.
       $$ \text{density\_max} = \text{ramp}[i] $$
       [S1, unofficial].
    e. Multiply the base density by the ramped factor:
       $$ \rho *= \text{density\_max} $$
       [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fast_turbulence(p):
    # Standard 3D turbulence function from parent context [G]
    # Returns a float typically in [0, 1]
    return 0.5

def spherical_density_ramp_with_noise(p_world, p_solid, density, params, vol, offset_table, ramp_table, pow_table):
    """
    Computes the density of a gas confined to a slab with spherical ramp and noise offset.
    
    Args:
        p_world: (float, float, float) World space position.
        p_solid: (float, float, float) Solid texture space position (for turbulence).
        density: float Current base density.
        params: list[float] Parameters [_, p1, p2, p3] (1-based indexing matches source).
            params[1]: Scales turbulence input.
            params[2]: Exponent for power function.
            params[3]: Randomness multiplier for fall off.
        vol: dict Volume structure with 'center' (vec3) and 'inv_rad_sq' (vec3).
        offset_table: list[float] Precomputed random offsets.
        ramp_table: list[float] Precomputed cosine ramp values.
        pow_table: list[float] Precomputed power table values.
        
    Returns:
        float: Updated density.
    """
    # 1. Base Turbulence
    turb = fast_turbulence(p_solid)
    turb_index = int(turb * 0.5 * (len(pow_table) - 1))
    turb_index = max(0, min(turb_index, len(pow_table) - 1))
    density = pow_table[turb_index]
    
    # 2. Spherical Confinement
    diff_x = p_world[0] - vol['center'][0]
    diff_y = p_world[1] - vol['center'][1]
    diff_z = p_world[2] - vol['center'][2]
    
    dist_sq = diff_x**2 + diff_y**2 + diff_z**2
    
    density_max = dist_sq * vol['inv_rad_sq'][1]
    
    # Index into offset table
    sum_coords = p_world[0] + p_world[1] + p_world[2]
    indx = int(sum_coords * 100) & (len(offset_table) - 1)
    
    density_max += params[3] * offset_table[indx]
    
    if density_max >= 0.25:
        i = int((density_max - 0.25) * (4.0 / 3.0) * len(ramp_table))
        # Clamp i to [0, RAMP_SIZE-1]
        i = max(0, min(i, len(ramp_table) - 1))
        density_max = ramp_table[i]
        
    density *= density_max
    
    return density
```

## Subtopics

### [Vertical Exponential Density Ramp](2688-vertical-exponential-density-ramp.md)
The Vertical Exponential Density Ramp is a procedural density modulation technique used to simulate natural phenomena like rising steam or smoke by attenuating density as a function of vertical height. It builds on [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md) by adding a vertical falloff, solving the problem of volumetric objects appearing as solid, unnatural columns by ensuring density decays to zero above a certain altitude. It differs from generic noise textures by applying a specific mathematical ramp based on world-space Y coordinates rather than procedural pattern evaluation.

### [NoiseTexture value evaluation](3448-noisetexture-value-evaluation.md)
NoiseTexture evaluates a volumetric color and density field by combining radial/vertical fading with double-warping via Fractal Brownian Motion (FBM) and sinusoidal noise modulation. It differs from simple procedural noise by applying spatial warping to the noise domain and modulating the result with a sine wave, creating structured, organic patterns suitable for volumetric effects like smoke or clouds.

## Operating characteristics
Time and memory complexity are not given in the sources. Measured performance, scaling behavior, strengths, limitations, and usage recommendations are not given in the sources.

## Background
The method originates from David Ebert's work on procedural modeling of gases, fluids, and textures [S1, unofficial]. It was developed to address the limitations of basic turbulence functions, which produced gas that was not confined and did not decrease in density as it rose, resulting in unrealistic visuals [S1, unofficial]. The technique combines spherical distance ramps with noise offsets to create a more convincing steam effect [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S2] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
