# DARBF-Splatting

Also called: DARB-Splatting, Decaying Anisotropic Radial Basis Functions Splatting

## Overview
DARBF-Splatting is a method for differentiable volume rendering that replaces the standard Gaussian kernel in 3D Gaussian Splatting with a broader class of non-exponential functions called Decaying Anisotropic Radial Basis Functions (DARBFs) [S1]. It solves the limitation of being restricted to the exponential family by allowing kernels such as half-cosine and inverse multiquadric, which span larger regions with higher opacity for the same variance, thereby reducing the number of primitives required [S1]. This approach achieves comparable visual quality to standard 3DGS while offering up to 34% faster training convergence and 15% lower memory usage [S1].

## How it works
DARBF-Splatting operates within the standard differentiable rasterization pipeline of 3D Gaussian Splatting [Parent](10145-representation-and-kernel-modifications.md). It modifies the kernel function $K(d_M)$ used to compute the opacity contribution of a primitive based on its Mahalanobis distance from a point in space [S1].

### Inputs and Outputs
*   **Inputs:**
    *   A set of 3D primitives, each defined by a mean position $\boldsymbol{\mu}$ and a $3 \times 3$ covariance matrix $\boldsymbol{\Sigma}$ [S1].
    *   Camera parameters (view and projection matrices) for the target view [S1].
    *   A selected DARBF kernel function $K(\cdot)$ [S1].
*   **Outputs:**
    *   A rendered 2D image $C$ [S1].
    *   Gradients for optimizing $\boldsymbol{\mu}$, $\boldsymbol{\Sigma}$, and other primitive attributes [S1].

### Data Structures
The primitive attributes are stored per Gaussian ellipsoid [S1]:
*   **Mean Position ($\boldsymbol{\mu}$):** A 3D vector $(x, y, z)$ [S1].
*   **Covariance Matrix ($\boldsymbol{\Sigma}$):** A $3 \times 3$ symmetric positive-definite matrix defining the anisotropic shape and orientation [S1].
*   **Kernel Parameters:** Parameters specific to the chosen DARBF (e.g., $\xi$ for half-cosine) [S1].

### The Mahalanobis Distance
The distance metric used by all DARBFs is the Mahalanobis distance $d_M$ from a point $\mathbf{x}$ to the center $\boldsymbol{\mu}$ with covariance $\boldsymbol{\Sigma}$ [S1]:

$$
d_M(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = [(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})]^{\frac{1}{2}}
$$

where $\boldsymbol{\Sigma}^{-1}$ is the inverse of the covariance matrix [S1].

### Kernel Functions
DARBFs are non-negative, decaying functions of $d_M$ [S1]. The standard Gaussian $e^{-d_M^2/2}$ is a subset of this family. Key variants include:

1.  **Half-Cosine:**
    Defined as $\cos \left( \frac{d_M^2}{\xi} \right)$ for $d_M^2 < \frac{\xi \pi}{2}$, and 0 otherwise, where $\xi > 0$ is a scaling parameter [S1]. This function has finite support [S1].
2.  **Inverse Multiquadric (Inverse MQ):**
    A function that spans a larger region with higher opacity compared to Gaussians for the same variance, exhibiting a sharp peak and wide spread [S1].
3.  **Modified Raised Cosine / Modular Sinc:**
    Other decaying ARBFs supported by the framework [S1].

Unbounded growing functions are omitted due to instability [S1].

### Projection and Correction Factor
A key challenge with non-Gaussian kernels is that they do not possess the closed-form integration property that allows 3D Gaussians to be projected to 2D by simply skipping the third row and column of the $3 \times 3$ covariance matrix to obtain the $2 \times 2$ 2D covariance matrix [S1].

To preserve computational efficiency, DARBF-Splatting introduces a correction factor $\psi$ [S1]. This factor approximates the closure property of Gaussians, allowing the use of the standard 2D projection shortcut [S1]. The authors evaluated three cases for $\psi$:
1.  No correction factor.
2.  A matrix $\psi$ predicted by a Multi-Layer Perceptron (MLP).
3.  A scalar $\psi$ precomputed from closed-form regression experiments [S1].

The scalar $\psi$ is selected for implementation as it is computationally efficient [S1]. The final opacity contribution of a splat is modulated by this factor to ensure accurate rendering quality comparable to 3DGS [S1].

### Rendering Steps
1.  **Initialization:** Place 3D DARBF primitives at 3D points from an SfM pipeline (e.g., COLMAP) [S1].
2.  **Transformation:** Transform the 3D mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ into the camera frame [S1].
3.  **Projection:** Compute the 2D covariance matrix $\boldsymbol{\Sigma}_{2D}$ by skipping the third row and column of the transformed $3 \times 3$ covariance matrix [S1].
4.  **Splatting:** For each pixel, identify overlapping primitives. Calculate the Mahalanobis distance $d_M$ for each primitive relative to the pixel center [S1].
5.  **Kernel Evaluation:** Evaluate the chosen DARBF kernel $K(d_M)$ [S1].
6.  **Correction:** Apply the scalar correction factor $\psi$ to the kernel output [S1].
7.  **Compositing:** Alpha composite the contributions using the standard equation:
    $$
    C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\alpha_i$ is the opacity derived from the corrected kernel evaluation [S1].
8.  **Optimization:** Backpropagate the pixel error to update primitive parameters using gradient descent [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mahalanobis_distance(x, mu, sigma_inv):
    """
    Calculate the Mahalanobis distance d_M.
    x: (3,) point in space
    mu: (3,) mean position
    sigma_inv: (3,3) inverse covariance matrix
    """
    diff = x - mu
    return np.sqrt(diff @ sigma_inv @ diff)

def half_cosine_kernel(d_m, xi):
    """
    Half-cosine DARBF kernel.
    d_m: Mahalanobis distance
    xi: scaling parameter
    """
    d_m_sq = d_m ** 2
    threshold = (xi * np.pi) / 2
    if d_m_sq < threshold:
        return np.cos(d_m_sq / xi)
    else:
        return 0.0

def inverse_mq_kernel(d_m, alpha=1.0):
    """
    Inverse Multiquadric DARBF kernel.
    d_m: Mahalanobis distance
    alpha: shape parameter
    """
    return 1.0 / np.sqrt(1.0 + alpha * (d_m ** 2))

def project_covariance_3d_to_2d(sigma_3d):
    """
    Approximate 2D covariance by skipping the third row/column.
    sigma_3d: (3,3) 3D covariance matrix in camera space
    """
    return sigma_3d[:2, :2]

def render_darb_splatting(primitives, camera, kernel_func, psi_scalar):
    """
    Render a frame using DARBF-Splatting.
    primitives: List of dicts with keys 'mu', 'sigma', 'color', 'kernel_params'
    camera: Object with 'view_matrix', 'proj_matrix', 'image_width', 'image_height'
    kernel_func: Function taking (d_m, params) -> opacity
    psi_scalar: Scalar correction factor
    """
    height = camera.image_height
    width = camera.image_width
    image = np.zeros((height, width, 3))
    
    # Sort primitives by depth for correct alpha compositing
    # Note: In practice, this involves projection to screen space first
    # to determine bounding boxes and sorting.
    # Here we assume a simplified sorted list for demonstration.
    sorted_primitives = sort_primitives_by_depth(primitives, camera)
    
    for y in range(height):
        for x in range(width):
            pixel_pos = np.array([x, y, 1.0]) # Homogeneous 2D coord
            
            # Inverse projection to get ray direction or point in camera space
            # For splatting, we typically evaluate the splat function at the pixel center
            # in the projected 2D plane.
            
            accumulated_color = np.zeros(3)
            accumulated_alpha = 0.0
            
            for prim in sorted_primitives:
                # 1. Transform 3D mean to camera space
                mu_cam = transform_to_camera_space(prim['mu'], camera.view_matrix)
                
                # 2. Transform covariance to camera space
                # Sigma_cam = J * Sigma_3D * J^T where J is the Jacobian of the projection
                # Simplified: Assume sigma_cam is pre-transformed or computed via standard 3DGS projection
                sigma_cam = transform_covariance(prim['sigma'], camera, mu_cam)
                
                # 3. Project to 2D covariance
                sigma_2d = project_covariance_3d_to_2d(sigma_cam)
                
                # 4. Compute Mahalanobis distance in 2D
                # Point in 2D camera space (assuming orthographic or simplified perspective for d_M calc)
                # In standard 3DGS, d_M is computed in 2D screen space using the 2D covariance
                point_2d = project_point_to_screen(mu_cam, camera)
                diff_2d = np.array([x, y]) - point_2d
                
                # Ensure sigma_2d is positive definite
                if not is_positive_definite(sigma_2d):
                    continue
                    
                sigma_2d_inv = np.linalg.inv(sigma_2d)
                d_m = mahalanobis_distance(diff_2d, np.zeros(2), sigma_2d_inv)
                
                # 5. Evaluate Kernel
                opacity = kernel_func(d_m, prim['kernel_params'])
                
                # 6. Apply Correction Factor
                opacity *= psi_scalar
                
                # 7. Alpha Compositing
                alpha = opacity * prim['opacity'] # Assuming primitive has base opacity
                if alpha > 0.0:
                    weight = alpha * (1.0 - accumulated_alpha)
                    accumulated_color += weight * prim['color']
                    accumulated_alpha += weight
                    if accumulated_alpha > 0.99:
                        break # Early exit optimization
            
            image[y, x] = accumulated_color
            
    return image
```

### Parameters and Typical Values
*   **$\xi$ (Half-Cosine):** Scaling parameter. Specific values depend on the scene scale and variance of the primitives [S1].
*   **$\psi$ (Correction Factor):** A scalar precomputed from closed-form regression experiments [S1].
*   **Variance:** The variance of the DARBF primitives should be tuned similarly to 3DGS, but due to the wider spread of kernels like half-cosine, fewer primitives may be needed [S1].

### Edge Cases and Pitfalls
*   **Unbounded Functions:** Functions that grow unbounded cause instability and are omitted [S1].
*   **Integration Cost:** Using functions outside the Gaussian family typically incurs costly integration processes. The correction factor $\psi$ is essential to avoid this [S1].
*   **Positive Definiteness:** The 2D covariance matrix must remain positive definite after projection. If not, the splat should be skipped or regularized [G].
*   **Finite Support:** Half-cosine has finite support ($d_M^2 < \frac{\xi \pi}{2}$). Points outside this range contribute zero opacity [S1].

## Operating characteristics
*   **Time Complexity:** Training convergence is up to 34% faster than standard 3DGS [S1].
*   **Memory Complexity:** Memory usage is reduced by up to 15% compared to standard 3DGS [S1].
*   **Performance:** Achieves comparable PSNR, SSIM, and LPIPS results to 3DGS [S1].
*   **Strengths:** Faster training, lower memory footprint, ability to use wider-spread kernels (half-cosine, inverse MQ) which reduce the need for multiple splats [S1].
*   **Limitations:** Requires a correction factor $\psi$ to approximate Gaussian integration properties; not all DARBFs may perform equally well [S1].
*   **When to use:** When computational efficiency (training time and memory) is critical and comparable visual quality to 3DGS is acceptable [S1].
*   **When not to use:** When the absolute highest fidelity of the exponential family Gaussian is required, or when the overhead of tuning $\psi$ is prohibitive [S1].

## Background
DARBF-Splatting was introduced by Hashiru Pramuditha, Vinasirajan Viruthshaan, Vishagar Arunan, Saeedha Nazar, Sameera Ramasinghe, Simon Lucey, and Ranga Rodrigo [S1]. It was published in 2026 in the IEEE Winter Conference on Applications of Computer Vision (WACV) [S1]. The work generalizes splatting techniques beyond the conventional exponential family, addressing the underexplored area of non-Gaussian reconstruction kernels in 3D Gaussian Splatting [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
