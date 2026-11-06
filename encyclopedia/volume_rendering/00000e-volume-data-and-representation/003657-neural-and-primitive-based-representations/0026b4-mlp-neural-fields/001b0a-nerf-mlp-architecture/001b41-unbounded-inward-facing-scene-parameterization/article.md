# Unbounded inward-facing scene parameterization

Also called: Contracted space parameterization, Geometric mean Jacobian contraction, Mip-NeRF 360 Scene Contraction, Mip-NeRF 360 contraction, Mip-NeRF 360 parameterization, Scene Contraction, Spatial Contraction for Unbounded Scenes

## Overview
This method parameterizes unbounded 360-degree scenes by mapping coordinates from $\mathbb{R}^3$ into a bounded domain (typically $[-2, 2]^3$ or a unit sphere), enabling the use of bounded acceleration structures like voxel grids for efficient rendering [S1][S2, S3, unofficial]. It allocates model capacity proportional to disparity (inverse distance), ensuring distant content is represented with lower resolution while preserving detail near the origin [S1]. Unlike standard NeRF fields that assume bounded volumes, this technique extends bounded data structures to unbounded scenes [S3, unofficial]. It differs from forward-facing parameterization by handling full spherical coverage rather than a viewing frustum [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D coordinates $\mathbf{x} \in \mathbb{R}^3$ [S1][S2, unofficial].
*   **Output:** Contracted coordinates $\mathbf{x}' \in [-2, 2]^3$ [S1].
*   **Optional Input:** Isotropic Gaussian scales $\sigma_j$ and means $\mathbf{x}_j$, requiring Jacobian-based scale contraction [S1].

**Data Structures:**
*   **Voxel Grid:** A cuboid grid centered at the origin [S2, unofficial].
    *   **Size:** Length $2 + 2b$ along each axis, where $b > 0$ is a hyperparameter [S2, unofficial].
    *   **Resolution:** Typically $320^3$ for inward-facing scenes [S2, unofficial].
    *   **Allocation:** Allocated after coordinate transformation to cover the contracted space [S2, unofficial].

**Preprocessing (Coordinate Alignment):**
Before applying the non-linear contraction, the world coordinates are preprocessed to optimize grid usage [S2, unofficial]:
1.  **Rotation:** Rotate the world coordinate system to align the first two Principal Component Analysis (PCA) directions of the camera positions with the grid's XY axes [S2, unofficial].
2.  **Translation:** Shift the coordinates so the origin aligns with the centroid of the camera positions [S2, unofficial].
3.  **Scaling:** Scale the coordinates such that all cameras lie near the planes of a unit sphere [S2, unofficial].

**Contraction Function:**
The core transformation maps unbounded space to a bounded volume. Two variants are cited in the sources:

1.  **Mip-NeRF 360 Variant (Standard):** Maps $\mathbb{R}^3$ to $[-2, 2]^3$ using the Euclidean norm [S1].
    $$
    \operatorname{contract}(\mathbf{x}) = \begin{cases} 
    \mathbf{x} & \|\mathbf{x}\| \leq 1 \\ 
    \left(2 - \frac{1}{\|\mathbf{x}\|}\right)\left(\frac{\mathbf{x}}{\|\mathbf{x}\|}\right) & \|\mathbf{x}\| > 1 
    \end{cases}
    $$
    Here, $\|\mathbf{x}\|$ denotes the L2 norm [S1].

2.  **DVGOv2 Variant (Generalized Norm):** Uses an arbitrary $L_p$ norm and a background hyperparameter $b > 0$ [S2, unofficial].
    $$
    \mathbf{x}' = \begin{cases} 
    \mathbf{x} & \|\mathbf{x}\|_p \leq 1 \\ 
    \left(1 + b - \frac{b}{\|\mathbf{x}\|_p}\right) \left(\frac{\mathbf{x}}{\|\mathbf{x}\|_p}\right) & \|\mathbf{x}\|_p > 1 
    \end{cases}
    $$
    *   $\|\mathbf{x}\|_p$: The $L_p$ norm of $\mathbf{x}$ [S2, unofficial].
    *   $b$: Hyperparameter controlling the proportion of voxel grid points allocated to the background [S2, unofficial].
    *   $p$: Norm order. $p=2$ creates a spherical contraction. $p=\infty$ creates a cuboid contraction [S2, unofficial].

**Gaussian Scale Contraction (for 3DGS variants):**
When contracting isotropic Gaussians, the mean $\mathbf{x}_j$ is contracted directly using $\operatorname{contract}(\mathbf{x}_j)$ [S1]. The scale $\sigma_j$ must be adjusted to account for the local distortion of the contraction map [S1]. This is done by computing the geometric mean of the eigenvalues of the Jacobian $J_C(\mathbf{x}_j)$ of the contraction function [S1].

For an isotropic Gaussian in $d=3$ dimensions, the contracted scale $\operatorname{contract}(\sigma_j)$ is:
$$
\operatorname{contract}(\sigma_j) = \sigma_j \left| \det(J_C(\mathbf{x}_j)) \right|^{1/3}
$$
This approach is preferred over the full multivariate Gaussian contraction from Mip-NeRF 360 because it is less computationally expensive and sufficiently expressive for isotropic scales [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def contract_coordinate(x, p=2, b=1.0):
    """
    Contracts unbounded 3D coordinates to a bounded space.
    
    Args:
        x: numpy array of shape (N, 3), coordinates in world space.
        p: Norm order (2 for Euclidean/sphere, inf for cuboid).
        b: Background hyperparameter. b=1.0 matches Mip-NeRF 360; b>0 matches DVGOv2.
        
    Returns:
        x_prime: Contracted coordinates.
    """
    # Calculate norm
    if p == np.inf:
        norm_x = np.max(np.abs(x), axis=1, keepdims=True)
    else:
        norm_x = np.linalg.norm(x, axis=1, keepdims=True, ord=p)
        
    # Avoid division by zero in unused branches
    norm_x_safe = np.maximum(norm_x, 1e-8)
    
    # Condition: ||x||_p <= 1
    inside = norm_x <= 1.0
    
    # For points inside, x' = x
    x_prime = np.where(inside, x, 0.0)
    
    # For points outside, apply formula
    factor = 1.0 + b - (b / norm_x_safe)
    direction = x / norm_x_safe
    
    x_prime = np.where(inside, x, factor * direction)
    
    return x_prime

def contract_gaussian_scale(x, sigma, p=2, b=1.0):
    """
    Contracts the scale of an isotropic Gaussian under the scene contraction.
    
    Args:
        x: numpy array of shape (N, 3), Gaussian means.
        sigma: numpy array of shape (N,), isotropic scales.
        p: Norm order.
        b: Background hyperparameter.
        
    Returns:
        contracted_sigma: Scales adjusted for the contraction Jacobian determinant.
    """
    r = np.linalg.norm(x, axis=1, keepdims=True)
    r_safe = np.maximum(r, 1e-8)
    
    outside = r > 1.0
    
    # Radial factor f(r)
    f_r = np.where(outside, 1.0 + b - (b / r_safe), r_safe)
    
    # Radial derivative f'(r)
    f_prime_r = np.where(outside, b / (r_safe ** 2), 1.0)
    
    d = 3
    lambda_tangential = f_r / r_safe
    lambda_radial = f_prime_r
    
    # Determinant = lambda_radial * (lambda_tangential)^(d-1)
    det_J = lambda_radial * (lambda_tangential ** (d - 1))
    
    # Geometric mean of eigenvalues = |det(J)|^(1/d)
    jacobian_factor = np.abs(det_J) ** (1.0 / d)
    
    contracted_sigma = sigma * jacobian_factor
    
    return contracted_sigma
```

**Parameters:**
*   **$d$**: Dimension of coordinates, typically 3 [S1].
*   **$b$**: Hyperparameter $> 0$ controlling the proportion of voxel grid points allocated to the background [S2, unofficial].
*   **$p$**: Norm order. $p=2$ creates a spherical contraction. $p=\infty$ creates a cuboid contraction [S2, unofficial].
*   **Grid Resolution**: $320^3$ for inward-facing scenes [S2, unofficial].
*   **Initial Alpha ($\alpha_{init}$)**: $10^{-4}$ [S2, unofficial].
*   **Voxel Step Size**: 0.5 (in contracted space) [S2, unofficial].
*   **TV Loss Weight (Density)**: $10^{-6}$ [S2, unofficial].
*   **TV Loss Weight (Feature)**: $10^{-7}$ [S2, unofficial].
*   **Distortion Loss Weight**: $10^{-2}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Norm Selection:** Using $p=2$ (Euclidean norm) results in a spherical contraction within a cubic grid, wasting approximately 50% of the grid points because a sphere occupies only a fraction of a cube [S2, unofficial]. Using $p=\infty$ creates a cuboid contracted space, utilizing the grid more efficiently [S2, unofficial].
*   **Boundedness Assumption:** Standard Occupancy Grids (e.g., from Instant-NGP) only support bounded scenes [S3, unofficial]. Scene contraction is required to extend these structures to unbounded scenes [S3, unofficial].
*   **Gaussian Contraction Complexity:** Using the full multivariate Gaussian contraction method from Mip-NeRF 360 is computationally expensive and overly expressive for isotropic Gaussians [S1]. The Jacobian geometric mean method is preferred [S1].

## Subtopics

### Forward-facing scene parameterization
A method to handle unbounded forward-facing scenes by warping the viewing frustum to a bounded volume using a dense voxel grid allocation similar to Multiplane Images (MPIs) [S2, unofficial]. It places $D$ RGB-density images at fixed depths, each with $X \times Z$ resolution [S2, unofficial]. Rays are traced from the first to the $D$-th images with a step size $s$, resulting in $2D - 1$ sampled points if $s=0.5$ [S2, unofficial]. Parameters include $D=256$ depth layers, voxel resolution $XZ=384^2$ per layer, and sampling step size $s=1.0$ layer [S2, unofficial]. For this forward-facing variant, TV loss weights are $10^{-5}$ (density) and $10^{-6}$ (feature), with a distortion loss weight of $10^{-2}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The contraction function itself is $O(1)$ per sample [G]. The primary cost comes from the voxel grid lookups and MLP queries, which are accelerated by the bounded grid structure [S2, S3, unofficial].
*   **Memory Complexity:** Depends on the voxel grid resolution. For a $320^3$ grid, memory usage is proportional to $320^3$ voxels [S2, unofficial].
*   **Performance:** On unbounded inward-facing scenes, this method enables efficient reconstruction using bounded acceleration structures [S3, unofficial]. DVGOv2 with this parameterization achieves comparable quality to other fast convergence approaches with less training time on low-spec GPUs [S2, unofficial].
*   **Strengths:** Enables the use of highly optimized bounded data structures (like Instant-NGP grids) for unbounded scenes [S3, unofficial]. Allocates capacity proportional to disparity, which is natural for perspective projection [S1].
*   **Limitations:** The choice of norm ($p$) affects grid efficiency. $p=2$ wastes grid space [S2, unofficial]. The contraction introduces non-linearities that must be handled carefully during gradient backpropagation (e.g., via Jacobian for Gaussians) [S1].
*   **When to use:** When rendering unbounded 360-degree scenes and using voxel-based acceleration structures [S3, unofficial].
*   **When not to use:** For forward-facing scenes where frustum-specific parameterization (MPI-like) is more efficient [S2, unofficial].

## Background
The scene contraction idea originates from Mip-NeRF 360 [S3, unofficial][S1][S2, unofficial], specifically Equation 10 in that paper [S3, unofficial]. It was adapted to support unbounded 360-degree scenes in Direct Voxel Grid Optimization (DVGOv2) [S2, unofficial] and integrated into acceleration toolboxes like NerfAcc to support unbounded scenes with Instant-NGP's occupancy grids [S3, unofficial]. The method addresses the limitation that standard neural field acceleration structures require bounded input domains [S3, unofficial].

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S2] Improved Direct Voxel Grid Optimization for Radiance Fields Reconstruction, https://doi.org/10.48550/arxiv.2206.05085, fetched 2026-10-26 (unofficial)
[S3] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
