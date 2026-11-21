# Absorption Term Restriction

Also called: Absorption-only restriction, Non-emissive absorption disabling

## Overview
Absorption Term Restriction is an optical model simplification that restricts the volume density model to the absorption term, effectively treating the medium as non-emissive and removing absorption collisions from the path simulation. It solves the problem of computational efficiency in participating media by reducing variance and the magnitude of outliers, particularly when the bounding function $\bar{\mu}$ does not tightly bound the total extinction coefficient $\mu_t(x)$. Unlike the full Emission-Absorption Model, it assumes no light emission, and unlike the Particle Model, it relies on a specific texture-based or probabilistic restructuring of the transport equation.

## How it works
**Inputs and Outputs:**
- Input: Volume density field (absorption coefficient $\mu_a(x)$), bounding function $\bar{\mu}$, and scene geometry.
- Output: Radiance values for pixels, computed via path tracing or ray marching with modified termination probabilities.

**Data Structures:**
- **3D Texture Map:** Contains the integral of the absorption coefficient along the viewing ray segment. This is used for hardware-accelerated lookup [S1].
- **Probability Tables:** Normalization constants $c$ and probabilities $P_s(x_j)$ (scattering) and $P_n(x_j)$ (null collision) are recomputed without the absorption term [S2].

**Steps:**
1. **Initialize:** Define the total extinction coefficient $\mu_t(x) = \mu_a(x) + \mu_s(x)$.
2. **Restrict Model:** Set the absorption probability $P_a(x_j) = 0$ for all points $x_j$ [S2]. This removes absorption collisions from the simulation.
3. **Recompute Normalization:** Remove the absorption term from the normalization constant $c$. The new constant $c'$ is derived from the remaining scattering and null-collision terms [S2].
4. **Recompute Probabilities:** Recalculate the scattering probability $P_s(x_j)$ and null-collision probability $P_n(x_j)$ using the updated normalization constant. Since the denominator in the probability definitions increases (as the absorption term is removed from the total weight), the throughput upon scattering or null-collision increases [S2].
5. **Simulate Paths:** Trace rays through the volume. At each step, sample a collision type based on the recomputed $P_s$ and $P_n$. If a scattering event occurs, update the radiance contribution. If a null collision occurs, continue. Termination occurs only via scattering or null-collision logic, not absorption [S2].
6. **Accumulate:** Accumulate the radiance contributions. Since absorption is not simulated as a collision, the attenuation is handled implicitly through the probability weights or pre-integrated in the texture map [S1].

**Formulas:**
The probability of a scattering event at position $x_j$ is:
$$ P_s(x_j) = \frac{\mu_s(x_j)}{c'} $$
The probability of a null collision is:
$$ P_n(x_j) = \frac{\bar{\mu} - \mu_t(x_j)}{c'} $$
Where $c'$ is the new normalization constant excluding absorption [S2].

**Parameters:**
- $\bar{\mu}$: The bounding function for the total extinction coefficient. It must bound $\mu_t(x)$ across all wavelengths if spectral tracking is used [S2].

**Edge Cases and Pitfalls:**
- **Increased Computational Cost:** Disabling absorption increases the computational cost because paths must be simulated longer to terminate via scattering or null collisions [S2].
- **Variance Reduction:** This technique reduces variance and the magnitude of outliers if $\bar{\mu}$ does not bound $\mu_t(x)$ tightly [S2].
- **Non-Emissive Assumption:** The method is only valid for non-emissive media. If emission is present, this restriction will produce incorrect results.

## Subtopics

### Reduced Termination Rates for Non-Emissive Media
This variant specifically targets non-emissive media by setting $P_a(x_j) = 0$ and redistributing the absorption probability among scattering and null-collision types. It recomputes $P_s(x_j)$ and $P_n(x_j)$ to increase throughput upon these events. While it increases the path length required for termination, it reduces variance and outlier magnitude if the bounding function $\bar{\mu}$ is loose [S2].

## Operating characteristics
- **Time Complexity:** Increases compared to standard absorption-inclusive models because paths are longer. The cost is proportional to the increased path length required for termination via scattering or null collisions [S2].
- **Memory Complexity:** Requires a 3D texture map containing the integral of the absorption coefficient for hardware-based implementations [S1].
- **Strengths:** Reduces variance and outlier magnitude, especially when the bounding function $\bar{\mu}$ is not tight. Efficient for non-emissive media.
- **Limitations:** Higher computational cost due to longer paths. Not applicable to emissive media.
- **When to Use:** When rendering non-emissive volumes with loose bounding functions where variance reduction is critical.
- **When Not to Use:** When the medium emits light or when computational cost is the primary constraint and variance is acceptable.

## Background
The Absorption Term Restriction is part of the broader field of volume rendering and light transport in participating media. It was discussed in the context of hardware-based view-independent cell projection [S1] and PDF spectral and decomposition tracking [S2]. The technique leverages the properties of delta tracking and null-collision methods to optimize rendering for non-emissive media.

## Sources
[S1] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S2] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
