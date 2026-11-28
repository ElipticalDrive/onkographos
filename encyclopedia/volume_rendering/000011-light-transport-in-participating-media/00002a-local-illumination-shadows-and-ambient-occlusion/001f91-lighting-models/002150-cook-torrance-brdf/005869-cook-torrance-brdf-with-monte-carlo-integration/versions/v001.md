# Cook-Torrance BRDF with Monte Carlo Integration

Also called: Monte Carlo Integration for Cook-Torrance BRDF, NDJIR Forward Rendering Model, Simplified Rendering Equation

## Overview
This method computes the rendered color of a surface by integrating the diffuse and specular components of the Cook-Torrance BRDF over the upper hemisphere using Monte Carlo integration [S1, unofficial]. It distinguishes itself by employing a simplified visibility function derived from the Filament model and utilizing importance sampling for the specular term to prevent light baking artifacts, which uniform sampling fails to mitigate [S1, unofficial].

## How it works
**Inputs:**
*   $\hat{\mathbf{x}}$: Query point on the surface.
*   $\hat{\mathbf{n}}$: Surface normal at $\hat{\mathbf{x}}$.
*   $\mathbf{v}$: Viewing direction.
*   $\hat{\alpha}_r$: Roughness parameter.
*   $\mathbf{f}_0$: Specular reflectance (Fresnel coefficient at normal incidence).
*   $c_b$: Base color.
*   $\widehat{L}_{pb}$: Photogrammetric light intensity.
*   $L_E(\mathbf{l})$: Environment light intensity as a function of direction $\mathbf{l}$.
*   $\widehat{L}_I(\hat{\mathbf{x}})$: Implicit illumination term.
*   $N$: Number of Monte Carlo samples.

**Outputs:**
*   $\widehat{L}_{out}$: Estimated rendered color.

**Data Structures:**
*   Sample directions $\mathbf{l}_i$: Vectors sampled from the upper hemisphere $\Omega_+$, weighted by the specular distribution.

**Formulas:**

1.  **Rendering Equation:**
    The total radiance is estimated as:
    $$ \widehat{L}_{out} \approx \frac{\widehat{L}_{pb} c_b}{\pi N} \sum_{i=1}^{N} \left( L_V(\hat{\mathbf{x}}, \mathbf{l}_i) L_E(\mathbf{l}_i) (\hat{\mathbf{n}} \cdot \mathbf{l}_i) + \widehat{L}_I(\hat{\mathbf{x}}) \right) $$
    where $L_V$ is the soft visibility function [S1, unofficial].

2.  **Cook-Torrance Components:**
    *   **Normal Distribution Function (NDF) $D$:**
        $$ D = \frac{\hat{\alpha}_r^2}{\pi \left( (\hat{\mathbf{n}} \cdot \mathbf{h})^2 (\hat{\alpha}_r^2 - 1) + 1 \right)^2} $$
    *   **Geometry Function $G$:**
        $$ G = \frac{2(\hat{\mathbf{n}} \cdot \mathbf{l})}{(\hat{\mathbf{n}} \cdot \mathbf{l}) + \sqrt{\hat{\alpha}_r^2 + (1 - \hat{\alpha}_r^2)(\hat{\mathbf{n}} \cdot \mathbf{l})^2}} \times \frac{2(\hat{\mathbf{n}} \cdot \mathbf{v})}{(\hat{\mathbf{n}} \cdot \mathbf{v}) + \sqrt{\hat{\alpha}_r^2 + (1 - \hat{\alpha}_r^2)(\hat{\mathbf{n}} \cdot \mathbf{v})^2}} $$
    *   **Fresnel Term $F$:**
        $$ F = \mathbf{f}_0 + (1 - \mathbf{f}_0)(1 - (\mathbf{v} \cdot \mathbf{h}))^5 $$

3.  **Simplified Visibility Function $V$:**
    The method uses a simplified visibility function $V$ derived from the Filament model, which approximates the combined effect of $G$ and other factors:
    $$ V := \frac{G}{4(\hat{\mathbf{n}} \cdot \mathbf{v})(\hat{\mathbf{n}} \cdot \mathbf{l})} = \frac{1}{(\hat{\mathbf{n}} \cdot \mathbf{l}) + \sqrt{\hat{\alpha}_r^2 + (1 - \hat{\alpha}_r^2)(\hat{\mathbf{n}} \cdot \mathbf{l})^2}} \times \frac{1}{(\hat{\mathbf{n}} \cdot \mathbf{v}) + \sqrt{\hat{\alpha}_r^2 + (1 - \hat{\alpha}_r^2)(\hat{\mathbf{n}} \cdot \mathbf{v})^2}} $$
    where $\hat{\mathbf{n}}$ is the normal, $\mathbf{v}$ is the viewing direction, $\mathbf{l}$ is the light direction, and $\hat{\alpha}_r$ is roughness [S1, unofficial].

4.  **Half Vector $\mathbf{h}$:**
    $$ \mathbf{h} = \frac{\mathbf{v} + \mathbf{l}}{\|\mathbf{v} + \mathbf{l}\|} $$

**Steps:**
1.  Initialize sum $S = 0$.
2.  For $i = 1$ to $N$:
    a. Sample a light direction $\mathbf{l}_i$ from the upper hemisphere $\Omega_+$ using importance sampling based on the specular term (Eq. 6 in [S1, unofficial]).
    b. Compute the half vector $\mathbf{h}_i = \text{normalize}(\mathbf{v} + \mathbf{l}_i)$.
    c. Compute $D_i$, $G_i$, and $F_i$ using the formulas above.
    d. Compute the simplified visibility $V_i$ using the formula above.
    e. Evaluate the integrand: $term_i = V_i \cdot L_E(\mathbf{l}_i) \cdot (\hat{\mathbf{n}} \cdot \mathbf{l}_i) + \widehat{L}_I(\hat{\mathbf{x}})$.
    f. Add to sum: $S \leftarrow S + term_i$.
3.  Compute final radiance: $\widehat{L}_{out} = \frac{\widehat{L}_{pb} c_b}{\pi N} S$.

**Parameters:**
*   $N$: Number of samples. Typical values are not explicitly given, but increasing $N$ reduces noise [S1, unofficial].
*   $\hat{\alpha}_r$: Roughness. Range $[0, 1]$.
*   $\mathbf{f}_0$: Specular reflectance. Typically $0.04$ for dielectrics, higher for metals [G].

**Edge Cases and Pitfalls:**
*   **Uniform Sampling:** Using uniform sampling instead of importance sampling causes "light baking," where high light intensity and specularity are incorrectly baked into the base color [S1, unofficial].
*   **Under/Overestimation:** The method may underestimate or overestimate values in specific regions, such as the left cheek and chest of scan69 or the left jaw of scan65 [S1, unofficial].
*   **Regularization:** Strong regularization of the implicit illumination term can make the base color flatter, removing small details like dot eyes and thin mouths [S1, unofficial].
*   **Metalness:** Metalness is not modeled in this method; metallic conversion is object-agnostic [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of Monte Carlo samples.
*   **Memory Complexity:** $O(1)$ per sample, assuming environment light and implicit illumination are accessible.
*   **Strengths:** Importance sampling effectively models specularity and mitigates light baking artifacts compared to uniform sampling [S1, unofficial].
*   **Limitations:** Can produce noisy artifacts or non-smooth geometry if dense voxel grid features are used without proper filtering [S1, unofficial]. May under/overestimate in complex geometry regions [S1, unofficial].
*   **When to use:** When accurate specular highlights are required and uniform sampling leads to baking artifacts.
*   **When not to use:** When metalness needs to be physically modeled (not supported) or when high-frequency details are critical and regularization flattens them [S1, unofficial].

## Background
The method is part of the NDJIR (Neural Direct and Joint Inverse Rendering) framework, proposed for geometry, lights, and materials of real objects tailored to photogrammetry settings [S1, unofficial]. It builds on the Cook-Torrance BRDF model [12] and the rendering equation [30], adapting them for neural inverse rendering with simplified visibility and implicit illumination terms [S1, unofficial].

## Sources
[S1] NDJIR: Neural Direct and Joint Inverse Rendering for Geometry, Lights, and Materials of Real Object, https://doi.org/10.48550/arxiv.2302.00675, fetched 2026-10-26 (unofficial)
