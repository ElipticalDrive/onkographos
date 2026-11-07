# Volumetric Rendering with Spherical Harmonics

Also called: Spherical Harmonics Rendering, Volume Rendering with Spherical Harmonics (Forward)

## Overview
Volumetric Rendering with Spherical Harmonics is a direct volume ray casting method that represents view-dependent color and opacity properties using spherical harmonic (SH) coefficients stored in a voxel grid [S1, S3, unofficial]. It solves the problem of efficiently rendering complex, view-dependent volumetric phenomena (such as anisotropic scattering or specular highlights in volumes) by converting the directional color function into a compact algebraic representation [S1, S3, unofficial]. It sets itself apart from standard direct volume rendering by replacing scalar-to-color transfer functions with SH basis evaluations, allowing for smooth, differentiable view-dependent shading without storing high-resolution directional textures [S1, S3, unofficial].

## How it works
The method implements the emission-absorption optical model, building on [Direct volume ray casting](19-direct-volume-ray-casting.md). It traces rays through a sparse grid of SH coefficients, computing the final pixel color and optionally the log-transmittance for background compositing [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Camera parameters: Origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel/ray.
*   Volume data: A sparse voxel grid storing SH coefficients and density (opacity) values. The grid supports 1st and 2nd degree SH bases [S3, unofficial].
*   Ray segment: Entry $t_{start}$ and exit $t_{end}$ parameters defining the intersection with the volume domain [S2, unofficial].
*   Configuration: Number of samples per ray $N$ (default 512 or 100) [S2, unofficial], and background compositing flag [S1, unofficial].

**Outputs:**
*   Final pixel color $C_{final}$.
*   Optional log-transmittance tensor for background integration [S1, unofficial].

### Data Structures
The volume is represented as a sparse grid. Each active voxel stores:
1.  **Density/Opacity Scalar:** A single float value $\sigma$ representing the extinction coefficient [S2, unofficial].
2.  **SH Coefficients:** A vector of coefficients $\mathbf{c}$ representing the directional color. For degree $L$, the number of coefficients is $(L+1)^2$. Degree 0 uses 1 coefficient, Degree 1 uses 4, and Degree 2 uses 9 [S3, unofficial].
3.  **Grid Metadata:** Indices for sparse storage (e.g., octree or hash map) to determine active voxels [S1, S3, unofficial].

### Steps

1.  **Ray Initialization:** For each ray, calculate the segment $[t_{start}, t_{end}]$ within the volume bounding box [S2, unofficial]. Generate $N$ equidistant sample points $\mathbf{p}_i$ along the ray [S2, unofficial].
    $$ \mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}, \quad t_i = t_{start} + \frac{i}{N-1}(t_{end} - t_{start}) $$
    *Note: Some implementations use a normalized parameter range $[0.05, 0.95]$ to avoid boundary artifacts [S2, unofficial].*

2.  **Interpolation:** At each sample point $\mathbf{p}_i$, perform trilinear interpolation on the voxel grid to retrieve the local density $\sigma_i$ and the SH coefficient vector $\mathbf{c}_i$ [S2, unofficial].
    *   Extract opacity: $\alpha_{raw} = \sigma_i$ [S2, unofficial].
    *   Extract SH coefficients: $\mathbf{c}_i = \text{Interpolate}(\text{Grid}, \mathbf{p}_i)$ [S2, unofficial].

3.  **SH Basis Evaluation:** Evaluate the spherical harmonic basis functions $\mathbf{Y}(\mathbf{d})$ for the ray direction $\mathbf{d}$ [S2, unofficial].
    $$ \mathbf{Y}(\mathbf{d}) = [Y_0(\mathbf{d}), Y_1(\mathbf{d}), \dots, Y_{L^2-1}(\mathbf{d})]^T $$
    Compute the harmonic projection:
    $$ \mathbf{h}_i = \mathbf{c}_i \odot \mathbf{Y}(\mathbf{d}) $$
    where $\odot$ denotes element-wise multiplication [S2, unofficial].

4.  **Color Computation:** Sum the harmonics to obtain the view-dependent color $\mathbf{C}_i$ for the sample [S2, unofficial].
    $$ \mathbf{C}_i = \sum_{k=0}^{L^2-1} \mathbf{h}_{i,k} $$
    *Note: In some implementations, this sum is performed over the axis corresponding to the SH coefficients [S2, unofficial].*

5.  **Transmittance and Opacity Calculation:** Compute the optical depth and transmittance for each sample interval [S2, unofficial].
    *   Calculate step sizes: $\Delta t_i = t_{i+1} - t_i$ [S2, unofficial].
    *   Compute weighted deltas (optical depth increments): $\delta_i = \Delta t_i \cdot \sigma_i$ [S2, unofficial].
    *   Compute cumulative transmittance $T_i$ from front to back:
        $$ T_0 = 1 $$
        $$ T_{i+1} = T_i \cdot e^{-\delta_i} $$
        Alternatively, using cumulative sums of negative deltas:
        $$ \text{cum\_deltas} = \text{cumsum}(-\delta) $$
        $$ T_i = e^{\text{cum\_deltas}_i} $$
    *   Compute per-sample opacity contribution: $\alpha_i = 1 - T_{i+1}/T_i = 1 - e^{-\delta_i}$ [S2, unofficial].

6.  **Compositing:** Integrate the color contributions front-to-back using the standard over operator [S2, unofficial].
    $$ C_{final} = \sum_{i=0}^{N-1} T_i \cdot \alpha_i \cdot \mathbf{C}_i $$
    *Note: The source snippet uses `sum(T * (1-T) * samples_colors)`, which implies a specific formulation where $T$ represents the transmittance *before* the sample and $(1-T)$ represents the opacity *of* the sample, or a simplified approximation. The standard formulation is $C_{final} = \sum T_i \alpha_i \mathbf{C}_i$ [S2, S12].*

7.  **Background Compositing (Optional):** If background rendering is enabled, compute the log-transmittance for the entire ray and combine with the background color [S1, unofficial].
    *   Allocate a 1D tensor for log-transmittance [S1, unofficial].
    *   Compute total transmittance $T_{total} = T_N$ [S1, unofficial].
    *   Log-transmittance: $L = \log(T_{total})$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volumetric_sh(rays_origin, rays_direction, voxel_grid, num_samples=512, use_background=False):
    """
    Renders volumetric data using Spherical Harmonics.
    
    Args:
        rays_origin: Tensor of shape (R, 3), ray origins.
        rays_direction: Tensor of shape (R, 3), ray directions (normalized).
        voxel_grid: Sparse grid object supporting trilinear interpolation of density and SH coeffs.
        num_samples: Number of samples per ray.
        use_background: Boolean flag for background compositing.
        
    Returns:
        colors: Tensor of shape (R, 3), final pixel colors.
        log_transmittance: Tensor of shape (R,), optional log transmittance.
    """
    R = rays_origin.shape[0]
    colors = torch.zeros(R, 3)
    log_transmittance = None
    
    if use_background:
        log_transmittance = torch.zeros(R)

    # 1. Generate sample points along rays
    # t ranges from 0 to 1 (normalized within the ray segment defined by t_start/t_end)
    # In practice, t_start and t_end are computed via ray-box intersection
    t_values = torch.linspace(0.05, 0.95, num_samples) # Using snippet default range [S2, unofficial]
    # Expand dims for broadcasting: (num_samples, 1)
    t_values = t_values.unsqueeze(1) 
    
    # Sample points in world space: (num_samples, 1) * (R, 3) -> (R, num_samples, 3)
    # Note: rays_origin and rays_direction are (R, 3)
    # We need to handle the segment [t_start, t_end] explicitly if not normalized to [0,1]
    # Assuming rays_direction is scaled to the segment length or t is absolute distance
    sample_points = rays_origin.unsqueeze(1) + rays_direction.unsqueeze(1) * t_values
    
    # 2. Interpolate Density and SH Coefficients
    # Interpolate returns: (R, num_samples, 1 + num_coeffs)
    # First channel is density (sigma), rest are SH coefficients
    interp_data = voxel_grid.trilinear_interpolate(sample_points)
    
    # Extract opacity (density) and SH coefficients
    # interp_opacities: (R, num_samples, 1)
    interp_opacities = interp_data[:, :, :1]
    # interp_sh_coeffs: (R, num_samples, num_coeffs)
    interp_sh_coeffs = interp_data[:, :, 1:]
    
    # 3. Compute SH Basis for Ray Directions
    # Evaluate SH basis functions Y(d) for each ray direction
    # my_sh_in: (R, num_coeffs)
    my_sh_in = evaluate_spherical_harmonics(rays_direction)
    
    # 4. Compute SH Harmonics and Colors
    # Element-wise multiplication: (R, num_samples, num_coeffs) * (R, 1, num_coeffs)
    # Broadcast my_sh_in to match sample dimension
    my_sh_in_expanded = my_sh_in.unsqueeze(1) # (R, 1, num_coeffs)
    interp_harmonics = interp_sh_coeffs * my_sh_in_expanded # (R, num_samples, num_coeffs)
    
    # Sum over SH coefficients to get color per sample
    # samples_colors: (R, num_samples, 3)
    samples_colors = torch.sum(interp_harmonics, dim=2)
    
    # 5. Compute Transmittance and Opacity
    # Calculate deltas (step sizes)
    # t_values is (num_samples, 1), so deltas is (num_samples-1, 1)
    deltas = t_values[1:, 0] - t_values[:-1, 0] # (num_samples-1,)
    deltas = deltas.unsqueeze(0).unsqueeze(2) # (1, num_samples-1, 1)
    
    # Weighted deltas (optical depth)
    # interp_opacities[:-1]: (R, num_samples-1, 1)
    deltas_times_sigmas = -deltas * interp_opacities[:, :-1, :] # (R, num_samples-1, 1)
    
    # Cumulative sum for transmittance
    # cum_weighted_deltas: (R, num_samples-1, 1)
    cum_weighted_deltas = torch.cumsum(deltas_times_sigmas, dim=1)
    
    # Prepend 0 for the first transmittance (T_0 = 1, log(T_0) = 0)
    # But for exp(cumsum), we need to align with samples
    # T_i = exp(cumsum_{j=0}^{i-1} delta_j)
    # So T_1 = exp(0) = 1, T_2 = exp(delta_0)
    # We compute T for each sample interval
    tmp1 = torch.exp(cum_weighted_deltas) # (R, num_samples-1, 1)
    
    # Per-sample transmittance T_i (before sample i)
    # T_0 = 1
    # T_1 = exp(-sigma_0 * delta_0)
    # We need T for all N samples. 
    # Let's compute T_i for i=0..N-1
    # T_0 = 1
    # T_i = exp(sum_{j=0}^{i-1} -sigma_j * delta_j)
    
    # Re-calculate T array of size (R, num_samples, 1)
    T = torch.ones(R, num_samples, 1)
    if num_samples > 1:
        # Cumulative transmittance up to sample i
        # cum_weighted_deltas[i] is sum from 0 to i-1
        T[:, 1:, :] = tmp1 # (R, num_samples-1, 1) -> (R, num_samples-1, 1)
        
    # Opacity of each sample: alpha_i = 1 - T_{i+1}/T_i = 1 - exp(-sigma_i * delta_i)
    # For the last sample, opacity is effectively 1 - T_N / T_{N-1} but usually we just use the delta
    # alpha_i = 1 - exp(deltas_times_sigmas_i)
    # deltas_times_sigmas is size (R, num_samples-1, 1)
    alpha = 1.0 - torch.exp(deltas_times_sigmas) # (R, num_samples-1, 1)
    
    # 6. Composite
    # Color contribution: T_i * alpha_i * color_i
    # T is (R, num_samples, 1), alpha is (R, num_samples-1, 1), colors is (R, num_samples, 3)
    # We need to align alpha with colors. 
    # Usually, alpha_i corresponds to the interval i, contributing to the color at that depth.
    # Contribution_i = T_i * (1 - exp(-sigma_i * delta_i)) * color_i
    
    # Recalculate alpha for all samples including the last one?
    # The snippet uses `interp_opacities[:-1]` for deltas.
    # Let's assume the last sample contributes with opacity 1 - T_N/T_{N-1} or similar.
    # Standard DVR: C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * C_i)
    
    # Re-compute alpha for all samples
    # sigma_i * delta_i
    sigma_delta = -deltas_times_sigmas # (R, num_samples-1, 1)
    # Add a dummy last delta? Or just use the first N-1 samples for integration?
    # The snippet sums over `samples_colors` which has N samples.
    # Let's assume alpha is computed for all N samples, with the last one having a small delta or just 1-T_N.
    
    # Simpler approach from snippet:
    # tmp1 = exp(cum_weighted_deltas) -> T_i for i=1..N-1
    # tmp2 = 1 - exp(deltas_times_sigmas) -> alpha_i for i=0..N-2
    # rays_color = sum(T * (1-T) * samples_colors)
    # This implies T in the snippet might be the transmittance *after* the sample?
    # Let's stick to the standard formula derived from the snippet's variables.
    
    # T_i (transmittance BEFORE sample i)
    T_before = torch.cat([torch.ones(R, 1, 1), tmp1], dim=1) # (R, num_samples, 1)
    
    # alpha_i (opacity OF sample i)
    # For i=0..N-2, alpha_i = 1 - exp(deltas_times_sigmas_i)
    # For i=N-1, alpha_{N-1} = 1 - T_before[N-1]/T_before[N-2]? Or just 1?
    # Let's assume alpha is computed for the first N-1 samples, and the last sample is handled by T_N.
    # Actually, standard integration: C = sum_{i=0}^{N-1} T_i * alpha_i * C_i
    # where alpha_i = 1 - exp(-sigma_i * delta_i)
    
    alpha_all = torch.cat([1.0 - torch.exp(deltas_times_sigmas), torch.ones(R, 1, 1)], dim=1) # (R, num_samples, 1)
    # Note: The last alpha is arbitrary if T_N is the final transmittance. 
    # Usually, the last sample's contribution is T_{N-1} * (1 - T_N/T_{N-1}) * C_{N-1} = (T_{N-1} - T_N) * C_{N-1}
    
    # Contribution
    contributions = T_before * alpha_all * samples_colors # (R, num_samples, 3)
    
    # Sum over samples
    colors = torch.sum(contributions, dim=1) # (R, 3)
    
    # 7. Background Compositing
    if use_background:
        # T_N is the transmittance after the last sample
        T_final = T_before[:, -1:, :] # (R, 1, 1)
        # Log transmittance
        log_transmittance = torch.log(T_final + 1e-8) # (R, 1)
        # Combine with background (not implemented here as background color is external)
        
    return colors, log_transmittance

def evaluate_spherical_harmonics(directions):
    """
    Evaluates SH basis functions for given directions.
    Returns coefficients for degree L.
    """
    # Implementation depends on degree L.
    # For L=0: [1]
    # For L=1: [0.282095, 0.488603*z, 0.488603*x, 0.488603*y] (normalized)
    # For L=2: [0.74312, 1.09254*y, -1.09254*x, 0.31539*(3z^2-1), -1.09254*xz, 0.54627*xy, ...]
    # This is a placeholder for the actual SH basis evaluation [S2, unofficial]
    pass
```

### Parameters
*   **SH Degree ($L$):** Determines the number of coefficients per voxel. Degree 0 (isotropic), 1 (dipole), 2 (quadrupole) are supported [S3, unofficial].
*   **Samples per Ray ($N$):** Default 512 or 100 [S2, unofficial].
*   **CUDA Threads:** `TRACE_RAY_CUDA_THREADS` for ray tracing kernel, `TRACE_RAY_BG_CUDA_THREADS` for background kernel [S1, unofficial].
*   **Learning Rate:** For SH coefficient optimization, `lr_basis` (default $10^{-6}$) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Splat Count Limits:** High SH degrees reduce the maximum number of splats/voxels that can be rendered due to memory constraints. Degree 0 allows ~16M, Degree 1 ~11M, Degree 2 ~8M [S3, unofficial].
*   **Boundary Artifacts:** Using sample ranges $[0.05, 0.95]$ helps avoid interpolation artifacts at the exact volume boundaries [S2, unofficial].
*   **Precision:** Trilinear interpolation of SH coefficients requires careful handling of the basis functions to maintain view-dependent continuity [S2, unofficial].

## Subtopics

### [Spherical Harmonics](5603-spherical-harmonics.md)
A family of techniques representing directional functions on the unit sphere using orthogonal basis functions, trading high-frequency accuracy for computational speed.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
A technique for rendering semi-transparent volumes with dynamic lighting by precomputing the scene's response to illumination on a surface mesh.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
A method for computing diffuse illumination in Fourier Volume Rendering by approximating shading functions with SH to allow interactive lighting changes.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Computes gradients for SH coefficients and ray parameters for optimizing volumetric representations.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R \cdot L^2)$, where $N$ is samples per ray, $R$ is number of rays, and $L^2$ is the number of SH coefficients [S2, unofficial].
*   **Memory Complexity:** $O(V \cdot L^2)$, where $V$ is the number of active voxels [S1, unofficial].
*   **Performance:** Supports 1st and 2nd degree SH for view-dependent effects [S3, unofficial]. Max splat counts are limited by SH degree (e.g., ~8M for Degree 2) [S3, unofficial].
*   **Strengths:** Efficient view-dependent shading, differentiable, suitable for optimization [S1, S3, unofficial].
*   **Limitations:** Limited by SH degree accuracy for high-frequency details; memory constraints on splat count for higher degrees [S3, unofficial].

## Background
The method originates from CUDA-based volume rendering implementations that use sparse grids represented by spherical harmonic coefficients [S1, unofficial]. It is used in neural volume rendering and Gaussian splatting contexts to handle view-dependent appearance [S1, S3, unofficial]. The approach allows for efficient rendering of complex volumetric data with dynamic lighting and view-dependent effects [S1, S3, unofficial].

## Sources
[S1] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
