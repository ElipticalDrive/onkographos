# Voxel-based Radiance Fields

Also called: Voxel NeRF, Voxel Radiance Fields

## Overview
Voxel-based Radiance Fields replace the multi-layer perceptron (MLP) in standard Neural Radiance Fields with discrete voxel grids to store volume density and spherical harmonic (SH) coefficients, enabling real-time volume rendering by eliminating heavy feed-forward neural network computations [S1]. Unlike MLP-based NeRF which requires evaluating fully-connected layers for every sample point, this method directly computes color from interpolated grid values, offering a memory-efficient framework that limits learnable parameters to the voxel grid values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D query position $\mathbf{X}_k$ and a viewing direction $\mathbf{d}$ for the $k$-th sample point on a ray.
*   **Output:** Volume density $\sigma_k$ and view-dependent color $\mathbf{c}_k$.

**Data Structures:**
Two dense voxel grids are defined over the 3D space:
1.  **Density Grid ($G_\sigma$):** Maps $\mathbb{R}^3 \rightarrow \mathbb{R}$. Stores scalar volume density values at voxel vertices.
2.  **SH Coefficient Grid ($G_{sh}$):** Maps $\mathbb{R}^3 \rightarrow \mathbb{R}^{9 \times 3}$. Stores degree-2 spherical harmonic coefficients for RGB color. Each voxel stores a vector of 27 values (9 SH coefficients $\times$ 3 color channels).

**Interpolation:**
For any query point $\mathbf{X}_k$, the features are retrieved via trilinear interpolation from the nearest 8 voxels. Let $\mathcal{G}_\sigma(\mathbf{X}_k)$ denote the interpolated density and $\mathcal{G}_{sh}(\mathbf{X}_k)$ denote the interpolated SH coefficient vector.

**Formulas:**
The volume density is computed directly from the density grid:
$$ \sigma_k = \mathcal{G}_\sigma(\mathbf{X}_k) $$

The color is computed by projecting the interpolated SH coefficients onto the viewing direction:
$$ \mathbf{c}_k = S(\mathbf{d})^\intercal \cdot \mathcal{G}_{sh}(\mathbf{X}_k) $$
Where:
*   $S(\mathbf{d}): \mathbb{R}^3 \rightarrow \mathbb{R}^9$ is the spherical harmonic function mapping the viewing direction $\mathbf{d}$ to a 9-dimensional basis vector.
*   $\mathcal{G}_{sh}(\mathbf{X}_k)$ is the $9 \times 3$ matrix of interpolated SH coefficients.
*   The dot product results in a 3D RGB color vector.

**Algorithm Steps:**
1.  **Query Density:** Perform trilinear interpolation on $G_\sigma$ using $\mathbf{X}_k$ to obtain $\sigma_k$.
2.  **Query SH Coefficients:** Perform trilinear interpolation on $G_{sh}$ using $\mathbf{X}_k$ to obtain the $9 \times 3$ coefficient matrix $\mathbf{C}_{sh}$.
3.  **Compute SH Basis:** Evaluate the degree-2 spherical harmonic basis functions for the viewing direction $\mathbf{d}$ to form the vector $\mathbf{s} = S(\mathbf{d}) \in \mathbb{R}^9$.
4.  **Compute Color:** Calculate $\mathbf{c}_k = \mathbf{s}^\intercal \mathbf{C}_{sh}$.
5.  **Volume Rendering:** Use $\sigma_k$ and $\mathbf{c}_k$ in the standard differentiable volume rendering integral (as defined in the parent NeRF framework) to compute the final pixel color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_radiance_field_query(X_k, d, G_sigma, G_sh):
    """
    Query voxel-based radiance field for density and color.
    
    Args:
        X_k: 3D query position (float, shape [3])
        d: Viewing direction (float, shape [3])
        G_sigma: Density grid, tensor of shape [D, D, D]
        G_sh: SH coefficient grid, tensor of shape [D, D, D, 27]
               (27 = 9 SH coefficients * 3 RGB channels)
               
    Returns:
        sigma: Volume density (float)
        c: RGB color (float, shape [3])
    """
    # 1. Trilinear interpolation for density
    # Assuming a standard trilinear interpolation function `trilinear`
    # which takes a grid and a normalized coordinate (0-1) or voxel index
    sigma = trilinear(G_sigma, X_k)
    
    # 2. Trilinear interpolation for SH coefficients
    # Extract the 27 values for the 3 color channels
    sh_coeffs_flat = trilinear(G_sh, X_k) # shape [27]
    
    # Reshape to [9, 3] (9 SH bases, 3 RGB channels)
    C_sh = sh_coeffs_flat.reshape(9, 3)
    
    # 3. Compute Spherical Harmonic basis for direction d
    # Degree 2 SH has 9 coefficients. 
    # S(d) maps direction to R^9.
    s = spherical_harmonic_basis(d, degree=2) # shape [9]
    
    # 4. Compute color: c = S(d)^T * C_sh
    # Matrix multiplication: (1, 9) @ (9, 3) -> (1, 3)
    c = s @ C_sh
    
    return sigma, c
```

**Parameters:**
*   **SH Degree:** 2 is used in the cited implementation, resulting in 9 basis functions per color channel [S1].
*   **Grid Resolution:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Trilinear interpolation requires handling coordinates outside the grid bounds. Standard practice is clamping or zero-padding, though not explicitly detailed in the source [S1].
*   **Precision:** The SH basis evaluation $S(\mathbf{d})$ must be numerically stable for all viewing directions.
*   **Memory:** While more efficient than MLPs, dense voxel grids scale cubically with resolution. The source mentions "coarse-to-fine sparse voxel reconstruction" as a strategy for memory-efficient training, implying that dense grids may be too large for high resolutions without sparsity techniques [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is significantly faster than MLP-based NeRF because it replaces expensive fully-connected layer evaluations with fast trilinear lookups and a small matrix-vector multiplication for SH projection [S1]. This enables real-time volume rendering.
*   **Memory Complexity:** Memory usage is proportional to the number of voxels. For a grid of size $N \times N \times N$, memory is $O(N^3)$ for the density grid and $O(27 N^3)$ for the SH grid. This is generally more memory-efficient than large MLPs for comparable spatial resolution, but less flexible than sparse representations for large scenes.
*   **Strengths:** Real-time rendering capability; memory-efficient parameter storage; differentiable nature allows for gradient-based optimization [S1].
*   **Limitations:** Fixed spatial resolution determined by grid size; aliasing artifacts at high frequencies if grid resolution is insufficient; less expressive than deep MLPs for complex geometry without very high resolution grids.
*   **When to use:** Applications requiring real-time rendering or efficient training of radiance fields where scene complexity fits within the voxel grid resolution.
*   **When not to use:** Scenes with extremely high-frequency details that exceed the voxel grid's Nyquist limit, or very large open-world scenes where sparse representations are more appropriate.

## Background
Voxel-based Radiance Fields were introduced as a memory-efficient alternative to the original Neural Radiance Fields (NeRF) [29] by replacing the MLP network with voxel grids [S1]. This approach leverages the differentiable volume rendering of NeRF while eliminating the heavy computation of fully-connected layers [S1]. The specific implementation cited, ExBluRF, utilizes this representation for efficient radiance field optimization in the context of extreme motion blurred image reconstruction [S1]. The method builds on the principle of decoupled prediction, where spatial features are stored in a grid and combined with view directions via SH coefficients [S1].

## Sources
[S1] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
