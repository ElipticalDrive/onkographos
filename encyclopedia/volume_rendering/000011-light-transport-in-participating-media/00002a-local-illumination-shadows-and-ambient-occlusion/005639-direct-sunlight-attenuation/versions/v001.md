# Direct Sunlight Attenuation

Also called: Light Attenuation Simulation

## Overview
Direct Sunlight Attenuation simulates the reduction of solar intensity as light passes through a participating medium, typically using precomputed ray templates or eye-buffer transparency to determine voxel brightness [S1, unofficial][S2]. It differs from full volumetric path tracing by decoupling the direct light contribution into a separate pass that attenuates a distant light source based on integrated density, rather than computing scattering events along the view ray [S2].

## How it works
**Inputs:**
- A 3D volume grid of density values $\rho$ (or $\sigma_t$).
- A light direction vector $\mathbf{l}$ (typically the sun).
- A view direction vector $\mathbf{v}$ (for the distant light pass).
- Parameters: `DECAY` (absorption coefficient), `AMBIENT` (minimum light floor), `n` (grid dimension).

**Outputs:**
- Attenuated light intensity $l$ for each voxel or pixel.
- A final shaded color incorporating the direct sunlight term.

**Data Structures:**
- **Density Grid:** A 3D array of floats, indexed linearly as `offset = ((z * n) + y) * n + x` [S1, unofficial]. Note: Implementation may require index reversal depending on texture generation vs. simulation coordinate systems [S1, unofficial].
- **Intensity Buffer:** An 8-bit unsigned char array (`unsigned char* intensity`) storing accumulated light levels [S1, unofficial].
- **Ray Template:** A precomputed array of 3D offsets (`_rayTemplate[i][0..2]`) defining the path from a source point into the volume [S1, unofficial].

**Steps:**

1.  **Precompute Ray Templates:** Generate a set of ray directions based on the light vector $\mathbf{l}$. For each non-zero component of $\mathbf{l}$, define a path through the grid [S1, unofficial].

2.  **Light Ray Propagation (Per Voxel/Source):**
    For each starting position $(x, y, z)$ in the grid:
    a. Initialize light level $l = 200$ (or source intensity) [S1, unofficial].
    b. Calculate decay factor: $\text{decay} = \frac{1.0}{n \times \text{DECAY}}$ [S1, unofficial].
    c. Iterate along the ray template indices $i = 0, 1, \dots$:
       i. Compute current coordinates: $xx = x + \text{template}[i][0]$, $yy = y + \text{template}[i][1]$, $zz = z + \text{template}[i][2]$ [S1, unofficial].
       ii. Check bounds: if $xx, yy, zz$ are outside $[0, n)$, terminate [S1, unofficial].
       iii. Calculate linear offset: $\text{offset} = ((zz \times n) + yy) \times n + xx$ [S1, unofficial].
       iv. Retrieve density: $d = \text{dens}[\text{offset}] \times 255.0$ [S1, unofficial].
       v. Update light level:
           $$ l_{\text{new}} = \max(\text{AMBIENT}, l_{\text{old}} - d \times \text{decay}) $$
           [S1, unofficial]
       vi. Update intensity buffer:
           $$ \text{intensity}[\text{offset}] = \max(\text{AMBIENT}, \text{intensity}[\text{offset}]) $$
           If $\text{intensity}[\text{offset}] > 0$:
           $$ \text{intensity}[\text{offset}] = \left\lfloor \frac{\text{intensity}[\text{offset}] + l_{\text{new}}}{2} \right\rfloor $$
           Else:
           $$ \text{intensity}[\text{offset}] = l_{\text{new}} $$
           [S1, unofficial]
       vii. If $l_{\text{new}} \le \text{AMBIENT}$, terminate loop [S1, unofficial].

3.  **Distant Light Pass (View-Based):**
    a. Store total volume transparency in the alpha channel of the eye buffer [S2].
    b. Calculate dot product: $dot = \mathbf{v} \cdot \mathbf{l}$ [S2].
    c. Check if view vector is within the sun's disk (e.g., $dot > \cos(0.5^\circ)$) [S2].
    d. If true, add the attenuated sunlight term to the pixel color, modulated by the stored transparency and atmospheric attenuation [S2].

**Parameters:**
- `DECAY`: 0.06f (light absorption coefficient) [S1, unofficial].
- `AMBIENT`: 100 (minimum light level floor) [S1, unofficial].
- Sun disk threshold: 0.5 degrees [S2].

**Edge Cases and Pitfalls:**
- **Intensity Averaging:** The accumulation step `(intensity[offset] + l)*0.5f` halves the contribution of previous rays, potentially underestimating brightness in dense regions [S1, unofficial].
- **Coordinate Alignment:** Density lookup offsets must match the texture generation coordinate system; mismatches cause visual artifacts [S1, unofficial].
- **Precision:** Using 8-bit integers for intensity limits dynamic range; consider floating-point buffers for high-dynamic-range scenes [G].

## Operating characteristics
- **Time Complexity:** $O(N \cdot R)$ where $N$ is the number of voxels and $R$ is the average ray length through the volume [S1, unofficial].
- **Memory Complexity:** $O(N)$ for density and intensity buffers [S1, unofficial].
- **Strengths:** Computationally efficient for static or slowly changing volumes; suitable for real-time applications on older hardware (e.g., GeForce FX) [S2].
- **Limitations:** Approximates scattering as simple attenuation; does not account for multiple scattering or complex shadowing within the volume [S1, unofficial][S2]. The averaging step can lead to incorrect brightness accumulation [S1, unofficial].

## Background
The method originates from efficient rendering techniques for atmospheric phenomena, where direct sunlight is treated as a distant light source attenuated by the volume's total transparency [S2]. It was implemented on early programmable graphics hardware using fragment programs [S2]. The ray-template approach for voxel-based attenuation is documented in unofficial smoke simulation repositories [S1, unofficial].

## Sources
[S1] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
[S2] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
