# Volumetric Primitive Radiance Field Integrator


## Overview
The Volumetric Primitive Radiance Field Integrator is a path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds [S1, S2, unofficial]. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation [S1, unofficial]. Unlike its sibling [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md), which relies on dense neural network queries along fixed ray segments, this integrator treats the volume as a collection of continuous analytical shapes, enabling physically correct global illumination, soft shadows, and view-dependent emission via spherical harmonics [S1, S2, unofficial].

## How it works
The integrator operates as a path tracer, inheriting from `RBIntegrator` [S1, unofficial]. It does not use the fixed-step discretization of [Direct volume ray casting](19-direct-volume-ray-casting.md) but instead samples interaction points based on the optical depth of the primitive mixture.

### Inputs and Outputs
*   **Inputs**: A scene containing `ellipsoidsmesh` primitives. Each primitive $i$ is defined by a center $\mathbf{c}_i$, rotation matrix $\mathbf{R}_i$, scale vector $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic (SH) coefficients for emission. The scene also includes an environment map for indirect illumination.
*   **Outputs**: The final pixel radiance $L$ (primal mode) or its gradient $\delta L$ (adjoint mode).

### Data Structures
*   **Primitive**: Stored in an `ellipsoidsmesh` asset.
    *   `center`: 3D vector $\mathbf{c}_i$.
    *   `rotation`: $3 \times 3$ matrix $\mathbf{R}_i$.
    *   `scale`: 3D vector $\mathbf{s}_i$.
    *   `opacity`: Scalar $\alpha_i \in [0, 1]$.
    *   `sh_coeffs`: Array of spherical harmonic coefficients.
*   **State**:
    *   `L`: Radiance accumulator (vector).
    *   `δL`: Differential radiance (vector).
    *   `β`: Path throughput (vector).
    *   `depth`: Current path depth (integer).
    *   `active`: Boolean mask for active rays.

### Parameters
*   `max_depth` (int): Maximum path depth. `-1` indicates no limit (mapped to `0xFFFFFFFF`) [S1, unofficial].
*   `rr_depth` (int): Minimum path depth before enabling Russian Roulette. `-1` indicates no limit [S1, unofficial].
*   `kernel_type` (str): Kernel function for density evaluation. Options: `'gaussian'`, `'epanechnikov'` [S1, unofficial].
*   `srgb_primitives` (bool): If true, converts final $L$ from sRGB to linear space [S1, unofficial].
*   `use_rr` (bool): Whether to use Russian Roulette. Determined by `rr_depth >= 0` and (`rr_depth < max_depth` or `max_depth == -1`) [S1, unofficial].

### Steps
1.  **Initialization**:
    *   Set `max_depth` to `0xFFFFFFFF` if `-1`.
    *   Set `rr_depth` to `0xFFFFFFFF` if `-1`.
    *   Determine `use_rr`.
    *   Initialize kernel with `kernel_full_range=True`, `kernel_normalized=True` [S1, unofficial].
    *   For each pixel ray:
        *   Detach ray for primal mode (if applicable).
        *   Initialize $L = 0$ (or `state_in`), $\delta L = 0$ (or `δL`), $\beta = 1.0$, `depth` = 0.
        *   Set `active` mask to all true.

2.  **Ray Intersection Loop**:
    *   While `active`:
        *   Intersect ray with scene using `scene.ray_intersect` with flags `All | BackfaceCulling` and `coherent=(depth==0)` [S1, unofficial].
        *   Filter active rays: `active &= si.is_valid() & (si.shape.shape_type() == +mi.ShapeType.Ellipsoids)` [S1, unofficial].
        *   If no active rays, break.

3.  **Primitive Evaluation**:
    *   For active rays intersecting an ellipsoid primitive $i$:
        *   **Emission (`Le`)**:
            *   Evaluate SH emission: `sh_degree = sqrt(sh_coeffs.shape[0] / 3 - 1)`.
            *   Compute `sh_dir_coef = sh_eval(ray.d, sh_degree)`.
            *   `emission = max(sum(sh * sh_coeffs) + 0.5, 0.0)` [S1, unofficial]. The `+0.5` handles potential negative SH values [S1, unofficial].
        *   **Transmission**:
            *   Compute peak location $t_{peak}$ along the ray relative to the ellipsoid:
                $$
                \mathbf{o} = \mathbf{R}_i^T (\mathbf{r}_o - \mathbf{c}_i) / \mathbf{s}_i, \quad \mathbf{d} = \mathbf{R}_i^T \mathbf{r}_d / \mathbf{s}_i
                $$
                $$
                t_{peak} = -\frac{\mathbf{o} \cdot \mathbf{d}}{\mathbf{d} \cdot \mathbf{d}}
                $$
            *   Evaluate density at $\mathbf{p}_{peak} = \mathbf{r}_o + t_{peak} \mathbf{r}_d$.
            *   Compute `transmission = 1.0 - min(opacity * density, 0.9999)` [S1, unofficial]. Clipping to `0.9999` avoids numerical issues with `1.0 - 1.0` [S1, unofficial].
        *   **Local Contribution**:
            *   Compute emitted radiance: $L_e = \beta \cdot (1.0 - \text{transmission}) \cdot \text{emission}$ [S1, unofficial].
            *   Clamp infinite $L_e$ to 0: `Le[~dr.isfinite(Le)] = 0.0` [S1, unofficial].
            *   Update radiance accumulator:
                $$
                L[\text{active}] = L[\text{active}] + L_e \quad (\text{primal})
                $$
                $$
                L[\text{active}] = L[\text{active}] - L_e \quad (\text{adjoint})
                $$
            *   Update path throughput: $\beta[\text{active}] *= \text{transmission}$ [S1, unofficial].

4.  **Indirect Illumination (Adjoint/Path Tracing)**:
    *   In adjoint mode, compute differentiable reflected indirect radiance:
        $$
        L_{r\_ind} = L \cdot \text{transmission} / \text{dr.detach}(\text{transmission})
        $$
    *   Compute total local contribution: $L_o = L_e + L_{r\_ind}$.
    *   Select finite values: $L_o = \text{dr.select}(\text{active} \& \text{dr.isfinite}(L_o), L_o, 0.0)$ [S1, unofficial].
    *   Propagate gradients:
        *   If Backward: `dr.backward_from(δL * Lo)`
        *   If Forward: `δL += dr.forward_to(Lo)` [S1, unofficial].

5.  **Path Continuation**:
    *   Spawn new ray origin to avoid self-intersection: $\mathbf{r}_o[\text{active}] = \mathbf{p} + \mathbf{r}_d \cdot 10^{-4}$ [S1, unofficial].
    *   Update active mask: `active &= si.is_valid()`.
    *   Increment depth: `depth[active] += 1`.
    *   Kill paths with insignificant contribution:
        $$
        \beta_{max} = \text{dr.max}(\beta)
        $$
        $$
        \text{active} \&= (\beta_{max} > 0.01)
        $$
    *   Russian Roulette (if `primal` and `use_rr`):
        *   Calculate `rr_prob = dr.maximum(β_max, 0.1)`.
        *   Identify candidates: `rr_active = (depth >= rr_depth) & (β_max < 0.1)`.
        *   Scale throughput: `β[rr_active] *= dr.rcp(rr_prob)`.
        *   Determine continuation: `rr_continue = sample_rr < rr_prob`.
        *   Update `active` with `rr_continue`.
    *   Limit recursion depth: `active &= depth < max_depth` [S1, unofficial].

6.  **Finalization**:
    *   If `srgb_primitives` is true, convert $L$ to linear space: $L = \text{mi.math.srgb\_to\_linear}(L)$ [S1, unofficial].
    *   Return $L$ if primal, else $\delta L$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_volprim_ray(ray_o, ray_d, primitives, params):
    """
    Integrates a single ray through volumetric primitives.
    Returns (L, dL) where L is radiance and dL is gradient.
    """
    max_depth = params['max_depth']
    if max_depth == -1: max_depth = 0xFFFFFFFF
    rr_depth = params['rr_depth']
    if rr_depth == -1: rr_depth = 0xFFFFFFFF
    use_rr = rr_depth >= 0 and (rr_depth < max_depth or max_depth == -1)
    kernel_type = params['kernel_type']
    srgb = params['srgb_primitives']

    L = 0.0
    dL = 0.0
    beta = 1.0
    depth = 0
    active = True

    while active:
        # 1. Intersect with scene
        # scene.ray_intersect returns shape_info (si)
        si = scene.ray_intersect(ray_o, ray_d, coherent=(depth==0), flags=All | BackfaceCulling)
        
        # 2. Filter for ellipsoids
        is_ellipsoid = si.is_valid() and (si.shape.shape_type() == Ellipsoids)
        active = active & is_ellipsoid
        if not active:
            break

        # 3. Evaluate Primitive
        # Gather primitive data: center, rot, scale, opacity, sh_coeffs
        center = si.shape.center
        rot = si.shape.rotation
        scale = si.shape.scale
        opacity = si.shape.opacity
        sh_coeffs = si.shape.sh_coeffs
        
        # Transform ray to primitive local space
        # o_local = R^T * (ray_o - center) / scale
        # d_local = R^T * ray_d / scale
        o_local = rot.T @ (ray_o - center) / scale
        d_local = rot.T @ ray_d / scale
        
        # Compute peak location t_peak
        # t_peak = -dot(o_local, d_local) / dot(d_local, d_local)
        dot_od = dot(o_local, d_local)
        dot_dd = dot(d_local, d_local)
        t_peak = -dot_od / dot_dd
        
        # Evaluate density at p_peak
        p_peak = ray_o + t_peak * ray_d
        density = evaluate_kernel(p_peak, center, rot, scale, kernel_type)
        
        # Compute transmission
        # transmission = 1.0 - min(opacity * density, 0.9999)
        trans_val = min(opacity * density, 0.9999)
        transmission = 1.0 - trans_val
        
        # Evaluate Emission (SH)
        sh_degree = sqrt(len(sh_coeffs) / 3 - 1)
        sh_dir_coef = sh_eval(ray_d, sh_degree)
        emission = max(dot(sh_coeffs, sh_dir_coef) + 0.5, 0.0)
        
        # Compute Local Contribution Le
        # Le = beta * (1.0 - transmission) * emission
        Le = beta * (1.0 - transmission) * emission
        
        # Clamp infinite Le
        if not is_finite(Le):
            Le = 0.0
            
        # Update Radiance Accumulator
        L = L + Le
        
        # Update Throughput
        beta = beta * transmission
        
        # Indirect Illumination (Simplified for single bounce/path tracing context)
        # In full path tracing, this would involve sampling the BRDF/BSDF of the medium
        # or reflecting the ray. Here, we assume the 'transmission' term handles the
        # attenuation of indirect light if L contained indirect components.
        # For pure emission + absorption (Tomography), Lr_ind is 0.
        # For PRB (Scattering), Lr_ind is computed.
        # Lr_ind = L * transmission / detach(transmission)
        # Lo = Le + Lr_ind
        Lo = Le # Simplified for emission-only or assuming L is direct
        
        # Propagate Gradients (Adjoint)
        # dL = dL + Lo (Forward) or backward_from(dL * Lo)
        dL = dL + Lo
        
        # Spawn new ray origin
        ray_o = si.p + ray_d * 1e-4
        
        # Update Depth and Active Mask
        depth = depth + 1
        active = si.is_valid()
        
        # Kill insignificant paths
        beta_max = max(beta)
        active = active & (beta_max > 0.01)
        
        # Russian Roulette
        if use_rr and active:
            rr_prob = max(beta_max, 0.1)
            rr_active = (depth >= rr_depth) & (beta_max < 0.1)
            if rr_active:
                beta = beta / rr_prob
                rr_continue = sample_uniform() < rr_prob
                active = active & rr_continue
        
        # Limit Depth
        active = active & (depth < max_depth)

    # Final SRGB conversion
    if srgb:
        L = srgb_to_linear(L)
        
    return L, dL
```

### Edge Cases and Pitfalls
*   **Numerical Stability**: Transmission calculation clips `opacity * density` to `0.9999` to prevent `1.0 - 1.0` resulting in `0.0` due to floating-point precision errors, which would incorrectly terminate the ray [S1, unofficial].
*   **SH Emission**: Emission calculation adds `0.5` before clamping to `0.0` to handle potential negative values from Spherical Harmonic evaluation [S1, unofficial].
*   **Infinite Values**: Infinite $L_e$ values are explicitly clamped to `0.0` to prevent NaN propagation [S1, unofficial].

## Subtopics

### [Volumetric Primitive Tomography Integrator](2542-volumetric-primitive-tomography-integrator.md)
The Volumetric Primitive Tomography Integrator is a simplified rendering method for volumetric primitives that accounts exclusively for absorption, ignoring scattering and emission. It is designed for tomography applications where the goal is to visualize density or attenuation rather than photometric appearance. Unlike its sibling [Volumetric Primitive PRB Integrator](2543-volumetric-primitive-prb-integrator.md), which handles scattering media, this integrator treats the volume as a purely absorbing medium.

### [Volumetric Primitive PRB Integrator](2543-volumetric-primitive-prb-integrator.md)
The Volumetric Primitive PRB Integrator is a path tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation. It distinguishes itself from its sibling, the Volumetric Primitive Tomography Integrator, by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering.

### Epanechnikov Kernel Volumetric Primitive
The Epanechnikov kernel variant uses a closed-form solution for transmittance [S2, unofficial]. The 3D Epanechnikov kernel is defined as:
$$
E_i(\mathbf{x}) = \begin{cases} \frac{15}{8\pi} (\frac{7}{3} |\Sigma_i| )^{1/2} [1 - \frac{1}{7} d(\mathbf{x})] & \text{if } d(\mathbf{x}) \le 1 \\ 0 & \text{otherwise} \end{cases}
$$
where $d(\mathbf{x}) = (\mathbf{x} - \mathbf{c}_i)^T \Sigma_i^{-1} (\mathbf{x} - \mathbf{c}_i)$ [S2, unofficial]. Although an analytical solution for transmittance exists, free-flight distance sampling typically requires numerical root-finding methods (e.g., Newton-Raphson) due to complexity [S2, unofficial]. The Epanechnikov kernel has finite support, generating more compact primitive shells than Gaussian, enabling more efficient acceleration structures and faster rendering, especially suitable for modeling sharp edges [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Rendering speed depends on the number of primitives intersected and the path depth. The Epanechnikov kernel is faster than the Gaussian kernel due to its finite support and compact shells [S2, unofficial].
*   **Memory Complexity**: Memory usage is proportional to the number of primitives and their SH coefficients. This is generally more compact than dense voxel grids for high-frequency details.
*   **Performance**: Benchmarks show faster rendering speeds compared to Instant-NGP and other ray-traced radiance field solutions [S2, unofficial].
*   **Strengths**: Physically based rendering (PBR) integration, support for global illumination, soft shadows, and view-dependent emission via SH. Flexible camera models (e.g., 360-degree, telecentric) [S2, unofficial].
*   **Limitations**: Complex free-flight distance sampling for overlapping primitives requires numerical root-finding [S2, unofficial]. Quality vs. performance trade-off controlled by `max_depth` [S2, unofficial].

## Background
The method originates from the `volprim` library by Meta Platforms, Inc., released under the MIT license [S1, unofficial]. It is designed to bridge the gap between implicit neural models (like NeRF) and explicit voxel-based methods (like Plenoxels) by offering a physically based, kernel-based representation suitable for forward and inverse rendering in engines like Mitsuba [S2, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
