# DARBF-Splatting

Also called: DARB-Splatting, Decaying Anisotropic Radial Basis Functions Splatting

## Overview
DARBF-Splatting is a method for differentiable volume rendering that replaces the standard Gaussian reconstruction kernels of 3D Gaussian Splatting with a broader class of functions called Decaying Anisotropic Radial Basis Functions (DARBFs) [S1]. It solves the limitation of being restricted to the exponential family by allowing non-exponential kernels (e.g., half-cosine, inverse multiquadric) that offer faster training convergence and lower memory usage while maintaining comparable visual quality [S1]. Unlike standard 3DGS, which relies on the closed-form integration properties of Gaussians, DARBF-Splatting introduces a correction factor to approximate this efficiency for other kernel types [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D primitives (ellipsoids) defined by position $\mathbf{p}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $\alpha_i$, and color attributes $\mathbf{c}_i$ (typically spherical harmonics coefficients), initialized from a Structure-from-Motion (SfM) point cloud [S1].
*   **Output:** A rendered 2D image and gradients for optimizing the primitive parameters [S1].

**Data Structures:**
*   **3D Covariance Matrix ($\boldsymbol{\Sigma}$):** A $3 \times 3$ symmetric positive-definite matrix defining the anisotropic shape and orientation of the primitive in 3D space [S1].
*   **2D Covariance Matrix ($\boldsymbol{\Sigma}'$):** A $2 \times 2$ matrix used for rasterization. It is derived from the 3D covariance matrix by skipping the third row and column of the $3 \times 3$ matrix after transforming the 3D covariance into the camera coordinate system [S1].

**Core Concepts:**
1.  **Mahalanobis Distance:** The shape of the kernel is determined by the Mahalanobis distance $d_M$ from a point $\mathbf{x}$ to the center $\boldsymbol{\mu}$ with covariance $\boldsymbol{\Sigma}$ [S1]:
    $$d_M(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = [(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})]^{\frac{1}{2}}$$
2.  **DARBF Kernels:** These are non-negative, decaying functions of $d_M$. Examples include the half-cosine kernel, defined as $\cos \left (\frac {d\_M^2}{\xi }\right )$ for $d\_M^2 < \frac {\xi \pi }{2}$ where $\xi > 0$, and the inverse multiquadric (Inverse MQ) [S1]. These functions have finite support or steeper rolloffs than Gaussians, spanning a larger region with higher opacity for the same variance, which reduces the need for multiple splats [S1].
3.  **Correction Factor ($\psi$):** Since non-Gaussian kernels do not naturally preserve the closed-form integration shortcut used in 3DGS, a correction factor $\psi$ is applied. The sources indicate that a scalar $\psi$ precomputed from closed-form regression experiments is used for computational efficiency [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize 3D primitives at points from an SfM pipeline (e.g., COLMAP) with parameters $\mathbf{p}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{c}_i$ [S1].
2.  **Transformation:** Transform the 3D primitives into the camera coordinate system. Compute the 2D covariance matrix $\boldsymbol{\Sigma}'$ by taking the upper-left $2 \times 2$ submatrix of the transformed 3D covariance matrix [S1].
3.  **Projection:** Project the 3D DARBF primitive onto the 2D image plane. Calculate the footprint function (splat function) for each primitive using the 2D covariance and the chosen DARBF kernel function (e.g., half-cosine) [S1].
4.  **Correction:** Apply the scalar correction factor $\psi$ to the kernel values to approximate the Gaussian integration shortcut [S1].
5.  **Rasterization:** Sort primitives by depth and accumulate color via alpha blending using the corrected footprint functions [S1].
6.  **Optimization:** Compute the loss between the rendered image and the ground truth. Backpropagate gradients to update primitive parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mahalanobis_distance(x, mu, Sigma_inv):
    """
    Calculate the Mahalanobis distance.
    x: (N, 3) array of points
    mu: (3,) array of center
    Sigma_inv: (3, 3) inverse covariance matrix
    Returns: (N,) array of distances
    """
    diff = x - mu
    # (x - mu)^T * Sigma_inv * (x - mu)
    temp = np.dot(diff, Sigma_inv)
    sq_dist = np.sum(temp * diff, axis=1)
    return np.sqrt(sq_dist)

def half_cosine_kernel(d_m, xi):
    """
    Half-cosine DARBF kernel.
    d_m: Mahalanobis distance
    xi: width parameter
    Returns: kernel value
    """
    d_m_sq = d_m ** 2
    threshold = (xi * np.pi) / 2
    # cos(d_m^2 / xi) for d_m^2 < xi * pi / 2, else 0
    mask = d_m_sq < threshold
    kernel = np.zeros_like(d_m)
    kernel[mask] = np.cos(d_m_sq[mask] / xi)
    return kernel

def get_2d_covariance(Sigma_3d, view_matrix):
    """
    Extract 2D covariance from 3D covariance after view transformation.
    Sigma_3d: (3, 3) 3D covariance matrix
    view_matrix: (4, 4) view transformation matrix
    Returns: (2, 2) 2D covariance matrix
    """
    # Note: The source simplifies this to skipping the third row/col of the 
    # transformed 3D covariance. In practice, this involves projecting the 
    # covariance using the Jacobian of the projection. 
    # The source states: "2x2 covariance matrix ... obtained by skipping 
    # the third row and column of the 3x3 covariance matrix".
    # We assume Sigma_3d is already in camera space.
    return Sigma_3d[:2, :2]

def render_darb_splatting(primitives, image_size, xi, psi):
    """
    Render a scene using DARBF-Splatting.
    primitives: list of dicts with keys 'p', 'Sigma', 'alpha', 'color', 'SH_coeffs'
    image_size: (height, width)
    xi: kernel width parameter
    psi: scalar correction factor
    Returns: rendered image
    """
    image = np.zeros(image_size)
    
    # Sort primitives by depth (simplified)
    sorted_primitives = sorted(primitives, key=lambda k: k['p'][2])
    
    for prim in sorted_primitives:
        p_3d = prim['p']
        Sigma_3d = prim['Sigma']
        alpha = prim['alpha']
        color = prim['color']
        
        # Transform to camera space (simplified)
        # In a real implementation, this involves full projection matrix multiplication
        p_cam = transform_to_camera(p_3d) 
        Sigma_cam = transform_covariance(Sigma_3d)
        
        # Get 2D covariance
        Sigma_2d = get_2d_covariance(Sigma_cam, view_matrix)
        
        # Determine bounding box for the splat
        # For half-cosine, support is finite: d_m^2 < xi * pi / 2
        # This defines an elliptical region
        # Calculate bounds based on eigenvalues of Sigma_2d and xi
        # ... (Bounding box calculation omitted for brevity, but required for implementation)
        
        # Rasterize splat
        # For each pixel in bounding box:
        #   1. Calculate 2D point x relative to projected center
        #   2. Calculate Mahalanobis distance d_m using Sigma_2d
        #   3. Calculate kernel value k = half_cosine_kernel(d_m, xi)
        #   4. Apply correction factor: k_corrected = k * psi
        #   5. Accumulate color: image += k_corrected * alpha * color
        
        pass # Implementation detail depends on specific CUDA kernel or CPU loop
        
    return image
```

**Parameters:**
*   **$\xi$ (xi):** Width parameter for the half-cosine kernel. Typical value: Not given in the sources.
*   **$\psi$ (psi):** Scalar correction factor. Typical value: Precomputed from closed-form regression experiments [S1].

**Edge Cases and Pitfalls:**
*   **Unbounded Functions:** Functions that grow unbounded cause instability and are omitted in favor of decaying functions (DARBFs) [S1].
*   **Integration Cost:** Using functions outside the Gaussian family typically incurs costly integration processes. The correction factor $\psi$ is necessary to approximate the Gaussian's closed-form integration shortcut; without it, performance degrades [S1].
*   **Precision:** The half-cosine kernel has finite support ($d\_M^2 < \frac {\xi \pi }{2}$), which must be handled correctly to avoid artifacts at the boundaries [S1].

## Subtopics

### Half-Cosine Kernel
A DARBF variant with finite support. It achieves 34% faster training convergence and 15% lower memory usage compared to standard Gaussians, with comparable PSNR, SSIM, and LPIPS [S1]. Choose this when computational efficiency is prioritized.

### Inverse Multiquadric (Inverse MQ)
A DARBF variant with a wide-spread area under the curve. It spans a larger region with higher opacity for the same variance, reducing the need for multiple splats [S1].

### Radial Cosine
Another DARBF variant. It achieves comparable PSNR to 3DGS-updated but with slightly lower SSIM and LPIPS in some ablations [S1].

## Operating characteristics
*   **Time Complexity:** Training convergence is 34% faster for the half-cosine variant compared to standard 3DGS [S1].
*   **Memory Complexity:** Memory usage is reduced by 15% for the half-cosine variant compared to standard 3DGS [S1].
*   **Performance:** On Mip-NeRF 360 scenes, the half-cosine kernel achieves PSNR 27.04, SSIM 0.92, and LPIPS 27.06, comparable to 3DGS-updated (PSNR 27.45) [S1].
*   **Strengths:** Faster training, lower memory footprint, ability to use non-exponential kernels.
*   **Limitations:** Requires a correction factor $\psi$ to maintain efficiency; not all DARBFs perform equally well (e.g., radial cosine had lower SSIM/LPIPS in ablations) [S1].
*   **When to use:** When computational savings (training time and memory) are critical and slight trade-offs in metric scores are acceptable.
*   **When not to use:** When maximum possible PSNR/SSIM is required and computational resources are not constrained (standard 3DGS may be preferable).

## Background
DARBF-Splatting was introduced by Hashiru Pramuditha, Vinasirajan Viruthshaan, Vishagar Arunan, Saeedha Nazar, Sameera Ramasinghe, Simon Lucey, and Ranga Rodrigo from the University of Moratuwa and University of Adelaide [S1]. The paper was published in 2026 (WACV) [S1]. It generalizes the 3D Gaussian Splatting framework, which was introduced by Kerbl et al. [S1, Ref 11], by exploring reconstruction kernels beyond the exponential family. The method addresses the underexplored area of non-Gaussian splatting functions, which were previously avoided due to the computational cost of integration [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
