# Isotropic Gaussian Scale Contraction

Also called: Isotropic scale contraction, Jacobian-based isotropic scale estimation

## Overview
Isotropic Gaussian Scale Contraction is a technique for efficiently adapting the scale of isotropic Gaussians when a scene contraction function $C(\mathbf{x})$ is applied. It solves the problem of maintaining correct anti-aliasing properties in contracted coordinate spaces without the computational overhead of general multivariate Gaussian transformations. It sets itself apart from the standard mip-NeRF 360 approach by linearizing the contraction around each Gaussian center to derive an isotropic scale from the Jacobian determinant, rather than contracting a full covariance matrix [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of isotropic Gaussians defined by their means $\mathbf{x}_j$ and scales $\sigma_j$, and a scene contraction function $C: \mathbb{R}^3 \to \mathbb{R}^3$.
*   **Output:** The contracted means $\mathbf{x}'_j = C(\mathbf{x}_j)$ and the contracted isotropic scales $\sigma'_j$.

**Data Structures**
The method operates on scalar values for scale and 3D vectors for position. No complex covariance matrices are stored or manipulated.

**Steps**
1.  **Contract Means:** Evaluate the contraction function at the original Gaussian center:
    $$ \mathbf{x}'_j = C(\mathbf{x}_j) $$
2.  **Compute Jacobian Determinant Factor:** Calculate the scaling factor derived from the determinant of the Jacobian of the contraction function $J_C(\mathbf{x}_j)$. For the specific contraction function used in Zip-NeRF, this factor has a closed-form solution:
    $$ \text{scale\_factor}_j = \left| \det(J_C(\mathbf{x}_j)) \right|^{1/3} = \left( \frac{\sqrt[3]{2 \max(1, \|\mathbf{x}_j\|) - 1}}{\max(1, \|\mathbf{x}_j\|)} \right)^2 $$
    Here, $\|\mathbf{x}_j\|$ is the Euclidean norm of the original mean $\mathbf{x}_j$.
3.  **Update Scale:** Multiply the original isotropic scale $\sigma_j$ by the computed factor:
    $$ \sigma'_j = \sigma_j \times \text{scale\_factor}_j $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def isotropic_gaussian_scale_contraction(means: np.ndarray, scales: np.ndarray) -> tuple:
    """
    Contracts isotropic Gaussians using the Zip-NeRF specific closed-form Jacobian determinant.
    
    Args:
        means: Array of shape (N, 3) containing Gaussian centers x_j.
        scales: Array of shape (N,) containing isotropic scales sigma_j.
        
    Returns:
        contracted_means: Array of shape (N, 3) containing C(x_j).
        contracted_scales: Array of shape (N,) containing sigma'_j.
    """
    # Step 1: Contract means
    # Note: The specific definition of C(x) is required here. 
    # In Zip-NeRF, C(x) typically involves a radial contraction.
    # Assuming C(x) is defined elsewhere or passed in, we evaluate it.
    # For the scale factor derivation, we only need the norm of the original mean.
    
    norms = np.linalg.norm(means, axis=1)
    
    # Step 2: Compute the closed-form scale factor from the Jacobian determinant
    # Formula: ( cbrt(2 * max(1, norm) - 1) / max(1, norm) )^2
    max_norm = np.maximum(1.0, norms)
    numerator = np.cbrt(2.0 * max_norm - 1.0)
    scale_factor = (numerator / max_norm) ** 2
    
    # Step 3: Update scales
    contracted_scales = scales * scale_factor
    
    # Return contracted means (requires evaluating C(x_j)) and new scales
    # C(x) implementation depends on the specific scene contraction used.
    # A common contraction maps x to x / max(1, ||x||) or similar.
    # Here we assume a helper function `contract_point` exists.
    contracted_means = np.array([contract_point(m) for m in means])
    
    return contracted_means, contracted_scales

def contract_point(x: np.ndarray) -> np.ndarray:
    """
    Placeholder for the specific scene contraction function C(x).
    The scale factor formula provided in sources is specific to the contraction
    used in Zip-NeRF. The mean contraction must be consistent with the 
    Jacobian derivation.
    """
    # Example contraction logic (must match the Jacobian source):
    # This is a generic placeholder; the actual C(x) must be implemented 
    # based on the specific Zip-NeRF contraction definition.
    # The scale factor formula implies a specific radial behavior.
    pass
```

**Parameters**
*   $\|\mathbf{x}_j\|$: Euclidean norm of the Gaussian mean.
*   $\max(1, \|\mathbf{x}_j\|)$: Ensures stability for points near the origin.

**Edge Cases and Pitfalls**
*   **Origin Singularity:** The formula uses $\max(1, \|\mathbf{x}_j\|)$ in the denominator. If $\|\mathbf{x}_j\| < 1$, the denominator is 1, avoiding division by zero. The numerator becomes $\sqrt[3]{2(1)-1} = 1$, so the scale factor is $1^2/1^2 = 1$. This implies no scale change for points within the unit sphere, which is consistent with many contraction functions that leave the central region uncontracted.
*   **Naive Multivariate Approach:** Contracting the scale by treating the Gaussian as a full multivariate Gaussian with covariance $\sigma^2 I$ and then identifying the isotropic Gaussian with the same generalized variance is significantly more expensive [S1]. This method avoids that by using the linearized Jacobian approximation.

## Subtopics

## Operating characteristics
**Time and Memory Complexity**
The method is $O(N)$ where $N$ is the number of Gaussians. It requires only vector norm calculations and scalar multiplications, making it significantly less computationally expensive than the multivariate Gaussian contraction approach used in mip-NeRF 360 [S1].

**Strengths and Limitations**
*   **Strengths:** High computational efficiency; avoids matrix operations; closed-form solution for the specific Zip-NeRF contraction.
*   **Limitations:** It is an approximation based on linearization. It assumes the contraction function is locally linear around $\mathbf{x}_j$. For very large Gaussians or highly non-linear contractions, this approximation may introduce errors. It is specific to isotropic Gaussians.

**When to Use**
Use this method when rendering with isotropic Gaussians in a scene that has been contracted (e.g., for distant scene content allocation) and computational efficiency is a priority. It is particularly suitable for Zip-NeRF and similar architectures that use multisampled isotropic Gaussians.

## Background
This technique was introduced in Zip-NeRF [S1]. It was developed to address the need for efficient anti-aliasing in neural radiance fields that use scene contraction to allocate model capacity proportional to disparity. The method improves upon the mip-NeRF 360 approach [S1] by reducing the computational cost of scale contraction while maintaining the necessary anti-aliasing properties.

## Sources
[S1] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
