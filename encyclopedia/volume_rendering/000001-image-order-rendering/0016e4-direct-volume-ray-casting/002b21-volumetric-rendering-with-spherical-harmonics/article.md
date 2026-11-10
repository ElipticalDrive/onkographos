# Volumetric Rendering with Spherical Harmonics

Also called: Spherical Harmonics Rendering, Volume Rendering with Spherical Harmonics (Forward)

## Overview
Volumetric Rendering with Spherical Harmonics is a direct volume ray casting method that represents view-dependent color and opacity data using spherical harmonic (SH) coefficients stored in a voxel grid [S1, S3, unofficial]. It solves the problem of expensive per-sample shading and transfer function evaluation by encoding the directional dependence of radiance directly into the volume data, allowing color to be computed via algebraic dot products during compositing [S1, S3, unofficial]. It sets itself apart from standard direct volume rendering by replacing the scalar-to-color transfer function lookup with a continuous SH basis evaluation, and from [Neural volume rendering](1576-neural-volume-rendering.md) by using a fixed, non-learnable basis for directional representation rather than a neural network [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A sparse voxel grid where each occupied voxel stores a scalar density (or opacity coefficient) and a vector of spherical harmonic coefficients [S1, S2, unofficial]. Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$) and ray bounds $[t_{near}, t_{far}]$ [S2, unofficial].
- **Output:** Final pixel color $\mathbf{C}_{out}$ and optionally accumulated log-transmittance for background compositing [S1, unofficial].

**Data Structures:**
- **Voxel Grid:** A sparse 3D grid. Each active voxel contains:
    - `density`: Scalar value representing extinction coefficient or opacity proxy [S2, unofficial].
    - `sh_coeffs`: Array of $K$ floats, where $K$ depends on the SH degree $L$ ($K = (L+1)^2$) [S1, S3, unofficial].
- **Ray Data:** For $Q$ rays, arrays of origin $\mathbf{o}_i$, direction $\mathbf{d}_i$, and sample positions [S2, unofficial].

**Numbered Steps:**

1.  **Ray Initialization:** For each pixel/ray $i$, define the ray segment $[t_{near}, t_{far}]$ by intersecting the ray with the volume bounding box [S2, unofficial]. Discretize the segment into $N$ samples (default $N=512$ or $100$) at positions $t_j = t_{near} + j \cdot \Delta t$, where $\Delta t = (t_{far} - t_{near}) / N$ [S2, unofficial]. Sample positions are often parameterized in $[0.05, 0.95]$ to avoid boundary artifacts [S2, unofficial].

2.  **Voxel Interpolation:** At each sample position $\mathbf{p}_j$, perform trilinear interpolation on the voxel grid to retrieve the local density $\sigma_j$ and the SH coefficient vector $\mathbf{c}_j$ [S1, S2, unofficial].
    - Extract opacity proxy: $\alpha_{raw, j} = \text{interp}(\text{density})$ [S2, unofficial].
    - Extract SH coefficients: $\mathbf{c}_j = \text{interp}(\text{sh\_coeffs})$ [S2, unofficial].

3.  **SH Basis Evaluation:** Compute the SH basis functions $\mathbf{Y}(\mathbf{d}_i)$ for the viewing direction $\mathbf{d}_i$ of the ray [S2, unofficial]. The basis evaluation depends on the chosen degree $L$ (typically 1 or 2) [S3, unofficial].
    - Let $\mathbf{y}_i = \mathbf{Y}(\mathbf{d}_i)$ be the vector of basis values [S2, unofficial].

4.  **Color Computation:** Compute the view-dependent color $\mathbf{C}_j$ at sample $j$ by taking the dot product of the interpolated SH coefficients and the basis vector [S2, unofficial]:
    $$
    \mathbf{C}_j = \mathbf{c}_j \cdot \mathbf{y}_i
    $$
    Note: In some implementations, the SH coefficients are stored per-sample and multiplied by the basis before summation [S2, unofficial].

5.  **Transmittance and Opacity Calculation:** Compute the optical depth and transmittance for each sample [S2, unofficial].
    - Compute step sizes: $\Delta t_j = t_{j+1} - t_j$ (for $j=0 \dots N-2$) [S2, unofficial].
    - Compute weighted optical depth deltas: $\delta_j = \sigma_j \cdot \Delta t_j$ [S2, unofficial].
    - Compute cumulative transmittance $T_j$ from front to back:
        $$
        T_0 = 1
        $$
        $$
        T_{j+1} = T_j \cdot e^{-\delta_j}
        $$
        Alternatively, using cumulative sum of negative deltas:
        $$
        \text{cum\_delta}_j = \sum_{k=0}^{j-1} -\delta_k
        $$
        $$
        T_j = e^{\text{cum\_delta}_j}
        $$
    - Compute per-sample opacity contribution: $\alpha_j = 1 - e^{-\delta_j}$ [S2, unofficial].

6.  **Front-to-Back Compositing:** Composite the samples into the final color [S2, unofficial]:
    $$
    \mathbf{C}_{out} = \sum_{j=0}^{N-1} \mathbf{C}_j \cdot T_j \cdot \alpha_j
    $$
    $$
    \alpha_{out} = \sum_{j=0}^{N-1} T_j \cdot \alpha_j
    $$
    Or in iterative form:
    $$
    \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + T_j \cdot \mathbf{C}_j \cdot \alpha_j
    $$
    $$
    T_{j+1} \leftarrow T_j \cdot (1 - \alpha_j)
    $$

7.  **Background Compositing (Optional):** If background rendering is enabled, compute the log-transmittance for the ray and composite the background color using the accumulated transmittance [S1, unofficial].
    - Log-transmittance: $\log T_{final} = \sum_{j=0}^{N-1} -\delta_j$ [S1, unofficial].
    - Composite with background color $\mathbf{C}_{bg}$:
        $$
        \mathbf{C}_{final} = \mathbf{C}_{out} + T_{final} \cdot \mathbf{C}_{bg}
        $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_basis(direction, degree):
    """
    Computes the spherical harmonic basis vector for a given direction.
    direction: (3,) array, normalized viewing direction.
    degree: int, SH degree (0, 1, or 2).
    Returns: (K,) array of basis values, where K = (degree + 1)**2.
    """
    x, y, z = direction
    basis = []
    
    # Degree 0
    basis.append(0.28209479177387814) # 1 / sqrt(4*pi)
    
    if degree >= 1:
        # Degree 1
        basis.append(-0.4886025119029199 * y)
        basis.append(0.4886025119029199 * z)
        basis.append(-0.4886025119029199 * x)
        
    if degree >= 2:
        # Degree 2
        basis.append(1.0925484305920792 * x * y)
        basis.append(-1.0925484305920792 * y * z)
        basis.append(0.31539156525252005 * (3 * z**2 - 1))
        basis.append(-1.0925484305920792 * x * z)
        basis.append(0.5462742152960396 * (x**2 - y**2))
        
    return np.array(basis)

def render_volumetric_sh(voxel_grid, ray_origins, ray_directions, 
                         num_samples=512, sh_degree=1, 
                         background_enabled=False, background_colors=None):
    """
    Renders a volume using spherical harmonics for view-dependent color.
    
    Args:
        voxel_grid: Sparse grid object supporting trilinear interpolation.
                    Must support query(pos) -> (density, sh_coeffs).
        ray_origins: (Q, 3) array of ray origins.
        ray_directions: (Q, 3) array of ray directions (normalized).
        num_samples: int, number of samples per ray.
        sh_degree: int, degree of spherical harmonics (0, 1, 2).
        background_enabled: bool, whether to composite with background.
        background_colors: (Q, 3) array of background colors.
        
    Returns:
        colors: (Q, 3) array of final pixel colors.
        transmittances: (Q,) array of final transmittance (if background enabled).
    """
    Q = ray_origins.shape[0]
    K = (sh_degree + 1)**2
    
    # Precompute SH basis for all ray directions
    # Shape: (Q, K)
    ray_sh_bases = np.zeros((Q, K))
    for i in range(Q):
        ray_sh_bases[i] = spherical_harmonics_basis(ray_directions[i], sh_degree)
        
    colors = np.zeros((Q, 3))
    transmittances = np.ones(Q)
    
    # Determine ray bounds (simplified: assume unit sphere or pre-computed t_near/t_far)
    # In practice, intersect with AABB. Here we assume t_near=0, t_far=1 for simplicity
    # or that ray_directions are scaled to the volume extent.
    t_near = 0.0
    t_far = 1.0
    
    for i in range(Q):
        # Sample positions along the ray
        t_samples = np.linspace(t_near, t_far, num_samples + 1)
        # Sample points in world space
        sample_points = ray_origins[i] + t_samples[:, None] * ray_directions[i]
        
        # Interpolate density and SH coefficients at sample points
        # Note: In a real implementation, this is done in bulk or via GPU kernel
        # Here we simulate the interpolation result
        # interp_data shape: (num_samples + 1, 1 + K)
        # Column 0: density, Columns 1:K+1: sh_coeffs
        interp_data = voxel_grid.interpolate(sample_points)
        
        densities = interp_data[:, 0]
        sh_coeffs = interp_data[:, 1:]
        
        # Compute SH basis for the ray direction (constant for all samples on the ray)
        ray_basis = ray_sh_bases[i]
        
        # Compute color at each sample
        # Color = dot(sh_coeffs, ray_basis)
        # sh_coeffs shape: (num_samples + 1, K)
        # ray_basis shape: (K,)
        sample_colors = np.dot(sh_coeffs, ray_basis) # Shape: (num_samples + 1, 3)
        
        # Compute optical depth deltas
        # deltas = t_samples[1:] - t_samples[:-1]
        deltas = np.diff(t_samples)
        
        # Use densities as extinction coefficients
        # Note: densities might need scaling or clamping
        sigmas = np.maximum(densities[:-1], 1e-6)
        
        # Weighted deltas
        weighted_deltas = sigmas * deltas
        
        # Cumulative transmittance
        # T_j = exp(-sum_{k=0}^{j-1} weighted_deltas_k)
        cum_weighted_deltas = np.cumsum(weighted_deltas)
        T = np.exp(-np.concatenate(([0], cum_weighted_deltas)))
        
        # Per-sample opacity
        alpha = 1 - np.exp(-weighted_deltas)
        
        # Front-to-back compositing
        # C_out = sum(T_j * alpha_j * C_j)
        # T_j here is transmittance BEFORE sample j
        # alpha_j is opacity of sample j
        
        # Adjust T to be transmittance before each sample
        # T[0] is 1.0 (before first sample)
        # T[1] is transmittance after first sample
        
        contribution = T[:-1] * alpha * sample_colors[:-1]
        colors[i] = np.sum(contribution, axis=0)
        transmittances[i] = T[-1] # Transmittance after last sample
        
    if background_enabled and background_colors is not None:
        colors = colors + transmittances[:, None] * background_colors
        
    return colors, transmittances
```

**Parameters:**
- `num_samples`: Default 512 or 100 [S2, unofficial].
- `sh_degree`: Typically 1 or 2 [S3, unofficial].
- `cuda_n_threads`: Number of CUDA threads per block for ray tracing (macro `TRACE_RAY_CUDA_THREADS`) [S1, unofficial].
- `TRACE_RAY_BG_CUDA_THREADS`: Number of CUDA threads per block for background rendering [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Artifacts:** Sampling at exact boundaries can cause interpolation errors. Using parameterized ranges like $[0.05, 0.95]$ helps [S2, unofficial].
- **SH Degree Limits:** Higher SH degrees increase memory usage and reduce the maximum number of splats/voxels that can be rendered due to memory constraints (e.g., Degree 2 limits splat count to ~8,000,000 vs ~16,000,000 for Degree 0) [S3, unofficial].
- **Precision:** Floating-point precision in cumulative transmittance calculation can lead to artifacts in deep volumes. Using log-transmittance can mitigate this [S1, unofficial].

## Subtopics

### [Spherical Harmonics](5603-spherical-harmonics.md)
General SH representation for directional functions.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
Precomputes radiance transfer for dynamic lighting.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
SH-based shading in Fourier Volume Rendering.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Gradient computation for SH coefficients in ray tracing.

### [Zonal Method (Bhate and Tokuta)](12221-zonal-method-bhate-and-tokuta.md)
Deterministic global illumination for participating media with surface interactions.

## Operating characteristics
- **Time Complexity:** $O(Q \cdot N \cdot K)$, where $Q$ is the number of rays, $N$ is the number of samples per ray, and $K$ is the number of SH coefficients ($K=(L+1)^2$) [S1, S2, unofficial]. The SH basis evaluation is $O(K)$ per sample, but since the direction is constant per ray, it can be optimized to $O(K)$ per ray [S2, unofficial].
- **Memory Complexity:** $O(V \cdot K)$, where $V$ is the number of active voxels and $K$ is the number of SH coefficients per voxel [S1, S3, unofficial].
- **Measured Performance:** Limits on splat count depend on SH degree: Degree 0 allows ~16,000,000 splats, Degree 1 allows ~11,000,000, and Degree 2 allows ~8,000,000 [S3, unofficial].
- **Strengths:** Efficient view-dependent shading without per-sample transfer function lookups; compact representation of directional data [S1, S3, unofficial].
- **Limitations:** Memory intensive for high SH degrees; fixed basis may not capture complex directional effects as well as learned bases [S1, S3, unofficial].
- **When to Use:** When view-dependent color effects are important and computational resources allow for the memory overhead of SH coefficients [S3, unofficial].
- **When Not to Use:** When memory is constrained or view-dependent effects are minimal (use Degree 0 or standard scalar transfer functions) [S3, unofficial].

## Background
The method originates from CUDA-based volume rendering implementations that use sparse grids represented by spherical harmonic coefficients [S1, unofficial]. It is part of the broader family of spherical harmonic methods for directional data representation [S1, S3, unofficial]. The approach is used in neural volume rendering contexts and real-time rendering applications where view-dependent effects are desired [S1, S3, unofficial].

## Sources
[S1] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
