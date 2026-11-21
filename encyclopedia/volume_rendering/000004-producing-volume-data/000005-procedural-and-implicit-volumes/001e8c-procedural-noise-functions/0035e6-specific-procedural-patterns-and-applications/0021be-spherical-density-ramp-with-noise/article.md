# Spherical Density Ramp with Noise

Also called: ramp-confined turbulence gas, steam_slab1 confined gas modeling

## Overview
Spherical Density Ramp with Noise is a procedural volume density function that confines a gas (such as steam or smoke) to a spherical slab region using a spherical distance ramp and an offset table, simulating steam rising from a source [S1, unofficial]. It differs from its siblings by combining a base turbulence pattern with a specific radial falloff mechanism that uses a precomputed cosine ramp and spatially varying noise offsets to break symmetry, whereas [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) relies on sinusoidal modulation and [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md) uses iterative complex number transformations.

## How it works
**Inputs:**
- `pnt`: 3D point in solid texture space (used for turbulence evaluation).
- `pnt_world`: 3D point in world space (used for distance calculation).
- `density`: Initial density value (output parameter, modified in place).
- `parms`: Array of floating-point parameters.
  - `parms[1]`: Scaling factor for the power function.
  - `parms[2]`: Exponent for the power function.
  - `parms[3]`: Multiplier for the random offset table (noise amplitude).
- `vol`: Volume structure containing shape information.
  - `vol.shape.center`: 3D center point of the volume slab.
  - `vol.shape.inv_rad_sq.y`: Inverse squared radius of the slab ($1/r^2$).

**Outputs:**
- `density`: The final computed density value.

**Data Structures:**
1.  **`pow_table`**: Array of size `POW_TABLE_SIZE`. Precomputed values mapping turbulence intensity to base density.
2.  **`ramp`**: Array of size `RAMP_SIZE`. Precomputed cosine falloff values.
3.  **`offset`**: Array of size `OFFSET_SIZE`. Precomputed random numbers used to add spatial variation to the density falloff.

**Initialization (Precomputed once per image):**
The tables are initialized using a static flag `calcd`.

1.  **Pow Table**: For $i$ from `POW_TABLE_SIZE - 1` down to $0$:
    $$ \text{pow\_table}[i] = \left( \frac{i}{\text{POW\_TABLE\_SIZE} - 1} \times \text{parms}[1] \times 2.0 \right)^{\text{parms}[2]} $$
2.  **Ramp Table**: For $i$ from $0$ to `RAMP_SIZE - 1`:
    $$ \text{dist} = \frac{i}{\text{RAMP\_SIZE} - 1.0} $$
    $$ \text{ramp}[i] = \frac{\cos(\text{dist} \times \pi) + 1.0}{2.0} $$

**Algorithm Steps:**

1.  **Evaluate Base Turbulence**:
    Calculate the turbulence value at the texture point:
    $$ \text{turb} = \text{fast\_turbulence}(\text{pnt}) $$
    Map this to a base density using the power table. The index is calculated as:
    $$ \text{idx} = \left\lfloor \text{turb} \times 0.5 \times (\text{POW\_TABLE\_SIZE} - 1) \right\rfloor $$
    $$ \text{density} = \text{pow\_table}[\text{idx}] $$
    *Note: `fast_turbulence` is an optimized version of the standard turbulence function, typically involving layered noise [S1, unofficial].*

2.  **Calculate Spherical Distance Factor**:
    Compute the vector from the volume center to the world point:
    $$ \text{diff} = \text{pnt\_world} - \text{vol.shape.center} $$
    Compute the squared distance:
    $$ \text{dist\_sq} = \text{diff} \cdot \text{diff} $$
    Scale by the inverse squared radius to get a normalized distance factor:
    $$ \text{density\_max} = \text{dist\_sq} \times \text{vol.shape.inv\_rad\_sq.y} $$

3.  **Add Spatial Noise Offset**:
    Calculate an index into the offset table based on the sum of the world coordinates:
    $$ \text{indx} = \left( \lfloor (\text{pnt\_world}.x + \text{pnt\_world}.y + \text{pnt\_world}.z) \times 100 \rfloor \right) \ \& \ (\text{OFFSET\_SIZE} - 1) $$
    Add the noise offset scaled by `parms[3]`:
    $$ \text{density\_max} += \text{parms}[3] \times \text{offset}[\text{indx}] $$

4.  **Apply Cosine Ramp Falloff**:
    If `density_max` exceeds the threshold $0.25$ (25% from center):
    $$ i = \lfloor (\text{density\_max} - 0.25) \times \frac{4}{3} \times \text{RAMP\_SIZE} \rfloor $$
    Clamp $i$ to the valid range:
    $$ i = \min(i, \text{RAMP\_SIZE} - 1) $$
    Retrieve the ramp value:
    $$ \text{ramp\_val} = \text{ramp}[i] $$
    Multiply the density by this factor:
    $$ \text{density} *= \text{ramp\_val} $$
    If `density_max` $\le 0.25$, the density remains unchanged by the ramp (effectively multiplied by 1.0, as the ramp starts at 1.0 for distance 0).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_density_ramp_with_noise(pnt, pnt_world, density, parms, vol):
    # 1. Initialize tables if not done
    if not pow_table_initialized:
        for i in range(POW_TABLE_SIZE - 1, -1, -1):
            t = (i / (POW_TABLE_SIZE - 1)) * parms[1] * 2.0
            pow_table[i] = t ** parms[2]
        
        for i in range(RAMP_SIZE):
            dist = i / (RAMP_SIZE - 1.0)
            ramp[i] = (cos(dist * PI) + 1.0) / 2.0
        
        pow_table_initialized = True

    # 2. Evaluate base turbulence
    turb = fast_turbulence(pnt)
    idx = int(turb * 0.5 * (POW_TABLE_SIZE - 1))
    # Clamp idx to table bounds if necessary
    idx = max(0, min(idx, POW_TABLE_SIZE - 1))
    density = pow_table[idx]

    # 3. Calculate spherical distance factor
    diff_x = pnt_world.x - vol.shape.center.x
    diff_y = pnt_world.y - vol.shape.center.y
    diff_z = pnt_world.z - vol.shape.center.z
    dist_sq = diff_x * diff_x + diff_y * diff_y + diff_z * diff_z
    density_max = dist_sq * vol.shape.inv_rad_sq.y

    # 4. Add spatial noise offset
    sum_coords = pnt_world.x + pnt_world.y + pnt_world.z
    indx = int(sum_coords * 100) & (OFFSET_SIZE - 1)
    density_max += parms[3] * offset[indx]

    # 5. Apply cosine ramp falloff
    if density_max >= 0.25:
        i = int((density_max - 0.25) * (4.0 / 3.0) * RAMP_SIZE)
        if i >= RAMP_SIZE:
            i = RAMP_SIZE - 1
        density *= ramp[i]

    return density
```

**Edge Cases and Pitfalls:**
-   **Precision**: The index calculation for `indx` uses integer truncation of a scaled float sum. Small changes in position near the boundary of an integer index may cause discontinuities if the offset table is not smooth, though the offset table itself is static random data.
-   **Threshold**: The ramp only activates for `density_max >= 0.25`. Points within 50% of the radius (in squared distance terms) are unaffected by the ramp.
-   **Table Sizes**: `POW_TABLE_SIZE`, `RAMP_SIZE`, and `OFFSET_SIZE` must be defined constants. Typical values are powers of 2 for efficient bitwise masking (e.g., `OFFSET_SIZE = 256`).

## Subtopics

### [Vertical Exponential Density Ramp](2688-vertical-exponential-density-ramp.md)
The Vertical Exponential Density Ramp is a procedural density modulation technique used to simulate natural phenomena like rising steam or smoke by attenuating density as a function of vertical height. It builds on [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md) by adding a vertical falloff, solving the problem of volumetric objects appearing as solid, unnatural columns by ensuring density decays to zero above a certain altitude. It differs from generic noise textures by applying a specific mathematical ramp based on world-space Y coordinates rather than procedural pattern evaluation.

### [NoiseTexture value evaluation](3448-noisetexture-value-evaluation.md)
NoiseTexture evaluates a volumetric color and density field by combining radial/vertical fading with double-warping via Fractal Brownian Motion (FBM) and sinusoidal noise modulation. It differs from simple procedural noise by applying spatial warping to the noise domain and modulating the result with a sine wave, creating structured, organic patterns suitable for volumetric effects like smoke or clouds.

### Radial Exponential Density Ramp
If `ZoneIndex` is `(0, 0, -1)`, calculate radial falloff: $r = \sqrt{P.X^2 + P.Y^2 + P.Z^2}$. Calculate $t = 1 - \exp(-r^2 / (\text{MaxR} \times 100))$ where $\text{MaxR} = 1000$. Return $\text{Density} \times t$. If `IsCaveLayerZone(ZoneIndex.Z)` is true, the function currently returns `Density` (the cave layer call is commented out) [S2, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(1)$ per voxel, assuming `fast_turbulence` is $O(1)$ or bounded. The table lookups and arithmetic operations are constant time.
-   **Memory Complexity**: $O(N)$ for precomputed tables, where $N$ is the sum of `POW_TABLE_SIZE`, `RAMP_SIZE`, and `OFFSET_SIZE`. These are static and allocated once per image/render pass.
-   **Strengths**: Efficiently confines gas to a specific volume without complex boundary checks. The noise offset breaks radial symmetry, making the gas look more natural than a pure cosine falloff.
-   **Limitations**: The confinement is spherical/slab-based, not suitable for arbitrary shapes. The `fast_turbulence` function's performance depends on its implementation (e.g., number of noise octaves).
-   **When to use**: Simulating steam, smoke, or gas clouds originating from a central point or slab.
-   **When not to use**: When the volume shape is not roughly spherical or slab-like, or when complex internal structure requires more than a simple radial falloff.

## Background
The method originates from David Ebert's work on procedural modeling of gases and fluids, specifically described in "Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures" [S1, unofficial]. It was designed to correct the unnatural appearance of basic turbulence-based gases by confining them to a slab and adding a rising/falloff characteristic. The "steam_slab1" name reflects its specific application to simulating steam rising from a cup.

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S2] bw2012/UE4VoxelTerrain, https://github.com/bw2012/UE4VoxelTerrain, fetched 2026-10-26 (unofficial)
