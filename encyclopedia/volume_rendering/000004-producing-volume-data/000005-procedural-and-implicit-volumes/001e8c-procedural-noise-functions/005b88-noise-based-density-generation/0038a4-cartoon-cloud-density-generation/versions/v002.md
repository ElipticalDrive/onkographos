# Cartoon Cloud Density Generation

Also called: Procedural Cloud Shape

## Overview
Cartoon Cloud Density Generation is a procedural method for generating stylized, cartoon-style cloud densities by combining independent multi-octave noise sets with geometric envelopes specific to cloud types [S1, unofficial]. It solves the problem of rendering volumetric clouds in a style consistent with voxel-based or stylized games while providing simplified lighting calculations [S1, unofficial]. It differs from its sibling [Blocky Volumetric Clouds](12303-blocky-volumetric-clouds.md) by relying on multi-octave noise functions for the base structure rather than a static 2D density map [S1, unofficial].

## How it works
**Inputs:**
- `x, y, z`: Normalized 3D coordinates in the cloud volume $[-1, 1]^3$ (or similar normalized space).
- `type_id`: Integer identifier for the cloud type (1-9, where 9 is Cumulus, others vary).
- `variation`: A seed or parameter controlling shape variation.
- `strength`: Global density multiplier (typically $0.0 \le strength \le 2.0$) [S1, unofficial].
- `disturbance`: Flow disturbance intensity (typically $0.0 \le disturbance \le 2.0$) [S1, unofficial].
- `detail`: Detail intensity (typically $0.0 \le detail \le 2.0$) [S1, unofficial].
- `finite_boundary`: Boolean indicating if the cloud is within a finite weather region.

**Outputs:**
- `density`: A scalar value in $[0.0, 1.0]$ representing the final cartoon cloud density.

**Data Structures:**
- `_CartoonNoiseSet`: Contains four independent noise lattice sets: `macro_lattices`, `body_lattices`, `edge_lattices`, `flow_lattices`. Each lattice is a tuple of vectors defining the noise field structure.
- `CloudShapeVariation`: Contains parameters for lobe clusters, wisp clusters, and footprint shapes.

**Helper Functions:**
- `_fractal_value_noise(lattices, scale)`: Computes a fractal Brownian motion (fBm) value using the provided lattices and scale.
- `_smoothstep(edge0, edge1, x)`: Standard smoothstep interpolation.
- `_broad_footprint(x, y, variation)`: Returns a 2D footprint mask for the cloud base.
- `_lobe_cluster(x, y, z, variation)`: Computes the lobe cluster density using the soft union formula.
- `_wisp_cluster(x, y, z, variation)`: Computes the wisp cluster density.
- `_cumulus_base(x, y, z, variation)`: Computes the base shape for cumulus clouds.
- `_soft_union(fields, threshold)`: Computes the soft union of multiple density fields.

**Steps:**

1. **Generate Noise Fields:**
   Compute four noise fields using fractal value noise with specific frequency multipliers:
   $$
   \begin{aligned}
   macro &= \text{fractal\_value\_noise}(\text{macro\_lattices}, \text{detail} \times 0.10) \\
   body &= \text{fractal\_value\_noise}(\text{body\_lattices}, \text{detail} \times 0.30) \\
   edge &= \text{fractal\_value\_noise}(\text{edge\_lattices}, \text{detail} \times 0.55) \\
   flow &= \text{fractal\_value\_noise}(\text{flow\_lattices}, \text{disturbance} \times 0.25)
   \end{aligned}
   $$

2. **Compute Base Shapes:**
   Derive three intermediate shapes from the noise fields:
   $$
   \begin{aligned}
   broad &= \text{smoothstep}\left(\frac{macro - 0.27}{0.52}\right) \\
   puffs &= \text{smoothstep}\left(\frac{0.56 \times macro + 0.44 \times body - 0.37}{0.38}\right) \\
   rim &= \text{smoothstep}\left(\frac{edge - 0.33}{0.48}\right)
   \end{aligned}
   $$

3. **Compute Fluffy Composite:**
   Combine the base shapes into a fluffy density:
   $$
   fluffy = \text{clip}(0.62 \times puffs + 0.23 \times broad + 0.15 \times rim, 0.0, 1.0)
   $$

4. **Compute Vertical Profiles:**
   Define three Gaussian vertical profiles based on normalized height $z$:
   $$
   \begin{aligned}
   thin &= \exp\left(-\left(\frac{z + 0.04}{0.13}\right)^2\right) \\
   middle &= \exp\left(-\left(\frac{z + 0.10}{0.30}\right)^2\right) \\
   low &= \exp\left(-\left(\frac{z + 0.28}{0.23}\right)^2\right)
   \end{aligned}
   $$

5. **Compute Footprint and Clusters:**
   - If `finite_boundary` is false, `footprint = 1.0`. Otherwise, `footprint = broad_footprint(x, y, variation)`.
   - Compute `cluster = lobe_cluster(x, y, z, variation)`.
   - Compute `wisps = wisp_cluster(x, y, z, variation)`.

6. **Combine Based on Cloud Type:**
   Calculate the base `density` based on `type_id`. Examples:
   - **Type 1 (Cirrus):**
     $$
     density = thin \times wisps \times \text{smoothstep}\left(\frac{0.72 \times body + 0.28 \times edge - 0.56}{0.30}\right)
     $$
   - **Type 9 (Cumulus):**
     $$
     \begin{aligned}
     base &= \text{cumulus\_base}(x, y, z, variation) \\
     surface &= \text{smoothstep}\left(\frac{0.58 \times fluffy + 0.42 \times broad - 0.20}{0.70}\right) \\
     density &= \text{soft\_union}\left(cluster \times (0.22 + 0.78 \times fluffy), 0.68 \times base \times (0.30 + 0.70 \times surface)\right)
     \end{aligned}
     $$
   - **Cumulonimbus (Else branch):**
     $$
     \begin{aligned}
     anvil &= footprint \times \exp\left(-\left(\frac{z - 0.48}{0.16}\right)^2\right) \\
     density &= \text{soft\_union}(cluster, 0.82 \times anvil) \times (0.40 + 0.60 \times fluffy)
     \end{aligned}
     $$

7. **Final Shaping and Quantization:**
   - Apply smoothstep to maintain continuous density and avoid concentric shell artifacts:
     $$
     shaped = \text{smoothstep}\left(\frac{density - 0.07}{0.62}\right)
     $$
   - Quantize into 5 discrete cartoon levels:
     $$
     stepped = \frac{\text{floor}(\text{clip}(shaped, 0.0, 1.0) \times 4.0 + 0.20)}{4.0}
     $$
   - Blend stepped and shaped densities, apply strength and rim/detail factor:
     $$
     return \text{clip}\left((0.12 \times stepped + 0.88 \times shaped) \times strength \times (0.94 + 0.06 \times detail \times rim), 0.0, 1.0\right)
     $$

**Formulas:**
- **Soft Union:**
  $$
  S(F_1, F_2, \dots) = \text{clip}\left(1 - \exp\left(-1.12 \times \text{clip}\left(\sum F_i, 0, 1\right)\right), 0, 1\right)
  $$
- **Lobe Cluster:**
  $$
  L = \text{clip}\left(1 - \exp\left(-1.18 \times \sum (w_i \times \exp(-\text{dist}_i^2))\right), 0, 1\right)
  $$

**Parameters:**
- `detail`: Controls noise frequency. Typical range $[0.0, 2.0]$.
- `disturbance`: Controls flow noise frequency. Typical range $[0.0, 2.0]$.
- `strength`: Global density multiplier. Typical range $[0.0, 2.0]$.

**Edge Cases and Pitfalls:**
- **Concentric Shells:** Avoid artifacts in volume rendering by keeping the texture body continuous density and applying quantization only in the final stage [S1, unofficial].
- **Finite Boundaries:** Ensure `footprint` is correctly computed for finite weather regions to avoid infinite cloud columns [S1, unofficial].

## Subtopics

### [Blocky Volumetric Clouds](12303-blocky-volumetric-clouds.md)
A technique using a static 2D density map for a blocky appearance, suitable for voxel-based games. Choose when simplicity and performance are prioritized over noise-based detail.

### [Procedural Volume Rendering with Noise Perturbation](1370-procedural-volume-rendering-with-noise-perturbation.md)
Uses noise perturbation for realistic clouds. Choose when realism is desired over stylized/cartoon aesthetics.

### [Weather & Cloud Parameterization](3617-weather-cloud-parameterization.md)
Defines cloud types and parameters. Use as a reference for `type_id` and vertical profiles.

### [Cloud Mesh Generation](4822-cloud-mesh-generation.md)
Generates mesh representations of clouds. Use when surface rendering is preferred over volume rendering.

### [Signed Distance Function (SDF) Generation and Processing](5342-signed-distance-function-sdf-generation-and-processing.md)
Generates SDFs for clouds. Use when distance-based rendering or collision detection is required.

### [TGorillaVolumetricNoise](7663-tgorillavolumetricnoise.md)
A specific noise implementation. Use if compatibility with the TGorilla engine is required.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel requires independent noise evaluations and arithmetic operations.
- **Memory Complexity:** $O(N)$ for the output density field. Noise lattices are small and constant size.
- **Performance:** Efficient on modern CPUs/GPUs due to simple arithmetic and lack of complex ray marching. Performance scales linearly with resolution.
- **Strengths:** Stylized appearance, computationally inexpensive, easy to tune via parameters.
- **Limitations:** Less realistic than noise-based methods, requires careful tuning for different cloud types.
- **When to Use:** Stylized games, voxel-based games, real-time applications where performance is critical.
- **When Not to Use:** Photorealistic rendering, scientific visualization.

## Background
Originated from the "Noise-Cloud" project by mingtu149411 [S1, unofficial]. The method is designed for procedural generation of cartoon-style clouds in game development, particularly for voxel-based games like Minecraft [S1, unofficial]. It combines noise functions with geometric envelopes to create distinct cloud types.

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
