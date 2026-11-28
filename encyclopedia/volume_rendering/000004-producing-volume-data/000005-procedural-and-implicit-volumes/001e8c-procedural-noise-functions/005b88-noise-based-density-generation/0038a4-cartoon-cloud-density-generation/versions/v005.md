# Cartoon Cloud Density Generation

Also called: Procedural Cloud Shape

## Overview
Cartoon Cloud Density Generation is a procedural method for generating stylized, cartoon-style cloud densities by combining independent noise sets for macro structure, body puffs, edge details, and flow, combined with geometric envelopes for specific cloud types [S1, unofficial]. It differs from its sibling [Perlin-Worley Noise for Cloud Density](7139-perlin-worley-noise-for-cloud-density.md) by relying on a multi-layered noise composition and explicit geometric shapes (lobes, clusters) rather than a single unified noise field, allowing for distinct "blocky" or stylized appearances consistent with voxel-based games [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D query point $\mathbf{P} = (x, y, z)$, cloud type identifier `type_id` (integer 1-9), resolution parameters, and strength/detail/disturbance multipliers.
*   **Outputs:** A scalar density value in $[0.0, 1.0]$.

**Data Structures:**
*   `_CartoonNoiseSet`: Contains four independent sets of lattice gradient vectors: `macro_lattices`, `body_lattices`, `edge_lattices`, and `flow_lattices`. These are pre-generated and stored for reuse.
*   `VolumeResult`: Contains the resulting `density` and `disturbance` arrays (8-bit linear data).

**Steps:**

1.  **Generate Noise Fields:**
    Compute four fractal value noise fields using the independent lattice sets. The frequencies are scaled by multipliers:
    *   `macro`: Frequency multiplier $0.10$.
    *   `body`: Frequency multiplier $0.30$.
    *   `edge`: Frequency multiplier $0.55$.
    *   `flow`: Frequency multiplier $0.25 \times \text{disturbance}$.
    
    $$ \text{macro} = \text{FractalValueNoise}(\mathbf{P}, \text{macro\_lattices}, 0.10) $$
    $$ \text{body} = \text{FractalValueNoise}(\mathbf{P}, \text{body\_lattices}, 0.30) $$
    $$ \text{edge} = \text{FractalValueNoise}(\mathbf{P}, \text{edge\_lattices}, 0.55) $$
    $$ \text{flow} = \text{FractalValueNoise}(\mathbf{P}, \text{flow\_lattices}, 0.25 \times \text{disturbance}) $$

2.  **Compute Base Shapes:**
    Apply `smoothstep` to normalize and threshold the noise fields. `smoothstep(t)` is defined as $3t^2 - 2t^3$ for $t \in [0,1]$, clamped to $[0,1]$.
    *   Broad structure:
        $$ \text{broad} = \text{smoothstep}\left(\frac{\text{macro} - 0.27}{0.52}\right) $$
    *   Puffs (body + macro mix):
        $$ \text{puffs} = \text{smoothstep}\left(\frac{0.56 \times \text{macro} + 0.44 \times \text{body} - 0.37}{0.38}\right) $$
    *   Rim/Edge detail:
        $$ \text{rim} = \text{smoothstep}\left(\frac{\text{edge} - 0.33}{0.48}\right) $$

3.  **Compute Fluffy Composite:**
    Combine the base shapes into a unified "fluffy" density field.
    $$ \text{fluffy} = \text{clip}(0.62 \times \text{puffs} + 0.23 \times \text{broad} + 0.15 \times \text{rim}, 0.0, 1.0) $$

4.  **Compute Vertical Profiles:**
    Define Gaussian vertical profiles based on the $z$ coordinate. These determine the thickness and height of different cloud layers.
    *   Thin (high altitude):
        $$ \text{thin} = \exp\left(-\left(\frac{z + 0.04}{0.13}\right)^2\right) $$
    *   Middle:
        $$ \text{middle} = \exp\left(-\left(\frac{z + 0.10}{0.30}\right)^2\right) $$
    *   Low:
        $$ \text{low} = \exp\left(-\left(\frac{z + 0.28}{0.23}\right)^2\right) $$

5.  **Compute Horizontal Footprint and Clusters:**
    *   **Footprint:** A 2D shape mask based on $x, y$. If the cloud has a finite boundary, use `_broad_footprint(x, y, variation)`; otherwise, use ones (infinite).
    *   **Clusters:** Generate 3D lobe clusters using `_lobe_cluster(x, y, z, variation)` and wisp clusters using `_wisp_cluster(x, y, z, variation)`. These use the Soft Union function (see Formulas).

6.  **Combine Based on Cloud Type (`type_id`):**
    The density is assembled differently for each of the 9 cloud types. Examples:
    *   **Type 1 (Cirrus):**
        $$ \text{density} = \text{thin} \times \text{wisps} \times \text{smoothstep}\left(\frac{0.72 \times \text{body} + 0.28 \times \text{edge} - 0.56}{0.30}\right) $$
    *   **Type 9 (Cumulus):**
        $$ \text{base} = \text{CumulusBaseShape}(x, y, z) $$
        $$ \text{surface} = \text{smoothstep}\left(\frac{0.58 \times \text{fluffy} + 0.42 \times \text{broad} - 0.20}{0.70}\right) $$
        $$ \text{density} = \text{SoftUnion}(\text{cluster} \times (0.22 + 0.78 \times \text{fluffy}), 0.68 \times \text{base} \times (0.30 + 0.70 \times \text{surface})) $$
    *   **Cumulonimbus (Else branch):**
        Calculate anvil shape:
        $$ \text{anvil} = \text{footprint} \times \exp\left(-\left(\frac{z - 0.48}{0.16}\right)^2\right) $$
        Combine cluster and anvil:
        $$ \text{density} = \text{SoftUnion}(\text{cluster}, 0.82 \times \text{anvil}) \times (0.40 + 0.60 \times \text{fluffy}) $$

7.  **Final Shaping and Quantization:**
    *   Apply smoothstep to maintain continuous density and avoid concentric shell artifacts:
        $$ \text{shaped} = \text{smoothstep}\left(\frac{\text{density} - 0.07}{0.62}\right) $$
    *   Quantize into 5 discrete cartoon levels (mapped to 0-1 range):
        $$ \text{stepped} = \frac{\text{floor}(\text{clip}(\text{shaped}, 0.0, 1.0) \times 4.0 + 0.20)}{4.0} $$
    *   Blend stepped and shaped densities, apply strength and rim/detail modulation:
        $$ \text{result} = \text{clip}\left((0.12 \times \text{stepped} + 0.88 \times \text{shaped}) \times \text{strength} \times (0.94 + 0.06 \times \text{detail} \times \text{rim}), 0.0, 1.0\right) $$

**Formulas:**

*   **Soft Union:** Used to merge multiple density fields smoothly.
    $$ S(F_1, F_2, \dots) = \text{clip}\left(1 - \exp\left(-1.12 \times \text{clip}\left(\sum F_i, 0, 1\right)\right), 0, 1\right) $$
    Implementation: `np.clip(1.0 - np.exp(-total * 1.12), 0.0, 1.0)` where `total` is the sum of clipped fields.

*   **Lobe Cluster:** A specific instance of soft union for geometric lobes.
    $$ L = \text{clip}\left(1 - \exp\left(-1.18 \times \sum (w_i \times \exp(-\text{dist}_i^2))\right), 0, 1\right) $$
    Implementation: `np.clip(1.0 - np.exp(-field_sum * 1.18), 0.0, 1.0)`.

**Parameters:**
*   **Detail Multiplier:** 0.10 (macro), 0.30 (body), 0.55 (edge), 0.25 * disturbance (flow).
*   **Strength:** Typically in range $[0.0, 2.0]$.
*   **Disturbance:** Typically in range $[0.0, 2.0]$.

**Edge Cases and Pitfalls:**
*   **Concentric Shells:** To avoid artifacts in volume rendering, the texture body must maintain continuous density. Quantization should only be applied in the final shading stage if needed [S1, unofficial].
*   **Boundary Conditions:** For finite cloud regions, the footprint function must handle boundaries to prevent infinite column artifacts along the Z-axis [S1, unofficial].

## Subtopics

### [Blocky Volumetric Clouds](12303-blocky-volumetric-clouds.md)
Uses a static 2D density map for a blocky appearance, differing from Cartoon Cloud Density Generation which uses multi-octave noise and geometric shapes.

## Operating characteristics
*   **Time Complexity:** Depends on the number of noise octaves and the complexity of the geometric clusters. Each query point requires evaluating multiple noise fields and geometric functions.
*   **Memory Complexity:** Requires storing the pre-generated lattice gradients for the four noise sets and the cloud type parameters.
*   **Strengths:** Produces distinct, stylized cloud shapes suitable for non-photorealistic rendering. The separation of macro, body, and edge noise allows for fine-tuned artistic control.
*   **Limitations:** More complex to implement than single-noise methods. The geometric clusters require careful tuning of parameters for different cloud types.
*   **When to Use:** Stylized games, voxel-based environments, or any application requiring non-realistic, cartoon-like clouds.

## Background
Originated from procedural generation techniques in game development, specifically for creating consistent cloud shapes in voxel-based games like Minecraft. The method is documented in unofficial sources related to noise cloud generation libraries [S1, unofficial].

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
