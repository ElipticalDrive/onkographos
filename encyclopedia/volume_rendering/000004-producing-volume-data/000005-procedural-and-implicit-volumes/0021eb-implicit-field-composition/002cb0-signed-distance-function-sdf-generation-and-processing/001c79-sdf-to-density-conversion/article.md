# SDF to Density Conversion

Also called: signed distance to density conversion

## Overview
SDF to Density Conversion transforms a Signed Distance Function (SDF) $d(\mathbf{x})$ into a volumetric density field $\sigma(\mathbf{x})$ suitable for differentiable volume rendering and alpha compositing [S1][S4][S7, unofficial]. It addresses the mismatch between geometric distance values and optical absorption properties, enabling the integration of color and opacity along rays [S5, unofficial]. It differs from standard Volume Rendering by deriving density directly from an implicit function rather than pre-computed volumes, and from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) by its reliance on exponential or sigmoid-based mappings to ensure alpha-invariance [S3, unofficial][S4][S11].

## How it works
**Inputs and Outputs:**
*   **Input:** Signed distance value $d$ (or $s = -d$) at a point $\mathbf{x}$ along a ray; ray segment length $\Delta t$; learnable or tuned scalar parameters $\alpha$ (scaling) and $\beta$ (falloff/sharpness) [S4][S7, unofficial].
*   **Output:** Volumetric density $\sigma$ (or $\rho$) and optionally a color/feature vector $c$ [S4][S7, unofficial].

**Data Structures:**
*   **Ray Bundle:** Contains sample points $\mathbf{p}_i$ and depth values $t_i$ along the ray [S4][S7, unofficial].
*   **Parameters:** $\alpha \in \mathbb{R}^+$, $\beta \in \mathbb{R}^+$ [S4][S7, unofficial].

**Steps:**
1.  **Sample SDF:** Evaluate the SDF at the sample points along the ray to obtain signed distances $d_i$ [S7, unofficial].
2.  **Compute Segment Lengths:** Calculate the distance between consecutive sample points $\Delta t_i = t_{i+1} - t_i$. Pad the last segment with a large value (e.g., $10^{10}$) to prevent division by zero when the ray exits the sampling volume [S5, unofficial].
3.  **Convert SDF to Intermediate Density:** Map the signed distance to an intermediate density $\rho$ using a piecewise exponential function controlled by $\beta$ [S8, unofficial][S5, unofficial]. Let $s = -d$ (so $s > 0$ is interior, $s < 0$ is exterior) [S7, unofficial]. Compute $\psi$:
    $$ \psi = \begin{cases} 0.5 \cdot \exp(s / \beta) & \text{if } s \le 0 \\ 1 - 0.5 \cdot \exp(-s / \beta) & \text{if } s > 0 \end{cases} $$
    [S8, unofficial]. Scale by $\alpha$:
    $$ \rho = \alpha \cdot \psi $$
    [S8, unofficial].
4.  **Compute Final Density:** Convert the intermediate density $\rho$ to volumetric density $\sigma$ by accounting for the segment length $\Delta t$ [S5, unofficial][S8, unofficial]:
    $$ \sigma = -\frac{\ln(1 - \rho)}{\Delta t} $$
    [S5, unofficial]. *Note:* If $\rho$ is not clamped to $[0, 1)$, $\ln(1-\rho)$ may be undefined. Ensure $\rho < 1$ [G].
5.  **Compute Color/Feature (Optional):** Compute `base_color`: If `rainbow` mode is enabled, `base_color` is derived from the position relative to the SDF center, clamped to $[0.02, 0.98]$ [S8, unofficial]. Otherwise, `base_color` is $1.0$ [S8, unofficial]. Compute final feature: $c = \text{base\_color} \cdot \text{feature\_param}$ [S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def sdf_to_density(signed_distance, alpha, beta):
    """Converts signed distance values to intermediate density."""
    s = -signed_distance
    psi = torch.where(
        s <= 0.0,
        0.5 * torch.exp(s / beta),
        1 - 0.5 * torch.exp(-s / beta)
    )
    return alpha * psi

def compute_volumetric_density(intermediate_density, delta_t):
    """Converts intermediate density to volumetric density sigma."""
    rho = torch.clamp(intermediate_density, min=0.0, max=0.9999)
    return -torch.log(1.0 - rho) / delta_t

def render_step(ray_bundle, sdf_fn, alpha, beta, rainbow=False, center=None, feature_param=None):
    """Full rendering step for a ray bundle."""
    points = ray_bundle['sample_points']
    depths = ray_bundle['depths']
    deltas = depths[1:] - depths[:-1]
    deltas = torch.cat([deltas, torch.full_like(deltas[:1], 1e10)])
    signed_distances = sdf_fn(points)
    rho = sdf_to_density(signed_distances, alpha, beta)
    sigma = compute_volumetric_density(rho, deltas)
    if rainbow:
        if center is None:
            center = torch.zeros_like(points)
        base_color = torch.clamp(torch.abs(points - center), 0.02, 0.98)
    else:
        base_color = torch.ones_like(points)
    feature = base_color * feature_param if feature_param is not None else base_color
    return {'density': sigma, 'feature': feature}
```

**Parameters:**
*   $\alpha$: Scalar multiplier for intermediate density, controls overall opacity [S4][S7, unofficial]. Initialized from configuration or learned [S7, unofficial].
*   $\beta$: Scalar controlling the exponential falloff rate of density from the surface. Smaller values yield sharper surfaces [S4][S7, unofficial]. Initialized from configuration or learned [S7, unofficial].
*   `rainbow`: Boolean flag; if true, computes `base_color` as `clamp(|sample_points - center|, 0.02, 0.98)`, otherwise sets it to $1.0$ [S4][S7, unofficial].
*   `sharpness`: In some contexts (e.g., IOR), controls the gradient of the transition [S2].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The last segment's delta is padded with a large value (e.g., $10^{10}$) to prevent division by zero or undefined behavior when the ray exits the sampling volume [S4][S7, unofficial][S11].
*   **Log Domain:** Ensure intermediate density $\rho < 1$ before applying $\ln(1-\rho)$ [S4]. [G]
*   **Sign Convention:** The sign convention follows the parent article: $d < 0$ indicates interior and $d > 0$ indicates exterior [Parent]. The conversion function must handle this correctly (e.g., $s = -d$) [S7, unofficial].

## Subtopics

### [Procedural Index of Refraction Distribution](1810-procedural-index-of-refraction-distribution.md)
Generates a spatially varying index of refraction (IOR) field to simulate optical phenomena such as mirages. It differs from density-based volume methods by explicitly modeling refractive properties rather than just absorption or emission, allowing for ray bending effects. Choose when simulating temperature-induced density gradients in air.

### [Box with Smoothed/Eroded Edges](3417-box-with-smoothed-eroded-edges.md)
Converts an SDF value into volumetric density for a box primitive, applying a sigmoid-based erosion at the boundaries to create smooth, anti-aliased edges and adding Perlin noise to roughen the surface. It differs from standard conversions by explicitly expanding the box geometry to define an erosion zone and combining axis-aligned sigmoid falloffs with spatial noise. Choose when rendering box primitives with soft, noisy boundaries.

### [SDF Alpha Computation](3643-sdf-alpha-computation.md)
Computes the output alpha for an SDF given the Screen Space Ratio (SSR), Signed Distance (SD), Signed Distance Ratio (SDR), isoPerimeter, and softness. It converts a signed distance value into a density value using alpha and beta parameters. Choose when direct alpha compositing is preferred over volumetric integration.

### [Gray Transform (G_o, G_s)](10492-gray-transform-g-o-g-s.md)
Converts a discrete object description or a continuous surface description into a volumetric scene (density field) suitable for physical modeling, manufacturing, or volume rendering. It distinguishes itself by explicitly defining density values based on interior/exterior status or signed distance to a surface. Choose when mapping discrete CAD or mesh data to volumetric representations.

### [SDF Volume Rendering](19065-sdf-volume-rendering.md)
Integrates color and density along rays sampled from an implicit SDF. It converts SDF values to density using a Laplace distribution CDF and aggregates color using standard volume rendering weights. Unlike standard volume rendering which uses raw density fields, this method leverages the geometric priors of the SDF to better recover underlying geometry. Choose when recovering geometry from implicit fields is the primary goal.

### FBM-based Implicit Density Definition
Defines an implicit volumetric object where density $\rho$ is derived from an FBM field $f$ [S9]. Density is zero when $f(\mathbf{p}) < v$, and remapped via function $h$ when $f(\mathbf{p}) \ge v$ [S9]. The 0-isosurface of $\rho$ corresponds to the v-isosurface of $f$ [S9].
Formula:
$$ \rho(\mathbf{p}) = \begin{cases} h(f(\mathbf{p})) & \text{when } f(\mathbf{p}) \geq v \\ 0 & \text{otherwise} \end{cases} $$
where $f(\mathbf{p}) = B_n(\mathbf{p})$ is the FBM field, $v$ is the threshold, and $h$ is a positive remapping function [S9].
Pitfall: If $h$ is not a positive function, the 0-isosurface of $\rho$ will not correspond to the v-isosurface of $f$. The method assumes $h$ is positive [S9].

### Density-based Volume Region
DensityRegion extends VolumeRegion by scaling absorption, scattering, extinction, and emission coefficients by a spatially varying density function `density(pt)` [S10, unofficial].
Formula:
$\sigma_a(\text{pt}) = \text{sig\_a} \cdot \text{density}(\text{pt})$, where $\text{sig\_a}$ is the base absorption spectrum [S10, unofficial].
$\sigma_s(\text{pt}) = \text{sig\_s} \cdot \text{density}(\text{pt})$, where $\text{sig\_s}$ is the base scattering spectrum [S10, unofficial].
$\sigma_t(\text{pt}) = (\text{sig\_a} + \text{sig\_s}) \cdot \text{density}(\text{pt})$, representing extinction [S10, unofficial].
$L_{ve}(\text{pt}) = \text{le} \cdot \text{density}(\text{pt})$, where $\text{le}$ is the base emission spectrum [S10, unofficial].
$\text{phase}(\text{pt}, \mathbf{w}, \mathbf{w}') = \text{PhaseHG}(\mathbf{w}, \mathbf{w}', g_{\text{scatter}})$, using Henyey-Greenstein phase function with parameter $g_{\text{scatter}}$ [S10, unofficial].
Parameter: $g_{\text{scatter}}$: Henyey-Greenstein anisotropy parameter, passed in constructor as `greenstein` [S10, unofficial].
Layout: `DensityRegion` fields: `sig_a` (Spectrum), `sig_s` (Spectrum), `le` (Spectrum), `g_scatter` (float) [S10, unofficial].

### Smeared Heaviside Function for Mean Curvature
A smeared-out Heaviside function $H$ used to compute mean curvature for regularization, defined piecewise based on a threshold $\epsilon_H$ [S11].
Formula:
$$ H(\hat{\phi}) = \begin{cases} 0 & \text{if } \hat{\phi} < -\epsilon_H \\ \frac{1}{2} + \frac{\hat{\phi}}{2\epsilon_H} + \frac{1}{2\pi} \sin\left(\frac{\pi\hat{\phi}}{\epsilon_H}\right) & \text{if } -\epsilon_H \leq \hat{\phi} \leq \epsilon_H \\ 1 & \text{if } \hat{\phi} > \epsilon_H \end{cases} $$
where $\epsilon_H$ is the mean edge distance between sites excluding the 5% longer edges [S11].
Parameter: $\epsilon_H$ is equal to the mean edge distance between sites, excluding the 5% longer edges [S11].

### Squared Sigmoid (Fsq)
A variant of the sigmoid function used in GenDR where the absolute part of the distance is squared before applying the base sigmoid, interpreted as applying a transformed CDF $F_{sq}$ [S12].
Step:
1. Multiply the absolute distance by the signed distance: $|d(\mathbf{p}, t)| \cdot d(\mathbf{p}, t)$ [S12].
2. Divide by temperature: $(|d(\mathbf{p}, t)| \cdot d(\mathbf{p}, t)) / \tau$ [S12].
3. Apply the base sigmoid function $F$ to the result [S12].
Formula: $F_{sq}(d(\mathbf{p}, t)/\tau) := F(|d(\mathbf{p}, t)| \cdot d(\mathbf{p}, t)/\tau)$, where $F$ is the base sigmoid function, $d(\mathbf{p}, t)$ is the distance, and $\tau$ is temperature [S12].
Comparison: $F_{sq}$ is generally worse than $F$ for distributions with exponential decay, but comparable or better for those with linear convergence (Cauchy, Reciprocal Sigmoid) [S12].

## Operating characteristics
*   **Strengths:** Provides a smooth, differentiable transition from geometry to density, enabling gradient-based optimization of the SDF [S1][S4][S11]. Alpha-invariant formulations ensure consistent appearance across different scales [S4][S11].
*   **Limitations:** Requires careful tuning of $\alpha$ and $\beta$ (or $s$) to achieve desired sharpness and opacity [S4][S7, unofficial][S11]. The assumption that density magnitude is tied to sharpness can be restrictive for complex materials [S11].
*   **When to Use:** When rendering implicit surfaces defined by SDFs, especially in neural rendering pipelines like VolSDF or NeuS [S4][S11].
*   **When Not to Use:** When explicit density fields are already available and do not need to be derived from geometry [S1].

## Background
*   **Origin:** The method originates from differentiable volume rendering pipelines for implicit surfaces, notably introduced in VolSDF and NeuS [S4][S11].
*   **History:** Key developments include VolSDF, which introduced learnable scaling and shrinking coefficients for alpha-invariance [S4], and NeuS, which used a sigmoid-based transformation to match the CDF of volume rendering [S11].
*   **Usage:** Widely used in neural rendering, 3D reconstruction, and differentiable rendering pipelines [S1][S4][S11].

## Sources
[S1] Real-time volume rendering with octree-based implicit surface representation, https://doi.org/10.1016/j.cagd.2024.102322, fetched 2026-10-27
[S2] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
[S3] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S4] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S5] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S6] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
[S7] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S8] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S9] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S10] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S11] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S12] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
