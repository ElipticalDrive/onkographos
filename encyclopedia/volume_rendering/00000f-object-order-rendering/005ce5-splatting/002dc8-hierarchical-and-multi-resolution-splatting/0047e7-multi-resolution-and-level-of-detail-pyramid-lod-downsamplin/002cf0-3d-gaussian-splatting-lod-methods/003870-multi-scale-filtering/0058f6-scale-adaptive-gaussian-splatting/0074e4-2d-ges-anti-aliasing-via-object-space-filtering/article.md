# 2D-GES Anti-Aliasing via Object Space Filtering

Also called: Back-projected EWA filter, Object space scaling adjustment

## Overview
This method provides a training-free, test-time anti-aliasing technique for 2D Gaussian Splatting (2D-GES) by avoiding screen-space filtering. Instead, it back-projects the screen-space Elliptical Weighted Average (EWA) filter into object space, implementing the anti-aliasing effect as a simple scaling adjustment of the 2D Gaussians [S1]. This approach is distinct from screen-space filters, which are unsuitable for per-splat rendering in 2D-GES due to the use of planar depth and accurate perspective projection [S1]. It sets itself apart from sibling methods like [Alias-free 4D Gaussian Splatting](9872-alias-free-4d-gaussian-splatting.md) by operating directly on the 2D Gaussian scaling parameters rather than modifying the 4D representation or color offsets [S1, 13019-adaptive-scale-color-offset.md].

## How it works
**Inputs and Outputs:**
*   **Input:** The original 2D Gaussian parameters: covariance $\Sigma_i$, scaling matrix (implicitly or explicitly defined), opacity $\sigma_i$, and the affine approximation of the object-to-screen transformation $J_i$.
*   **Output:** Adjusted scaling and opacity parameters ready for rendering, which eliminate aliasing artifacts.

**Data Structures:**
*   **Covariance Matrix ($\Sigma_i$):** A $2 \times 2$ symmetric positive-definite matrix representing the Gaussian spread in object space.
*   **Scaling Matrix ($\mathbf{S}_i$):** A diagonal matrix $\text{diag}(s_{i,1}, s_{i,2})$ used to approximate the filtered covariance.
*   **Transformation Matrix ($J_i$):** The $2 \times 2$ affine approximation of the object-to-screen transformation.
*   **Filter Size ($r$):** A scalar representing the EWA filter radius.

**Parameters:**
*   **$r$:** EWA filter size. Typical value is not explicitly given in the sources, but it is a standard parameter in EWA splatting. [G: Commonly $r=2$ or $3$ pixels in screen space, but must be consistent with the filter definition used in the renderer].
*   **$\Sigma_i$:** Set to Identity matrix $I$ for the unit 2D Gaussian in object space [S1].

**Numbered Steps:**

1.  **Compute Screen-Space Filtered Covariance:**
    Calculate the effective covariance of the Gaussian in screen space. This accounts for the transformation $J_i$ and the EWA filter $r$. The filtered Gaussian in screen space is defined as:
    $$ \rho_i(\hat{\mathbf{x}}) = \frac{1}{|J_i^{-1}|} G_{J_i \Sigma_i J_i^T + rI}(\hat{\mathbf{x}} - \hat{\mathbf{p}}) $$
    where:
    *   $\hat{\mathbf{x}}$ is the pixel position in screen space.
    *   $\hat{\mathbf{p}}$ is the rasterized position of the Gaussian center in screen space.
    *   $J_i$ is the affine approximation of the object-to-screen transformation.
    *   $r$ is the EWA filter size.
    *   $I$ is the $2 \times 2$ identity matrix.
    *   $|J_i^{-1}|$ is the determinant of the inverse transformation matrix.

2.  **Back-Project to Object Space:**
    Transform the screen-space filter back into the local Gaussian object space. The filtered Gaussian in object space $\rho'_i(\mathbf{x})$ is given by:
    $$ \rho'_i(\mathbf{x}) = G_{\Sigma_i + r J_i^{-1} J_i^{-T}}(\mathbf{x} - \mathbf{p}) $$
    where:
    *   $\mathbf{x}$ is the position in object space.
    *   $\mathbf{p}$ is the Gaussian center in object space.
    *   Note: The source writes $J_i^{-1} J_i^{-1} T$, which denotes $J_i^{-1} (J_i^{-1})^T$ or $J_i^{-1} J_i^{-T}$ [S1].

3.  **Approximate with Scaling Matrix:**
    Since the method uses a unit 2D Gaussian in object space, set $\Sigma_i = I$. The covariance matrix of the filtered Gaussian becomes $I + r J_i^{-1} J_i^{-T}$. To simplify computation, approximate this covariance matrix using a diagonal scaling transformation $\mathbf{S}_i = \text{diag}(s_{i,1}, s_{i,2})$ such that:
    $$ \mathbf{S}_i \mathbf{S}_i^T \approx I + r J_i^{-1} J_i^{-T} $$
    This implies:
    $$ s_{i,1}^2 \approx 1 + r (J_i^{-1} J_i^{-T})_{0,0} $$
    $$ s_{i,2}^2 \approx 1 + r (J_i^{-1} J_i^{-T})_{1,1} $$
    [G: In practice, one might compute the eigenvalues of $I + r J_i^{-1} J_i^{-T}$ and take the square roots, or simply use the diagonal elements if the off-diagonal terms are negligible or rotated out. The source states "approximate... using a scaling transformation", implying a diagonal approximation is sufficient for the shader implementation].

4.  **Adjust Rendering Parameters:**
    *   Multiply the original scaling of the 2D Gaussians by $\mathbf{S}_i$. If the original scaling is represented by a matrix $\mathbf{S}_{orig}$, the new scaling is $\mathbf{S}_{new} = \mathbf{S}_{orig} \mathbf{S}_i$.
    *   Multiply the original opacity $\sigma_i$ by $1/(s_{i,1}s_{i,2})$ to preserve the integral (energy) of the Gaussian after scaling.
    $$ \sigma_{new} = \frac{\sigma_i}{s_{i,1} s_{i,2}} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def apply_object_space_filtering(gaussian_covariance, gaussian_opacity, J_inv, r):
    """
    Applies anti-aliasing via object space filtering to a 2D Gaussian.
    
    Args:
        gaussian_covariance (np.ndarray): 2x2 covariance matrix Sigma_i in object space.
        gaussian_opacity (float): Original opacity sigma_i.
        J_inv (np.ndarray): 2x2 inverse of the object-to-screen transformation matrix.
        r (float): EWA filter size.
        
    Returns:
        tuple: (new_scaling_matrix, new_opacity)
    """
    # Step 1 & 2: Compute the back-projected covariance in object space
    # Covariance = Sigma_i + r * J_inv @ J_inv.T
    # Note: The source uses Sigma_i = I for the unit Gaussian assumption in the approximation step,
    # but the general formula includes Sigma_i. We follow the general formula first.
    back_projected_cov = gaussian_covariance + r * (J_inv @ J_inv.T)
    
    # Step 3: Approximate with scaling matrix S_i = diag(s1, s2)
    # We approximate back_projected_cov with S_i @ S_i.T
    # A simple approximation is to take the square root of the diagonal elements
    # if we assume the off-diagonal elements are small or handled by rotation.
    # However, a more robust approximation for a diagonal scaling matrix 
    # matching the variance is to use the eigenvalues or just the diagonal.
    # The source says "approximate... using a scaling transformation".
    # Let's use the diagonal elements of the back-projected covariance as the squared scales.
    # This assumes the principal axes align with the coordinate system or the error is acceptable.
    
    s1_sq = back_projected_cov[0, 0]
    s2_sq = back_projected_cov[1, 1]
    
    # Guard against negative values due to numerical precision
    s1_sq = max(s1_sq, 1e-6)
    s2_sq = max(s2_sq, 1e-6)
    
    s1 = np.sqrt(s1_sq)
    s2 = np.sqrt(s2_sq)
    
    # Step 4: Adjust scaling and opacity
    # The original scaling is not explicitly passed as a matrix, but implied by the covariance.
    # If the Gaussian is defined by scaling S_orig and rotation R, then Sigma = S_orig R R^T S_orig^T.
    # The method says "multiply the original scaling ... by S_i".
    # So we return the scaling factors s1, s2 to be multiplied with the existing scale.
    # And the opacity adjustment.
    
    opacity_scale_factor = 1.0 / (s1 * s2)
    new_opacity = gaussian_opacity * opacity_scale_factor
    
    # Return the scaling adjustment factors
    # The caller should multiply their existing scale parameters by s1 and s2 respectively.
    return (s1, s2), new_opacity
```

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure $s_{i,1}$ and $s_{i,2}$ are positive. If the back-projected covariance diagonal elements are negative due to numerical error, clamp them to a small positive value [G].
*   **Boundaries:** The method assumes the affine approximation $J_i$ is valid. For highly non-linear projections, this approximation may introduce artifacts.
*   **Degenerate Input:** If $J_i$ is singular (e.g., Gaussian is viewed edge-on or at infinity), $J_i^{-1}$ is undefined. Check for singularity before inversion [G].

## Subtopics

### Alias-free 4D Gaussian Splatting
Extends anti-aliasing to 4D (3D space + time) by modifying the 4D Gaussian representation. Choose when temporal consistency and 4D scenes are required. [Link](9872-alias-free-4d-gaussian-splatting.md)

### Adaptive-Scale Color Offset
Adjusts color based on scale to reduce aliasing. Choose when color bleeding is a primary concern. [Link](13019-adaptive-scale-color-offset.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per Gaussian, as it involves matrix multiplication and square root operations.
*   **Memory Complexity:** $O(1)$ additional memory per Gaussian for storing the scaling factors.
*   **Measured Performance:** The method is described as "concise and efficient" for shader implementation [S1]. It does not increase the number of primitives. Table 6 in [S1] shows that 2D-GES uses 0.18M surfels + 2.21M Gaussians for the Garden scene, comparable to other methods.
*   **Strengths:** Training-free, test-time plugin, efficient shader implementation, no need for screen-space filtering.
*   **Limitations:** Relies on the affine approximation $J_i$, which may be inaccurate for extreme perspectives. Approximates the covariance with a diagonal scaling matrix, potentially losing rotational information.
*   **When to use:** When rendering 2D Gaussian Splatting scenes with high-frequency details and aliasing artifacts are visible, and a lightweight, training-free solution is preferred.
*   **When not to use:** When high accuracy in rotational covariance is required, or when the scene contains extreme perspective distortions that violate the affine approximation.

## Background
The method originates from the paper "When Gaussian Meets Surfel: Ultra-fast High-fidelity Radiance Field Rendering" [S1]. It builds on the concept of back-projecting screen-space filters into object space, a technique previously used in [Ren et al. 2002b] for EWA splatting [S1]. The specific application to 2D-GES addresses the limitations of screen-space filters in per-splat rendering with planar depth and accurate perspective projection [S1].

## Sources
[S1] When Gaussian Meets Surfel: Ultra-fast High-fidelity Radiance Field Rendering, https://doi.org/10.1145/3730925, fetched 2026-10-26
