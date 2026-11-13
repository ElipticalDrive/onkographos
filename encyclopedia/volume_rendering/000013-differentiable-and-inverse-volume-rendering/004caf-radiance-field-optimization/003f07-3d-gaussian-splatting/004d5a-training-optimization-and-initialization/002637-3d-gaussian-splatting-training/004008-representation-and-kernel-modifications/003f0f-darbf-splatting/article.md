# DARBF-Splatting

Also called: DARB-Splatting, Decaying Anisotropic Radial Basis Functions Splatting

## Overview
DARBF-Splatting is a method for differentiable volume rendering that replaces the standard Gaussian reconstruction kernels of 3D Gaussian Splatting with a broader class of functions called Decaying Anisotropic Radial Basis Functions (DARBFs) [S1]. It solves the limitation of existing splatting methods being restricted to the exponential family, offering improved training convergence speed and reduced memory usage while maintaining comparable visual quality [S1]. Unlike standard 3DGS, which relies on the closed-form integration properties of Gaussians, DARBF-Splatting introduces a correction factor to approximate this efficiency for non-exponential kernels [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D scene points (typically from an SfM pipeline like COLMAP), camera poses, and intrinsics. The method retains the standard 3DGS optimization loop minimizing photometric loss [S1].
*   **Outputs:** Rendered images and optimized 3D primitives parameterized by DARBF kernels [S1].

**Data Structures:**
The primitive representation is identical to 3D Gaussian Splatting, with the kernel function $K$ changed from Gaussian to a DARBF variant [S1].
*   **Mean Position:** $\boldsymbol{\mu} \in \mathbb{R}^3$ [S1].
*   **Covariance Matrix:** $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$, symmetric positive definite, defining the anisotropic shape and orientation [S1].
*   **Opacity:** $\alpha \in [0, 1]$ [S1].
*   **Color:** $\mathbf{c}$ (typically spherical harmonics coefficients or RGB) [S1].

**The Mahalanobis Distance:**
All DARBF kernels are defined as non-negative functions of the Mahalanobis distance $d_M$ [S1]. For a point $\mathbf{x}$ relative to the center $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$:
$$
d_M(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = \sqrt{(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})}
$$
where $\boldsymbol{\Sigma}^{-1}$ is the inverse covariance matrix [S1].

**Projection and Correction Factor:**
Standard 3DGS exploits the property that the 2D covariance matrix $\boldsymbol{\Sigma}_{2D}$ can be obtained by simply skipping the third row and column of the 3D covariance matrix $\boldsymbol{\Sigma}_{3D}$, bypassing costly integration [S1]. DARBFs do not share this exact closure property. To preserve computational efficiency, DARBF-Splatting introduces a correction factor $\psi$ [S1].
*   The authors evaluated using $\psi$ as a matrix from an MLP, a scalar from closed-form regression, and no correction. They selected the **scalar $\psi$** precomputed from regression experiments as it is computationally efficient and effective [S1].
*   This scalar $\psi$ is applied to approximate the Gaussian's closed-form integration shortcut [S1].

**Kernel Variants:**
The method supports various DARBFs. Key examples include:
1.  **Half-Cosine:** Defined as $\cos\left(\frac{d_M^2}{\xi}\right)$ for $d_M^2 < \frac{\xi \pi}{2}$, where $\xi > 0$ is a scale parameter. It has finite support and a blunt peak [S1].
2.  **Inverse Multiquadric (Inverse MQ):** Spans a larger region with higher opacity for the same variance [S1].
3.  **Radial Cosine / Modified Raised Cosine:** Other variants tested [S1].

These kernels decay spatially. Unbounded growing functions are omitted due to instability [S1]. DARBFs like Half-Cosine and Inverse MQ span a larger region with higher opacity for the same variance compared to Gaussians, reducing the need for multiple splats to achieve the same accumulated value [S1].

**Rendering Steps:**
1.  **Transform:** Transform 3D primitives to the camera coordinate frame [S1].
2.  **Project:** Compute the 2D footprint (splat) using the projected 2D covariance $\boldsymbol{\Sigma}_{2D}$ (derived by dropping the 3rd row/col of $\boldsymbol{\Sigma}_{3D}$) and the correction factor $\psi$ [S1].
3.  **Evaluate:** For each pixel covered by the splat, compute the kernel value $K(d_M)$ using the specific DARBF formula [S1].
4.  **Composite:** Perform alpha compositing using the computed opacity contributions [S1].
5.  **Optimize:** Backpropagate the photometric loss to update $\boldsymbol{\mu}$, $\boldsymbol{\Sigma}$, $\alpha$, and $\mathbf{c}$ [S1].

**Pseudocode:**
The following pseudocode illustrates the evaluation of a single DARBF splat at a pixel location, focusing on the kernel calculation and correction factor application.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_darbf_kernel(x, mu, Sigma_inv, kernel_type, xi, psi):
    """
    Computes the DARBF kernel value for a pixel at x.
    
    Args:
        x: numpy array, 2D pixel coordinate [u, v] (or 3D if evaluating in 3D space)
        mu: numpy array, 3D mean position [mu_x, mu_y, mu_z]
        Sigma_inv: numpy array, 3x3 inverse covariance matrix
        kernel_type: string, 'half_cosine', 'inverse_mq', etc.
        xi: float, scale parameter for the kernel
        psi: float, scalar correction factor to approximate Gaussian closure
        
    Returns:
        float: The corrected kernel value
    """
    # Calculate Mahalanobis distance squared
    diff = x - mu
    # Note: In 2D projection, Sigma_inv should be the 2x2 projected inverse covariance.
    # The paper states 2D covariance is obtained by skipping 3rd row/col of 3D.
    # Assuming x and mu are already in the projected 2D space or the relevant dimensions are selected.
    # For strict adherence to the 3D->2D projection shortcut described:
    # Sigma_inv_2d = Sigma_inv[:2, :2]
    # diff_2d = diff[:2]
    
    # General Mahalanobis distance calculation as per Eq 1
    # d_M^2 = (x - mu)^T * Sigma_inv * (x - mu)
    # Using the full 3D distance if evaluating in 3D, or 2D if projected.
    # The paper implies the projection shortcut allows using the 2D covariance directly.
    
    # Let's assume inputs are 2D projected for the rasterization step
    d_M_sq = diff @ Sigma_inv @ diff
    
    if kernel_type == 'half_cosine':
        # cos(d_M^2 / xi) for d_M^2 < xi * pi / 2
        threshold = xi * math.pi / 2
        if d_M_sq < threshold:
            val = math.cos(d_M_sq / xi)
        else:
            val = 0.0
    elif kernel_type == 'inverse_mq':
        # Standard Inverse Multiquadric: 1 / sqrt(1 + (d_M^2 / xi^2))
        # Or similar variant. The paper mentions "Inverse MQ" spans larger region.
        # Common form: 1 / (1 + (d_M^2 / xi^2))^beta
        # Assuming beta=1 or standard form. The paper doesn't specify beta, 
        # but mentions "Inverse MQ functions". 
        # Let's use a standard form: 1 / sqrt(1 + d_M^2 / xi^2)
        val = 1.0 / math.sqrt(1.0 + d_M_sq / (xi * xi))
    else:
        # Default to Gaussian if not specified, though this is not DARBF
        val = math.exp(-0.5 * d_M_sq)
        
    # Apply correction factor
    corrected_val = val * psi
    return corrected_val

def render_darbf_splat(pixel_x, primitive_mu, primitive_Sigma_inv, primitive_alpha, primitive_color, kernel_type, xi, psi):
    """
    Renders a single DARBF primitive contribution to a pixel.
    """
    # 1. Get 2D projected inverse covariance
    # Skip 3rd row and column of 3x3 Sigma_inv
    Sigma_inv_2d = primitive_Sigma_inv[:2, :2]
    
    # 2. Compute Kernel Value
    # Note: primitive_mu must also be projected to 2D for this calculation
    # Assuming primitive_mu_2d is passed or computed externally
    kernel_val = compute_darbf_kernel(pixel_x, primitive_mu, Sigma_inv_2d, kernel_type, xi, psi)
    
    # 3. Compute Opacity Contribution
    # The kernel value acts as the density/weight. 
    # In 3DGS, alpha is pre-computed or derived. Here, the kernel shape affects the spread.
    # The opacity alpha is a separate parameter.
    # The contribution is typically: color * alpha * kernel_norm
    # However, DARBFs have different integral properties. The correction factor psi 
    # approximates the Gaussian closure, implying it scales the kernel to match 
    # the expected integral or weight.
    
    # The paper states psi approximates the closure property.
    # We assume the standard compositing equation uses the kernel value as the weight.
    
    contribution = primitive_color * primitive_alpha * kernel_val
    
    return contribution
```

**Parameters:**
*   **$\xi$ (xi):** Scale parameter for DARBFs (e.g., Half-Cosine). Typical values are not explicitly given in the notes, but it controls the support width [S1].
*   **$\psi$ (psi):** Scalar correction factor. Precomputed from closed-form regression experiments [S1].
*   **Kernel Type:** 'half_cosine', 'inverse_mq', etc. [S1].

**Edge Cases and Pitfalls:**
*   **Unbounded Functions:** Functions that grow unbounded cause instability and are omitted [S1].
*   **Integration Cost:** Using functions outside the Gaussian family typically incurs costly integration. The correction factor $\psi$ is critical to avoid this [S1].
*   **Precision:** The finite support of Half-Cosine requires careful boundary checking ($d_M^2 < \frac{\xi \pi}{2}$) [S1].

## Operating characteristics
*   **Training Speed:** Half-cosine kernels achieve 34% faster training convergence compared to standard Gaussian kernels [S1].
*   **Memory Usage:** Half-cosine kernels reduce memory usage by 15% compared to standard Gaussian kernels [S1].
*   **Visual Quality:** Comparable PSNR, SSIM, and LPIPS results to 3DGS [S1].
*   **Scaling:** DARBFs like Half-Cosine and Inverse MQ span a larger region with higher opacity for the same variance, reducing the need for multiple splats to achieve the same accumulated value [S1]. This suggests better scaling in terms of primitive count for certain scene complexities.
*   **Strengths:** Faster training, lower memory, generalization beyond exponential family.
*   **Limitations:** Requires a correction factor $\psi$ to approximate Gaussian efficiency; not all DARBFs perform equally (Half-Cosine and Inverse MQ are preferred) [S1].

## Background
DARBF-Splatting was introduced by Hashiru Pramuditha, Vinasirajan Viruthshaan, Vishagar Arunan, Saeedha Nazar, Sameera Ramasinghe, Simon Lucey, and Ranga Rodrigo from the University of Moratuwa and University of Adelaide [S1]. The paper was published in 2026 (WACV 2026) [S1]. It addresses the limitation of 3D Gaussian Splatting being restricted to the exponential family, a gap identified due to the lack of easy integrability for other functions in 3D-to-2D projections [S1]. The method builds on the 3DGS framework, modifying the CUDA kernels to support DARBFs [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
