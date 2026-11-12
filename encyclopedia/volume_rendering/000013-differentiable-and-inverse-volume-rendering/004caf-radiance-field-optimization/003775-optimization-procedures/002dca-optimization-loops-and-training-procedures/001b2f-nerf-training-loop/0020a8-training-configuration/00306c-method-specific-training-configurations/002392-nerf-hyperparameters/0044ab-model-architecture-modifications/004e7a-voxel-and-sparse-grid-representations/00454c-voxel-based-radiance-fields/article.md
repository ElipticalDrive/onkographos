# Voxel-based Radiance Fields

Also called: Voxel NeRF, Voxel Radiance Fields

## Overview
Voxel-based Radiance Fields replace the multi-layer perceptron (MLP) of standard Neural Radiance Fields with structured voxel grids to store volume density and spherical harmonic (SH) coefficients, enabling real-time volume rendering by eliminating heavy feed-forward neural computations [S1]. This approach differs from MLP-based siblings like [Simple-NeRF](7371-simple-nerf.md) by trading the continuous, high-capacity function approximation of neural networks for a discrete, memory-efficient grid representation that allows direct interpolation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D position $\mathbf{X}_k$ and a viewing direction $\mathbf{d}$ for a sample point $k$ along a ray.
*   **Output:** Volume density $\sigma_k$ and view-dependent color $\mathbf{c}_k$.

**Data Structures**
The method utilizes two distinct voxel grids:
1.  **Density Grid ($G_\sigma$):** A 3D grid mapping $\mathbb{R}^3 \rightarrow \mathbb{R}$. It stores scalar volume density values.
2.  **Color Grid ($G_{sh}$):** A 3D grid mapping $\mathbb{R}^3 \rightarrow \mathbb{R}^{9 \times 3}$. It stores the coefficients for degree-2 spherical harmonics for RGB color. Each voxel contains $9 \times 3 = 27$ floating-point values (9 SH coefficients per RGB channel).

**Method Steps**
1.  **Density Interpolation:** For a query point $\mathbf{X}_k$, identify the nearest 8 voxels in $G_\sigma$. Perform trilinear interpolation to obtain the scalar density $\sigma_k$ [S1].
2.  **SH Coefficient Interpolation:** For the same query point $\mathbf{X}_k$, identify the nearest 8 voxels in $G_{sh}$. Perform trilinear interpolation to obtain a vector of 27 coefficients $\mathbf{v}_k \in \mathbb{R}^{27}$ [S1].
3.  **Direction Mapping:** Map the viewing direction $\mathbf{d}$ to a 9-dimensional SH basis vector $S(\mathbf{d}) \in \mathbb{R}^9$ using the standard degree-2 spherical harmonic functions [S1].
4.  **Color Computation:** Compute the color $\mathbf{c}_k$ by dotting the SH basis vector with the interpolated coefficients, reshaped appropriately for RGB channels [S1].

**Formulas**
The volume density is computed as:
$$
\sigma_k = \mathcal{G}_\sigma(\mathbf{X}_k)
$$
where $\mathcal{G}_\sigma$ denotes trilinear interpolation from the nearest 8 voxels of $G_\sigma$ [S1].

The color is computed as:
$$
\mathbf{c}_k = S(\mathbf{d})^\intercal \cdot \mathcal{G}_{sh}(\mathbf{X}_k)
$$
where:
*   $\mathbf{c}_k \in \mathbb{R}^3$ is the resulting color.
*   $S(\mathbf{d}): \mathbb{R}^3 \rightarrow \mathbb{R}^9$ is the spherical harmonic function mapping the direction to coefficients [S1].
*   $\mathcal{G}_{sh}(\mathbf{X}_k)$ represents the trilinearly interpolated SH coefficients from $G_{sh}$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_radiance_field(X_k, d, G_sigma, G_sh):
    """
    Compute density and color for a sample point using voxel grids.
    
    Args:
        X_k: 3D position (float32[3])
        d: Viewing direction (float32[3])
        G_sigma: Density grid (float32[D, D, D])
        G_sh: SH coefficient grid (float32[D, D, D, 27])
        
    Returns:
        sigma: Volume density (float32)
        c: Color (float32[3])
    """
    # 1. Interpolate density from G_sigma
    # Trilinear interpolation from the 8 nearest neighbors of X_k in G_sigma
    sigma = trilinear_interpolate(G_sigma, X_k)
    
    # 2. Interpolate SH coefficients from G_sh
    # Trilinear interpolation from the 8 nearest neighbors of X_k in G_sh
    # Result is a vector of 27 coefficients [S1]
    sh_coeffs = trilinear_interpolate(G_sh, X_k) # shape: [27]
    
    # 3. Compute SH basis vector for direction d
    # Maps direction to 9 degree-2 SH coefficients [S1]
    sh_basis = spherical_harmonic_basis(d, degree=2) # shape: [9]
    
    # 4. Compute color
    # The SH coefficients are stored as [9 coeffs * 3 channels]
    # Reshape to [3, 9] to dot with [9] basis vector
    sh_coeffs_rgb = sh_coeffs.reshape(3, 9)
    
    # Color = Basis^T * Coeffs for each channel
    c = sh_basis @ sh_coeffs_rgb.T # shape: [3]
    
    return sigma, c
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions:** Trilinear interpolation requires valid neighbors. Points outside the grid bounds must be clamped or handled with a default value (e.g., zero density) [G].
*   **Precision:** Storing SH coefficients in a fixed grid may limit the fidelity of high-frequency view-dependent effects compared to MLPs, though degree-2 SH is sufficient for many applications [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is significantly faster than MLP-based NeRF because it replaces expensive fully-connected layer evaluations with simple memory lookups and trilinear interpolation [S1]. This enables real-time volume rendering [S1].
*   **Memory Complexity:** Memory usage is determined by the resolution of the voxel grids. It is more memory-efficient than high-capacity MLPs for large scenes because parameters are limited to grid values [S1].
*   **Strengths:** Real-time rendering capability; memory efficiency; compatibility with coarse-to-fine sparse voxel reconstruction and pruning strategies [S1].
*   **Limitations:** Discrete representation may suffer from aliasing or lower fidelity compared to continuous MLPs for complex geometries; resolution is fixed by grid size [S1].
*   **When to use:** Applications requiring real-time rendering or efficient training on large-scale scenes, such as blurred view reconstruction [S1].

## Background
Voxel-based Radiance Fields were introduced as a memory-efficient alternative to the original NeRF [29] to address the computational bottleneck of MLP-based radiance fields [S1]. The method leverages differentiable volume rendering while replacing the neural network with voxel grids for density and spherical harmonic coefficients [S1]. It has been applied in contexts such as efficient radiance field optimization for extreme motion blurred image reconstruction (ExBluRF) [S1].

## Sources
[S1] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
