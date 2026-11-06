# Volumetric Rendering with Spherical Harmonics

Also called: Spherical Harmonics Rendering, Volume Rendering with Spherical Harmonics (Forward)

## Overview
Volumetric Rendering with Spherical Harmonics is a direct volume ray casting method that represents view-dependent color and opacity variations using spherical harmonic (SH) coefficients stored in a voxel grid [S1, S3, unofficial]. Unlike standard direct volume rendering which uses scalar fields for density and fixed transfer functions for color, this method encodes the angular dependence of radiance directly into the volume data, allowing for efficient view-dependent effects without per-ray lighting calculations [S1, S3, unofficial]. It sets itself apart from [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md) by integrating SH evaluation into the ray marching compositing loop rather than precomputing shading on slices, and from [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md) by handling the full volumetric integration at runtime rather than decoupling it into a surface-based transfer function [S1, S2, unofficial].

## How it works
The method implements the emission-absorption optical model, building on [Direct volume ray casting](19-direct-volume-ray-casting.md). It traces rays through a sparse or dense grid where each voxel stores SH coefficients for color and a scalar value for opacity/density.

### Inputs and Outputs
**Inputs:**
- Ray origins $\mathbf{o}$ and directions $\mathbf{d}$ for each pixel.
- A voxel grid containing:
  - Opacity/Density scalar field $\sigma(\mathbf{p})$.
  - Spherical Harmonic coefficients $C_{l,m}(\mathbf{p})$ for color, typically up to degree 2 [S3, unofficial].
- Rendering parameters: number of samples per ray $N$, ray start/end bounds $t_{start}, t_{end}$, and background compositing flag [S1, S2, unofficial].

**Outputs:**
- Final pixel color $C_{final}$.
- Optional log-transmittance for background compositing [S1, unofficial].

### Data Structures
The volume data is stored in a 3D grid. For each voxel at position $\mathbf{p}$, the data layout is:
- `density`: Float scalar representing local opacity/density $\sigma$.
- `sh_coeffs`: Array of SH coefficients. For degree $L$, there are $(L+1)^2$ complex or real coefficients per color channel (RGB). In practice, implementations often store real-valued coefficients for R, G, and B separately [S1, S3, unofficial].
- `valid`: Boolean or sparse index indicating if the voxel is active (in sparse grids) [S1, unofficial].

### Numbered Steps

1. **Ray Setup**: For each pixel, define ray $r(t) = \mathbf{o} + t\mathbf{d}$. Intersect with the volume bounding box to find $t_{start}$ and $t_{end}$ [S1, S2, unofficial].
2. **Sampling**: Discretize the segment $[t_{start}, t_{end}]$ into $N$ equidistant samples. Sample positions are $t_i = t_{start} + i \cdot \Delta t$ for $i=0 \dots N-1$, where $\Delta t = (t_{end} - t_{start}) / N$ [S2, unofficial].
3. **Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, perform trilinear interpolation on the voxel grid to extract:
   - Opacity $\alpha_i$ (or density $\sigma_i$).
   - SH coefficients $\mathbf{c}_i$ for the ray direction [S2, unofficial].
4. **SH Basis Evaluation**: Compute the SH basis functions $Y_{l,m}(\mathbf{d})$ for the ray direction $\mathbf{d}$ [S2, unofficial].
5. **Color Computation**: Compute the color contribution $C_i$ at each sample by projecting the SH coefficients onto the basis:
   $$ C_i = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_{i,l,m} \cdot Y_{l,m}(\mathbf{d}) $$
   This is implemented as a dot product between the coefficient vector and the basis vector [S2, unofficial].
6. **Transmittance and Compositing**: Perform front-to-back compositing using the standard over operator [S2, unofficial]:
   - Compute optical depth deltas: $\Delta \tau_i = \sigma_i \cdot \Delta t$.
   - Compute transmittance: $T_i = \exp(-\sum_{j=0}^{i-1} \Delta \tau_j)$.
   - Compute weight: $w_i = T_i \cdot (1 - \exp(-\Delta \tau_i))$.
   - Accumulate color: $C_{final} = \sum_{i=0}^{N-1} w_i C_i$.
   - Accumulate opacity: $\alpha_{final} = 1 - \exp(-\sum_{i=0}^{N-1} \Delta \tau_i)$.
7. **Background Compositing (Optional)**: If enabled, combine $C_{final}$ and $\alpha_{final}$ with the background color $C_{bg}$ using the over operator: $C_{out} = C_{final} + (1 - \alpha_{final}) C_{bg}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_basis(direction, degree):
    """
    Computes the real spherical harmonic basis functions for a given direction and degree.
    Returns a 1D array of size (degree + 1)**2.
    """
    # Implementation of real SH basis functions Y_l^m(theta, phi)
    # direction is a 3D unit vector (x, y, z)
    # Returns coefficients for l=0..degree, m=-l..l
    # Note: Exact formulas depend on normalization convention (e.g., orthonormal vs. unnormalized)
    # Standard computer graphics often use unnormalized real SHs.
    pass

def render_volumetric_sh(rays_origins, rays_directions, voxel_grid, degree=2, num_samples=512):
    """
    Renders a volume using spherical harmonics for view-dependent color.
    
    Args:
        rays_origins: (N, 3) array of ray origins.
        rays_directions: (N, 3) array of ray directions (normalized).
        voxel_grid: Object providing trilinear interpolation for 'density' and 'sh_coeffs'.
        degree: SH degree (0, 1, or 2).
        num_samples: Number of samples per ray.
        
    Returns:
        colors: (N, 3) array of final pixel colors.
        transmittances: (N,) array of final transmittance (1 - alpha).
    """
    N = rays_origins.shape[0]
    colors = np.zeros((N, 3))
    transmittances = np.ones(N)
    
    # Precompute SH basis for all ray directions
    # Shape: (N, (degree+1)**2)
    sh_bases = np.array([spherical_harmonics_basis(rays_directions[i], degree) for i in range(N)])
    
    for i in range(N):
        o = rays_origins[i]
        d = rays_directions[i]
        
        # Determine t_start and t_end by intersecting ray with volume bbox
        t_start, t_end = intersect_ray_bbox(o, d, voxel_grid.bbox)
        
        if t_start >= t_end:
            continue
            
        # Sample points
        t_values = np.linspace(t_start, t_end, num_samples, endpoint=False)
        sample_points = o + t_values[:, None] * d
        
        # Interpolate density and SH coefficients
        # interp_density: (num_samples,)
        # interp_sh_coeffs: (num_samples, (degree+1)**2 * 3) for RGB
        interp_density, interp_sh_coeffs = voxel_grid.interpolate(sample_points)
        
        # Reshape SH coefficients to (num_samples, 3, (degree+1)**2)
        # Assuming coefficients are stored as R, G, B blocks
        interp_sh_coeffs = interp_sh_coeffs.reshape(num_samples, 3, (degree + 1)**2)
        
        # Compute SH colors for each sample
        # sh_bases: (N, (degree+1)**2) -> for current ray i, sh_bases[i]
        # We need to broadcast: (num_samples, 3, (degree+1)**2) dot ( (degree+1)**2, )
        # Result: (num_samples, 3)
        sample_colors = np.einsum('ijk, k -> ij', interp_sh_coeffs, sh_bases[i])
        
        # Compute optical depth and transmittance
        # Delta t is constant
        dt = (t_end - t_start) / num_samples
        
        # Density to opacity/density sigma
        sigma = interp_density
        
        # Cumulative optical depth
        # tau_i = sum(sigma_j * dt for j < i)
        # Transmittance T_i = exp(-tau_i)
        optical_depths = np.cumsum(sigma * dt)
        transmittances_ray = np.exp(-optical_depths)
        
        # Weight for each sample: T_i * (1 - exp(-sigma_i * dt))
        # Note: The last sample's transmittance is not used for weighting in some formulations,
        # but standard front-to-back uses T_i for the contribution of sample i.
        # w_i = T_i * (1 - exp(-sigma_i * dt))
        weights = transmittances_ray * (1 - np.exp(-sigma * dt))
        
        # Accumulate color
        # C_final = sum(w_i * C_i)
        colors[i] = np.sum(weights[:, None] * sample_colors, axis=0)
        
        # Final transmittance for this ray
        transmittances[i] = np.exp(-np.sum(sigma * dt))
        
    return colors, transmittances
```

### Parameters and Edge Cases
- **SH Degree**: Degree 0 is isotropic (view-independent). Degree 1 and 2 add view-dependent effects [S3, unofficial]. Higher degrees increase memory and computation cost [S3, unofficial].
- **Sample Count**: Typical values are 512 or 100 samples per ray [S2, unofficial].
- **Precision**: Floating-point precision issues may arise in cumulative transmittance calculations for deep volumes. Log-transmittance is used in some implementations to avoid underflow [S1, unofficial].
- **Sparse Grids**: If the grid is sparse, rays may pass through empty space. Early termination or skipping empty voxels is required for performance [S1, unofficial].

## Subtopics

### [Spherical Harmonics](5603-spherical-harmonics.md)
The mathematical basis used to represent directional functions. Fixed basis, no optimization needed.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
Decouples volume integration from rendering by precomputing transfer on a surface mesh.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
Uses SH for diffuse illumination in Fourier Volume Rendering, handling nonlinearity via convolution.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Computes gradients for SH coefficients and ray parameters for differentiable rendering.

## Operating characteristics
- **Time Complexity**: $O(N \cdot S \cdot L^2)$ per ray, where $N$ is number of rays, $S$ is samples per ray, and $L$ is SH degree. SH evaluation is $O(L^2)$ per sample [S1, S2, unofficial].
- **Memory Complexity**: $O(V \cdot L^2)$ for the volume grid, where $V$ is number of voxels. Degree 2 requires 9 coefficients per channel per voxel [S3, unofficial].
- **Performance**: Degree 2 limits max splat count to ~8,000,000 in some implementations due to memory bandwidth [S3, unofficial]. Trilinear interpolation is standard [S2, unofficial].
- **Strengths**: View-dependent effects without per-light calculations. Efficient for static scenes with complex lighting interactions encoded in SH.
- **Limitations**: High memory usage for high SH degrees. Not suitable for dynamic lighting unless SH coefficients are updated. Aliasing if sample count is low.

## Background
The method originates from the integration of spherical harmonics into volumetric rendering pipelines, notably in systems like Svox2 [S1, unofficial] and GaussianSplats3D [S3, unofficial]. It leverages the ability of SH to compactly represent directional data, allowing view-dependent colors to be stored directly in the volume. This approach is particularly useful for neural volume representations and splat-based rendering where view-dependent appearance is critical [S1, S3, unofficial].

## Sources
[S1] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
