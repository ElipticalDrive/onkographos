# Volumetric Rendering with Spherical Harmonics

Also called: Spherical Harmonics Rendering, Volume Rendering with Spherical Harmonics (Forward)

## Overview
Volumetric Rendering with Spherical Harmonics is a direct volume ray casting method that represents view-dependent color and opacity properties of a sparse voxel grid using spherical harmonic (SH) coefficients [S1, S3, unofficial]. It solves the problem of efficiently rendering complex, view-dependent volumetric scenes by replacing per-sample color lookups with algebraic evaluations of SH basis functions, enabling real-time performance on GPU hardware [S1, S3, unofficial]. It sets itself apart from standard direct volume rendering by encoding directional variation directly into the volume data structure, and from neural volume rendering by using a fixed, non-learnable basis for explicit control over frequency content and memory usage [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel grid storing density ($\sigma$) and spherical harmonic coefficients ($\mathbf{c}$) per voxel; camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$); number of samples per ray $N$; SH degree $L$ (typically 1 or 2) [S1, S2, S3, unofficial].
*   **Output:** Final pixel color $C_{final}$ and optionally log-transmittance for background compositing [S1, unofficial].

**Data Structures:**
The volume is stored in a sparse grid structure (e.g., octree or hash table) where each active voxel contains:
*   Density scalar $\sigma$ (1 float) [S2, unofficial].
*   SH coefficients $\mathbf{c}$: A vector of size $(L+1)^2$ for degree $L$ [S1, S3, unofficial]. For $L=0$, size is 1; for $L=1$, size is 4; for $L=2$, size is 9 [S3, unofficial].
*   The coefficients are typically stored in a contiguous tensor or interleaved with density in the voxel data structure [S2, unofficial].

**Algorithm Steps:**

1.  **Ray Generation and Discretization:**
    For each pixel, define ray origin $\mathbf{o}$ and direction $\mathbf{d}$. Intersect the ray with the volume bounding box to find $t_{start}$ and $t_{end}$ [S2, S5]. Discretize the segment into $N$ samples at positions $t_i$ for $i=0 \dots N-1$. Sample points in world space are computed as:
    $$ \mathbf{p}_i = \mathbf{o} + t_i \mathbf{d} $$
    Typical values for $N$ are 512 or 100 [S2, unofficial]. Sample positions $t_i$ are often distributed in the range $[0.05, 0.95]$ or $[0, 1]$ relative to the segment length [S2, unofficial].

2.  **Sampling and Interpolation:**
    For each sample point $\mathbf{p}_i$, retrieve the density $\sigma_i$ and SH coefficients $\mathbf{c}_i$ from the sparse grid using trilinear interpolation of the surrounding voxels [S1, S2, unofficial].
    Let $\mathbf{c}_i$ be the vector of SH coefficients at $\mathbf{p}_i$.

3.  **Spherical Harmonics Evaluation:**
    Compute the SH basis evaluation vector $\mathbf{b}(\mathbf{d})$ for the ray direction $\mathbf{d}$ [S2, unofficial]. The basis functions $Y_{l,m}(\theta, \phi)$ depend on the spherical coordinates of $\mathbf{d}$ [S1, unofficial].
    Compute the harmonic contribution for each sample:
    $$ \mathbf{h}_i = \mathbf{c}_i \odot \mathbf{b}(\mathbf{d}) $$
    where $\odot$ denotes element-wise multiplication [S2, unofficial].
    Compute the sample color $\mathbf{C}_i$ by summing the harmonics:
    $$ \mathbf{C}_i = \sum_{k} \mathbf{h}_{i,k} $$
    This results in a 3D color vector (RGB) per sample [S2, unofficial].

4.  **Compositing:**
    Compute the optical depth deltas $\Delta t_i = t_{i+1} - t_i$ for $i=0 \dots N-2$ [S2, unofficial].
    Compute the weighted deltas for transmittance using the interpolated opacity/density $\sigma_i$:
    $$ \Delta \tau_i = -\Delta t_i \cdot \sigma_i $$
    Compute the cumulative weighted deltas, prepended with 0:
    $$ \tau_i = \sum_{j=0}^{i-1} \Delta \tau_j, \quad \tau_0 = 0 $$
    Compute per-sample transmittance $T_i$ and opacity $\alpha_i$:
    $$ T_i = \exp(\tau_i) $$
    $$ \alpha_i = 1 - \exp(\Delta \tau_i) $$
    Composite the final color using front-to-back accumulation:
    $$ C_{final} = \sum_{i=0}^{N-1} T_i \cdot \alpha_i \cdot \mathbf{C}_i $$
    If background rendering is enabled, compute log-transmittance for compositing over a background image [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_basis(d, degree):
    """
    Compute SH basis vector for direction d (3D unit vector) and degree.
    Returns vector of size (degree+1)^2.
    Implementation details for specific SH basis functions are omitted [G].
    """
    # Placeholder for actual SH basis calculation
    # d is (3,), returns vector of shape (degree+1)^2
    pass

def render_volumetric_sh(rays_o, rays_d, grid, degree, num_samples=512):
    """
    Render volumetric scene using Spherical Harmonics.
    
    Args:
        rays_o: Ray origins, shape (num_rays, 3)
        rays_d: Ray directions, shape (num_rays, 3)
        grid: Sparse voxel grid object with method interpolate(point)
        degree: SH degree (0, 1, or 2)
        num_samples: Number of samples per ray
        
    Returns:
        colors: Final pixel colors, shape (num_rays, 3)
    """
    num_rays = rays_o.shape[0]
    colors = np.zeros((num_rays, 3))
    
    # Precompute SH basis for all ray directions
    # d is normalized
    rays_d_norm = rays_d / np.linalg.norm(rays_d, axis=1, keepdims=True)
    sh_basis = np.array([spherical_harmonics_basis(rays_d_norm[i], degree) for i in range(num_rays)])
    
    for i in range(num_rays):
        o = rays_o[i]
        d = rays_d_norm[i]
        
        # Ray intersection with bounding box [G]
        # Assume bbox_min and bbox_max are known
        t_min = np.max((bbox_min - o) / d)
        t_max = np.min((bbox_max - o) / d)
        
        if t_min > t_max:
            continue
            
        # Discretize ray
        t_vals = np.linspace(t_min, t_max, num_samples)
        points = o + t_vals[:, None] * d
        
        # Sample grid
        # grid.interpolate returns (num_samples, 1 + (degree+1)^2)
        # First column is density, rest are SH coefficients
        samples = grid.interpolate(points)
        densities = samples[:, 0]
        sh_coeffs = samples[:, 1:]
        
        # Compute SH harmonics
        # sh_coeffs shape: (num_samples, (degree+1)^2)
        # sh_basis[i] shape: ((degree+1)^2,)
        # Element-wise multiply and sum over SH dimensions
        harmonics = sh_coeffs * sh_basis[i]
        sample_colors = np.sum(harmonics, axis=1)
        
        # Compute transmittance and opacity
        deltas = np.diff(t_vals)
        deltas_times_sigmas = -deltas * densities[:-1]
        cum_weighted_deltas = np.cumsum(deltas_times_sigmas)
        cum_weighted_deltas = np.insert(cum_weighted_deltas, 0, 0)
        
        transmittance = np.exp(cum_weighted_deltas)
        opacity = 1 - np.exp(deltas_times_sigmas)
        
        # Composite
        # T_i * alpha_i * C_i
        contribution = transmittance[:-1] * opacity * sample_colors[:-1]
        colors[i] = np.sum(contribution, axis=0)
        
    return colors
```

**Parameters:**
*   `degree`: Spherical Harmonics degree. Degree 0 is view-independent color. Degree 1 and 2 add view-dependent effects [S3, unofficial].
*   `num_samples`: Number of samples per ray. Default 512 or 100 [S2, unofficial].
*   `cuda_n_threads`: CUDA threads per block for ray tracing [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **SH Degree Limits:** Higher SH degrees increase memory usage and reduce the maximum number of splats/voxels that can be rendered due to memory constraints. Degree 0 allows ~16M splats, Degree 1 ~11M, Degree 2 ~8M [S3, unofficial].
*   **Precision:** Floating-point precision in SH basis evaluation can affect color accuracy, especially for high frequencies [G].
*   **Boundary Conditions:** Ray intersection with the volume bounding box must be handled correctly to avoid sampling outside the grid [S2, unofficial].

## Subtopics

### [Spherical Harmonics](5603-spherical-harmonics.md)
General SH representation for directional functions.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
Precomputing radiance transfer for dynamic lighting.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
SH-based diffuse illumination in Fourier Volume Rendering.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Gradient computation for SH coefficients in differentiable rendering.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot D)$ per ray, where $N$ is the number of samples and $D=(L+1)^2$ is the number of SH coefficients [S1, S2, unofficial].
*   **Memory Complexity:** $O(V \cdot D)$ for the volume grid, where $V$ is the number of active voxels [S1, unofficial].
*   **Performance:** GPU-accelerated implementations achieve real-time frame rates for moderate resolutions and SH degrees [S1, S3, unofficial].
*   **Strengths:** Efficient view-dependent shading; compact representation for low-frequency directional variation; differentiable [S1, S3, unofficial].
*   **Limitations:** High-frequency directional details are lost with low SH degrees; memory usage scales with $D^2$ [S3, unofficial].
*   **When to Use:** Real-time rendering of view-dependent volumetric effects; scenes with smooth directional variation.
*   **When Not to Use:** Scenes requiring high-frequency specular or complex view-dependent effects without high SH degrees.

## Background
Volumetric Rendering with Spherical Harmonics was introduced as part of the Svox2 framework by Yu et al. [S1, unofficial]. It extends traditional volume rendering by incorporating SH coefficients to model view-dependent color. The method has been adopted in various real-time rendering engines and libraries, including GaussianSplats3D, which supports up to degree 2 SH for splat rendering [S3, unofficial].

## Sources
[S1] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
