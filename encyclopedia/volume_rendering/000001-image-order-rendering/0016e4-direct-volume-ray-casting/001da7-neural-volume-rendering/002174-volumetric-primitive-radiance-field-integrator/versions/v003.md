# Volumetric Primitive Radiance Field Integrator


## Overview
The Volumetric Primitive Radiance Field Integrator is a path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds [S1, S2, unofficial]. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation [S1, unofficial]. It distinguishes itself from its sibling, the Volumetric Primitive Tomography Integrator, by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering [S1, unofficial].

## How it works
The integrator inherits from `RBIntegrator` and operates on ellipsoid-shaped primitives defined by Gaussian or Epanechnikov kernels for transmittance and spherical harmonics (SH) for emission [S1, unofficial]. It builds on [Direct volume ray casting](19-direct-volume-ray-casting.md) but replaces the discrete volume texture lookup with a neural network query [Parent]. The core mechanism follows the differentiable volume rendering integral [S1, S14, unofficial]:

1.  **Ray Sampling:** For each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is generated. Sample points $\mathbf{p}_i$ are sampled along the ray segment $[t_{start}, t_{end}]$ [S1, S14, unofficial].
2.  **Neural Query:** At each sample point $\mathbf{p}_i$, a neural network (typically an MLP) predicts the raw density $\sigma_i$ and color $\mathbf{c}_i$ (and optionally view-dependent color) [S1, S14, unofficial].
3.  **Alpha Conversion:** The raw density $\sigma_i$ is converted to opacity $\alpha_i$ using the discretized volume rendering equation:
    $$ \alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i) $$
    where $\delta_i$ is the distance between sample points [S1, S14, unofficial].
4.  **Differentiable Compositing:** The final pixel color is computed via front-to-back compositing, which is differentiable with respect to the network outputs, allowing gradients to flow back to the network weights during training [S1, S14, unofficial]:
    $$ C = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$

### Inputs and Outputs
*   **Inputs:**
    *   Scene containing `ellipsoidsmesh` primitives. Each primitive has properties: center $\mathbf{c}_i$, rotation matrix $\mathbf{R}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and SH coefficients for emission.
    *   Ray $r(t) = \mathbf{o} + t\mathbf{d}$ for each pixel.
    *   Parameters: `max_depth`, `rr_depth`, `kernel_type` ('gaussian' or 'epanechnikov'), `srgb_primitives`, `use_rr`.
*   **Outputs:**
    *   Radiance $L$ (primal mode) or differential radiance $\delta L$ (adjoint mode).

### Data Structures
*   **Primitive Storage:**
    *   `centers`: Array of shape $(N, 3)$, float32.
    *   `rotations`: Array of shape $(N, 3, 3)$, float32.
    *   `scales`: Array of shape $(N, 3)$, float32.
    *   `opacities`: Array of shape $(N, 1)$, float32.
    *   `sh_coeffs`: Array of shape $(N, K)$, float32, where $K$ is the number of SH coefficients.
*   **Ray State (per ray):**
    *   `active`: Boolean mask.
    *   `depth`: Integer counter.
    *   `L`: Radiance accumulator, shape $(3)$, float32.
    *   `δL`: Differential radiance accumulator, shape $(3)$, float32.
    *   `β`: Path throughput, float32.

### Numbered Steps
1.  **Initialization:**
    *   Set `max_depth` to `0xFFFFFFFF` if input is `-1`, else use input.
    *   Set `rr_depth` to `0xFFFFFFFF` if input is `-1`, else use input.
    *   Determine `use_rr`: `True` if `rr_depth >= 0` and (`rr_depth < max_depth` or `max_depth == -1`), else `False`.
    *   Initialize kernel with `kernel_full_range=True`, `kernel_normalized=True`.
    *   For each ray: `L = 0`, `δL = 0`, `β = 1.0`, `depth = 0`, `active = True`.
    *   Detach ray for primal mode.

2.  **Ray Intersection Loop (while `active`):**
    *   Intersect ray with scene using `scene.ray_intersect` with flags `All | BackfaceCulling` and coherent intersection for `depth == 0`.
    *   Filter active rays: `active &= si.is_valid() & (si.shape.shape_type() == +mi.ShapeType.Ellipsoids)`.
    *   If `active` is empty, break.

3.  **Evaluate Emission and Transmission:**
    *   For active rays, gather primitive data (ellipsoid, sh_coeffs, opacity).
    *   **Transmission (`eval_transmission`):**
        *   Compute peak location: $\mathbf{o}' = \mathbf{R}_i^T (\mathbf{o} - \mathbf{c}_i) / \mathbf{s}_i$, $\mathbf{d}' = \mathbf{R}_i^T \mathbf{d} / \mathbf{s}_i$.
        *   $t_{peak} = -\frac{\mathbf{o}' \cdot \mathbf{d}'}{\mathbf{d}' \cdot \mathbf{d}'}$.
        *   Evaluate density at $\mathbf{p}_{peak} = \mathbf{o} + t_{peak} \mathbf{d}$.
        *   `transmission` = $1.0 - \min(\text{opacity} \cdot \text{density}, 0.9999)$. [S1, unofficial] (Clipping avoids numerical issues with $1.0 - 1.0$).
    *   **Emission (`eval_sh_emission`):**
        *   Compute SH degree: $L = \sqrt{\frac{\text{sh\_coeffs.shape[0]}}{3} - 1}$.
        *   Evaluate SH direction coefficients: $\mathbf{sh\_dir\_coef} = \text{sh\_eval}(\mathbf{d}, L)$.
        *   Sum emission: $\text{emission} = \sum (\text{sh\_coeffs} \cdot \mathbf{sh\_dir\_coef})$.
        *   Return $\max(\text{emission} + 0.5, 0.0)$. [S1, unofficial] (Adding 0.5 handles potential negative values from SH evaluation).

4.  **Update Radiance and Throughput:**
    *   Clamp infinite Le values to 0: `Le[~dr.isfinite(Le)] = 0.0`.
    *   Compute local emission contribution: $L_e = \beta \cdot (1.0 - \text{transmission}) \cdot \text{emission}$. [S1, unofficial]
    *   Update radiance accumulator:
        *   If primal: `L[active] = L + Le`.
        *   If adjoint: `L[active] = L - Le`.
    *   Update path throughput: `β[active] *= transmission`.

5.  **Adjoint Indirect Radiance (Adjoint Mode Only):**
    *   Compute differentiable reflected indirect radiance: $L_{r\_ind} = L \cdot \text{transmission} / \text{dr.detach}(\text{transmission})$.
    *   Compute total local contribution: $L_o = L_e + L_{r\_ind}$.
    *   Select finite values: $L_o = \text{dr.select}(\text{active} \& \text{dr.isfinite}(L_o), L_o, 0.0)$.
    *   Propagate gradients:
        *   If Backward: `dr.backward_from(δL * Lo)`.
        *   If Forward: `δL += dr.forward_to(Lo)`.

6.  **Update State and Kill Paths:**
    *   Update active mask: `active &= si.is_valid()`.
    *   Increment depth: `depth[active] += 1`.
    *   Kill paths with insignificant contribution:
        *   `β_max = dr.max(β)`.
        *   `active &= (β_max > 0.01)`.

7.  **Russian Roulette (if `primal` and `use_rr`):**
    *   Calculate `rr_prob = dr.maximum(β_max, 0.1)`.
    *   Identify active RR candidates: `rr_active = (depth >= rr_depth) & (β_max < 0.1)`.
    *   Scale throughput: `β[rr_active] *= dr.rcp(rr_prob)`.
    *   Determine continuation: `rr_continue = sample_rr < rr_prob`.
    *   Update active: `active &= rr_continue`.

8.  **Limit Recursion Depth:**
    *   `active &= depth < max_depth`.

9.  **Final Conversion:**
    *   If `srgb_primitives` is true, convert $L$ to linear space: `L = mi.math.srgb_to_linear(L)`.

10. **Return:**
    *   Return $L$ if primal, else $\delta L$.

### Formulas
*   **Emission:** $L_e = \beta \cdot (1.0 - \text{transmission}) \cdot \text{emission}$ [S1, unofficial]
*   **Transmission:** $\text{transmission} = 1.0 - \min(\alpha \cdot \sigma(\mathbf{p}_{peak}), 0.9999)$ [S1, unofficial]
*   **Peak Location:** $t_{peak} = -\frac{\mathbf{o}' \cdot \mathbf{d}'}{\mathbf{d}' \cdot \mathbf{d}'}$ where $\mathbf{o}' = \mathbf{R}^T (\mathbf{o} - \mathbf{c}) / \mathbf{s}$, $\mathbf{d}' = \mathbf{R}^T \mathbf{d} / \mathbf{s}$ [S1, unofficial]
*   **SH Emission:** $\text{emission} = \max(\sum (\text{sh\_coeffs} \cdot \text{sh\_eval}(\mathbf{d}, L)) + 0.5, 0.0)$ [S1, unofficial]

### Parameters
*   `max_depth` (int): Maximum path depth. A value of `-1` indicates no limit. [S1, unofficial]
*   `kernel_type` (str): Name of the kernel to use for rendering the volumetric primitives, one of `['gaussian', 'epanechnikov']`. [S1, unofficial]
*   `srgb_primitives` (bool): If true, converts sRGB light transport to linear color space at the end. Default `True`. [S1, unofficial]
*   `use_rr` (bool): Whether to use Russian Roulette. [S1, unofficial]
*   `rr_depth` (int): Minimum path depth before enabling Russian roulette. `-1` indicates no limit. [S1, unofficial]

### Edge Cases and Pitfalls
*   **Numerical Stability:** Transmission calculation clips `opacity * density` to `0.9999` to avoid numerical issues with `1.0 - 1.0` [S1, unofficial].
*   **SH Emission:** Emission calculation adds `0.5` before clamping to `0.0` to handle potential negative values from SH evaluation [S1, unofficial].
*   **Infinite Values:** Infinite Le values are clamped to 0 to prevent NaN propagation [S1, unofficial].
*   **Self-Intersection:** New ray origin is spawned to avoid self-intersection: `ray.o[active] = si.p + ray.d * 1e-4` [S1, unofficial].

## Subtopics

### Epanechnikov Kernel Volumetric Primitive
The Epanechnikov Kernel Volumetric Primitive is a variant of the Volumetric Primitive Radiance Field Integrator that uses the Epanechnikov kernel for transmittance instead of the Gaussian kernel [S2, unofficial]. It computes transmittance using a closed-form solution [S2, unofficial]. Free-flight distance sampling is complex analytically, so numerical root-finding methods (e.g., Newton-Raphson, bisection) are generally used [S2, unofficial]. The 3D Epanechnikov kernel is defined as:
$$ E_i(\mathbf{x}) = \begin{cases} \frac{15}{8\pi} (\frac{7}{3} |\Sigma_i| )^{1/2} [1 - \frac{1}{7} d(\mathbf{x})] & \text{if } d(\mathbf{x}) \le 1 \\ 0 & \text{otherwise} \end{cases} $$
where $d(\mathbf{x}) = (\mathbf{x} - \mathbf{c}_i)^T \Sigma_i^{-1} (\mathbf{x} - \mathbf{c}_i)$ [S2, unofficial]. The Epanechnikov kernel has finite support, generating more compact primitive shells than Gaussian, enabling more efficient acceleration structures and faster rendering, especially suitable for modeling sharp edges [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of primitives intersected by rays and the depth of path tracing. Epanechnikov kernel is faster than Gaussian due to finite support and compact shells [S2, unofficial].
*   **Memory Complexity:** Proportional to the number of primitives and their SH coefficient size.
*   **Performance:** Faster than MLP-based neural volume rendering (e.g., Instant-NGP) and comparable to 3D Gaussian Splatting, but with physically accurate volume integration [S2, unofficial].
*   **Strengths:** Physically based, supports scattering and emission, integrates with path tracers, supports complex camera models [S2, unofficial].
*   **Limitations:** Requires numerical root-finding for free-flight sampling in complex scenes, may suffer from popping artifacts if not properly handled (unlike 3DGS which has its own issues) [S2, unofficial].
*   **When to Use:** When physically accurate volumetric rendering with scattering and emission is required, and performance is critical.
*   **When Not to Use:** When simple absorption-only rendering is sufficient (use Tomography Integrator) or when real-time performance on very low-end hardware is required without GPU acceleration.

## Background
The Volumetric Primitive Radiance Field Integrator was introduced by Meta Platforms, Inc. in the `volprim` library [S1, unofficial]. It is designed to bridge the gap between implicit neural models and explicit voxel-based representations by using kernel-based volumetric primitives [S2, unofficial]. It inherits from `RBIntegrator` in Mitsuba 3 [S1, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
