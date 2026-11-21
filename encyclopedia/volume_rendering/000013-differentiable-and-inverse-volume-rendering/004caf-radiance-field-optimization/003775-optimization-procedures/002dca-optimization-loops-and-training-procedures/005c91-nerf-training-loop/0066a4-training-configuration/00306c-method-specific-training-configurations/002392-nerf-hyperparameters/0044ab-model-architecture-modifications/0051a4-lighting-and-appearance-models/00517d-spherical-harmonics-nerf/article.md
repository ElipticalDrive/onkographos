# Spherical Harmonics NeRF
Also called: NeRF with Spherical Harmonics, SH-NeRF

## Overview
Spherical Harmonics NeRF replaces the standard view-dependent color prediction head of a NeRF with an explicit 2nd-order Spherical Harmonics (SH) illumination model, allowing for explicit relighting by varying learnable SH coefficients [S1]. It differs from sibling methods like Neural-PIL and StylizedNeRF by decoupling geometry (density) and appearance (albedo) into separate accumulations, then applying a global, per-image SH lighting model to the accumulated screen-space albedo and normals, rather than predicting color directly per sample or using learned style embeddings [S1].

## How it works
The method separates the neural network outputs into density $\sigma$, albedo $\mathbf{A}$, and normal gradients, accumulating them along rays. The final color is computed by multiplying the accumulated albedo by the SH illumination evaluated at the accumulated normal.

**Inputs and Outputs:**
*   **Inputs:** Ray samples $\{\mathbf{x}_i\}_{i=1}^{N_{\mathrm{depth}}}$, where $\mathbf{x}_i$ are 3D positions; per-image learnable SH coefficients $\mathbf{L} \in \mathbb{R}^{9\times3}$; neural network parameters for density $\sigma(\mathbf{x})$ and albedo $\mathbf{A}_{\mathrm{sample}}(\mathbf{x})$.
*   **Outputs:** Rendered color $\mathbf{C} \in \mathbb{R}^3$.

**Data Structures:**
*   $\mathbf{L}$: A matrix of shape $(9, 3)$ storing the 9 SH coefficients for each of the 3 RGB channels. This is stored as a learnable parameter per image.
*   $\mathbf{b}(\mathbf{n})$: A vector of shape $(9,)$ containing the 2nd-order SH basis functions evaluated at normal $\mathbf{n}$. These are fixed, non-learnable values.

**Numbered Steps:**

1.  **Accumulate Density and Albedo:** For each ray, sample points $\mathbf{x}_i$. Query the density MLP for $\sigma(\mathbf{x}_i)$ and the albedo MLP for $\mathbf{A}_{\mathrm{sample}}(\mathbf{x}_i)$. Compute transmittance $T(t_i)$ and opacity $\alpha(\sigma(\mathbf{x}_i)\delta_i)$ as in standard volume rendering. Accumulate the weighted albedo:
    $$ \mathbf{A} = \sum_{i=1}^{N_{\mathrm{depth}}} \mathbf{A}_{\mathrm{sample}}(\mathbf{x}_i) T(t_i) \alpha(\sigma(\mathbf{x}_i)\delta_i) $$
    where $\mathbf{A} \in \mathbb{R}^3$ is the accumulated screen-space albedo.

2.  **Accumulate Unnormalized Normal:** Compute the gradient of the density with respect to the sample positions $\frac{\partial}{\partial \mathbf{x}_i}\sigma(\mathbf{x}_i)$. Accumulate these gradients weighted by the visibility term $T(t_i)\alpha(\sigma(\mathbf{x}_i)\delta_i)$:
    $$ \mathbf{\hat{N}} = \sum_{i=1}^{N_{\mathrm{depth}}} \left( \frac{\partial}{\partial \mathbf{x}_i}\sigma(\mathbf{x}_i) \right) \odot T(t_i)\alpha(\sigma(\mathbf{x}_i)\delta_i) $$
    where $\mathbf{\hat{N}} \in \mathbb{R}^3$ is the unnormalized normal vector.

3.  **Normalize Normal:** Normalize the accumulated vector to a unit sphere to obtain the final normal $\mathbf{N}$:
    $$ \mathbf{N} = \frac{\mathbf{\hat{N}}}{\|\mathbf{\hat{N}}\|^2} $$
    Note: The source formula [S1] uses $\|\mathbf{\hat{N}}\|^2$ in the denominator, which is mathematically equivalent to normalizing to unit length if interpreted as dividing by the squared magnitude to achieve a unit vector (though standard normalization divides by $\|\mathbf{\hat{N}}\|$). We follow the source's explicit formula [S1].

4.  **Evaluate SH Basis:** Evaluate the 2nd-order SH basis functions at the normal $\mathbf{N}$ to get $\mathbf{b}(\mathbf{N}) \in \mathbb{R}^9$.

5.  **Compute Illumination:** Multiply the per-image SH coefficients $\mathbf{L}$ by the basis vector:
    $$ \mathbf{L}_{\mathrm{illuminated}} = \mathbf{L} \mathbf{b}(\mathbf{N}) $$
    This results in a vector in $\mathbb{R}^3$ representing the incident lighting color.

6.  **Render Color:** Compute the final color by element-wise multiplying the accumulated albedo $\mathbf{A}$ by the illuminated lighting term:
    $$ \mathbf{C} = \mathbf{A} \odot (\mathbf{L} \mathbf{b}(\mathbf{N})) $$

**Formulas:**
*   Rendering Equation:
    $$ \mathbf{C}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}},\textbf {L}\right )= \mathbf {A}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right )\odot \mathbf {L}\mathbf {b}\left (\mathbf {N}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right )\right ) $$ [S1]
*   Normal Extraction:
    $$ \mathbf {N}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right ) = \frac {\mathbf {\hat {N}}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right )}{\left \|\mathbf {\hat {N}}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right )\right \|^2} $$ [S1]
*   Unnormalized Normal:
    $$ \mathbf {\hat {N}}\left (\{\mathbf {x}_i\}_{i=1}^{N_\mathrm {depth}}\right ) = \sum _{i=1}^{N_\mathrm {depth}}\left (\frac {\partial }{\partial \mathbf {x}_i}\sigma (\mathbf {x}_i)\right )\odot T(t_i)\alpha (\sigma (\mathbf {x}_i)\delta _i) $$ [S1]

**Parameters:**
*   $\mathbf{L} \in \mathbb{R}^{9\times3}$: Per-image learnable SH coefficients for illumination. [S1]
*   $\mathbf{b}(\mathbf{n}) \in \mathbb{R}^9$: SH basis vector evaluated at normal $\mathbf{n}$. [S1]

**Edge Cases and Pitfalls:**
*   **Shadow Generation:** The model lacks direct shadow generation, which is crucial for modeling and subsequent relighting of outdoor scenes [S1]. A dedicated shadow model $S$ can be introduced to extend the rendering equation, but the base SH-NeRF method does not include it [S1].
*   **Normal Normalization:** The source specifies dividing by $\|\mathbf{\hat{N}}\|^2$ rather than $\|\mathbf{\hat{N}}\|$. Implementers should verify this specific normalization convention, as it differs from standard unit vector normalization [S1].
*   **Single Shading Calculation:** The method performs a single shading calculation per ray using accumulated albedo and normals, rather than shading each sample point individually. This reduces noise and aids convergence but assumes the lighting is consistent across the accumulated depth [S1].

## Operating characteristics
*   **Time Complexity:** The additional cost per ray is the evaluation of the SH basis $\mathbf{b}(\mathbf{N})$ and the matrix-vector multiplication $\mathbf{L}\mathbf{b}(\mathbf{N})$, which is constant time $O(1)$ relative to the number of samples, as the order of SH is fixed at 2nd order (9 coefficients).
*   **Memory Complexity:** Requires storing $\mathbf{L}$ for each image, which is $9 \times 3 = 27$ floats per image. This is negligible compared to the neural network weights.
*   **Strengths:** Allows explicit relighting by varying $\mathbf{L}$; separates geometry and appearance; reduces noise in albedo and normal estimates through accumulation [S1].
*   **Limitations:** Cannot model direct shadows without an additional shadow network; assumes Lambertian shading (implied by the albedo $\times$ lighting model) [S1].
*   **When to use:** When explicit relighting of outdoor or indoor scenes is required and the scene can be approximated as Lambertian.
*   **When not to use:** When accurate shadow modeling is critical and an additional shadow network is not feasible; when view-dependent specular effects are dominant and cannot be captured by 2nd-order SH.

## Background
The method was introduced by V. Rudnev et al. in "NeRF for Outdoor Scene Relighting" [S1]. It builds on the standard NeRF framework by replacing the view-dependent color prediction with an explicit SH illumination model to allow for relighting. The approach addresses the limitation of standard NeRFs, which do not have semantic meaning of underlying scene intrinsics or direct control over lighting [S1].

## Sources
[S1] NeRF for Outdoor Scene Relighting, https://doi.org/10.1007/978-3-031-19787-1_35, fetched 2026-10-26
