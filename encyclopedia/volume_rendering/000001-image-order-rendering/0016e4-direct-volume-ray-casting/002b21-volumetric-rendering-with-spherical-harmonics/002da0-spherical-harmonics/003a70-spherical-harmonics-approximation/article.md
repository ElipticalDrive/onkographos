# Spherical Harmonics Approximation

Also called: SH Approximation, Spherical Harmonics Basis Approximation

## Overview
Spherical Harmonics (SH) Approximation is a technique to approximate a spherical function $f(\omega)$ using a finite subset of spherical harmonics basis functions. It solves the problem of expensive Monte Carlo integration for spherical integrals by providing an efficient, differentiable, and analytic alternative for inverse problems [S1]. It sets itself apart from direct ray casting by trading high-frequency accuracy for computational speed and gradient compatibility, making it suitable for optimization tasks where the signal is sufficiently smooth [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A spherical function $f(\omega)$ (e.g., radiance, visibility, or hemisphere response) defined over the domain $\Omega$ (the unit sphere).
*   **Output:** A coefficient vector $\mathbf{f} \in \mathbb{R}^n$ representing the SH approximation, and optionally the reconstructed function $\tilde{f}(\omega)$ at specific directions.

**Data Structures:**
*   **Coefficient Vector:** A 1D array of size $n = (L+1)^2$, where $L$ is the maximum degree (bandwidth). The sources specify a typical bandwidth of $n=8$ (implying $L=2$ if $n$ refers to the count, or potentially higher if $n$ refers to the degree; the text states "bandwidth of $n=8$" and "number of bands", but standard SH notation uses $L$ for degree. The index formula $i=l^2+l+m$ implies $l$ is the degree. If $n=8$ is the number of coefficients, $L=2$ ($3^2=9$ coeffs? No, $L=2$ gives 9 coeffs. $L=1$ gives 4. The text says "bandwidth of $n=8$". This is ambiguous. However, the index formula is explicit. We will use $L$ for degree and $n=(L+1)^2$ for size).
*   **Index Mapping:** Coefficients are linearized using the index $i$:
    $$
    i = l^2 + l + m
    $$
    where $l \in \{0, \dots, L\}$ is the degree (band index) and $m \in \{-l, \dots, l\}$ is the order (index within the band) [S1, Parent].

**Steps:**

1.  **Projection (Coefficient Extraction):**
    Compute the SH coefficients $f_i$ by projecting the function $f(\omega)$ onto the basis functions $y_i(\omega)$:
    $$
    f_i = \int_{\Omega} f(\omega) y_i(\omega) d\omega
    $$
    [S1, Parent].
    *   *Note:* Direct integration is expensive. For inverse problems, terms like the environment map coefficients are precomputed using numerical integration at initialization [S1].

2.  **Reconstruction:**
    Approximate the function at a direction $\omega$ using the coefficients:
    $$
    \tilde{f}(\omega) = \sum_{i=1}^{n} f_i y_i(\omega)
    $$
    [S1, Parent].

3.  **Product Convolution (Multiplication):**
    To compute the SH coefficients of the product of two functions $\tilde{a}(\omega)$ and $\tilde{b}(\omega)$, denoted $\mathbf{a} * \mathbf{b}$:
    $$
    (\mathbf{a} * \mathbf{b})_i = \sum_{j=1}^{n} \sum_{k=1}^{n} C_{ijk} a_j b_k
    $$
    where $C_{ijk}$ is a precomputed 3D transfer matrix:
    $$
    C_{ijk} = \int_{\Omega} y_i(\omega) y_j(\omega) y_k(\omega) d\omega
    $$
    [S1, Parent].

4.  **Integral of Product:**
    The integral of the product of two approximated functions is the dot product of their coefficient vectors:
    $$
    \int_{\Omega} \tilde{a}(\omega)\tilde{b}(\omega)d\omega = \mathbf{a} \cdot \mathbf{b}
    $$
    [S1, Parent].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_sh_coefficients(function_samples, directions, degree_L):
    """
    Computes SH coefficients for a spherical function.
    Note: In practice, this integral is often precomputed or solved via
    numerical integration for static terms (like environment maps) [S1].
    """
    n = (degree_L + 1) ** 2
    coeffs = np.zeros(n)
    
    # Precompute basis functions Y(l, m, omega) for all l, m, omega
    # This is a heavy operation, typically done once or cached.
    # Y_matrix shape: (n, num_samples)
    Y_matrix = compute_basis_matrix(directions, degree_L)
    
    # Project function f(omega) onto basis
    # f_samples shape: (num_samples,)
    # coeffs = Y_matrix^T @ f_samples / (4*pi) ? 
    # The integral definition is f_i = int f(omega) y_i(omega) d omega.
    # Discrete approximation: sum(f(omega_k) * y_i(omega_k) * dOmega_k)
    # Assuming uniform or weighted quadrature is handled in Y_matrix construction.
    coeffs = Y_matrix.T @ function_samples
    
    return coeffs

def reconstruct_sh(coeffs, direction, degree_L):
    """
    Reconstructs the function value at a specific direction.
    """
    n = (degree_L + 1) ** 2
    # Get basis vector y(omega) for the specific direction
    # y_vec shape: (n,)
    y_vec = compute_basis_vector(direction, degree_L)
    
    # Reconstruction: sum(f_i * y_i(omega))
    value = np.dot(coeffs, y_vec)
    return value

def sh_multiply(coeffs_a, coeffs_b, degree_L):
    """
    Computes the SH coefficients of the product of two functions.
    """
    n = (degree_L + 1) ** 2
    # Load precomputed transfer matrix C
    # C shape: (n, n, n)
    C = load_transfer_matrix(degree_L)
    
    result_coeffs = np.zeros(n)
    for i in range(n):
        # (a * b)_i = sum_j sum_k C_ijk a_j b_k
        # Optimization: Use tensor contraction or matrix multiplication
        # result_coeffs[i] = np.dot(coeffs_a, C[i].T @ coeffs_b) 
        # But C is 3D. C[i] is (n, n).
        # Let's write it explicitly as per formula [S1]
        val = 0.0
        for j in range(n):
            for k in range(n):
                val += C[i, j, k] * coeffs_a[j] * coeffs_b[k]
        result_coeffs[i] = val
        
    return result_coeffs

def compute_hemisphere_response(normal, degree_L):
    """
    Computes the SH coefficients for the hemisphere response H(x).
    H(x) = int max(n(x) . omega, 0) y(omega) d omega.
    Reformulated as: H(x) = phi_SH(n(x)) * int max(i_z . omega, 0) y(omega) d omega [S1].
    """
    # 1. Precompute the z-axis aligned hemisphere coefficients H_z
    # This integral is solved numerically at initialization [S1].
    H_z = compute_z_hemisphere_coeffs(degree_L)
    
    # 2. Apply rotation to align with normal n(x)
    # phi_SH(n(x)) is a linear transformation (rotation matrix in SH space) [S1]
    # This makes the computation differentiable w.r.t n(x) [S1].
    H_x = rotate_sh_coefficients(H_z, normal, degree_L)
    
    return H_x

def rotate_sh_coefficients(coeffs, normal, degree_L):
    """
    Rotates SH coefficients to align with a new normal vector.
    Uses Wigner D-matrices or equivalent linear transformation phi_SH [Parent].
    """
    n = (degree_L + 1) ** 2
    # Construct rotation matrix R from normal
    # This is a standard SH rotation operation [Parent]
    R = construct_rotation_matrix(normal, degree_L)
    
    # Apply linear transformation
    rotated_coeffs = R @ coeffs
    return rotated_coeffs
```

**Parameters:**
*   **Bandwidth ($L$ or $n$):** The maximum degree of the spherical harmonics. The source uses $n=8$ as a bandwidth for good low-frequency approximations [S1]. Note: The text says "bandwidth of $n=8$" and "number of bands". Standard SH count is $(L+1)^2$. If $n=8$ is the count, it's not a perfect square. It is likely $L=2$ (9 coeffs) or $L=3$ (16 coeffs) and 8 is a typo or refers to something else. However, the text explicitly says "bandwidth of $n=8$". We will note this ambiguity. The index formula uses $l$ (degree).
*   **Transfer Matrix ($C_{ijk}$):** Precomputed for the chosen degree $L$.

**Edge Cases and Pitfalls:**
*   **Gibbs Phenomenon:** Truncating the series causes ringing artifacts for high-frequency signals [Parent].
*   **Smoothness:** The method assumes input functions are sufficiently smooth. Discontinuities (e.g., hard shadows) cause slow convergence [Parent].
*   **Differentiability:** The hemisphere projection must be reformulated to be differentiable with respect to the normal $n(x)$ [S1].

## Subtopics

### Hemisphere Projection in SH Space
A specific application of SH approximation to compute the visibility term $H(x) = \int_{\Omega} \max(n(x) \cdot \omega, 0) y(\omega) d\omega$. It is set apart by its reformulation using a linear rotation function $\phi_{SH}(n(x))$ applied to a precomputed z-axis aligned integral, making it differentiable with respect to the surface normal $n(x)$ [S1]. Choose this when you need to optimize geometry or normals in an inverse problem where visibility is a factor [S1].

## Operating characteristics
*   **Time Complexity:** Reconstruction is $O(n)$ per direction. Multiplication is $O(n^3)$ due to the transfer matrix, but $n$ is small (e.g., 9-16 for low degrees). Precomputation of $C_{ijk}$ is done once.
*   **Memory Complexity:** $O(n^3)$ to store the transfer matrix $C$. For $L=2$ ($n=9$), this is $729$ floats, which is negligible.
*   **Performance:** Significantly faster than Monte Carlo integration for inverse problems, as it avoids sampling [S1].
*   **Strengths:** Differentiable, analytic solution for integrals, efficient for low-frequency signals.
*   **Limitations:** Poor approximation of high-frequency details (Gibbs phenomenon), not suitable for sharp discontinuities.
*   **When to use:** Inverse problems, geometry optimization, soft shadows, environment lighting with smooth variations.
*   **When not to use:** Hard shadows, high-frequency specular highlights, real-time rendering where exact precision is required over optimization speed.

## Background
Originated in the context of efficient and differentiable shadow computation for inverse problems [S1]. The method leverages the properties of spherical harmonics to replace expensive Monte Carlo integration with analytic operations [S1]. It is used in rendering systems that optimize geometry and lighting parameters simultaneously.

## Sources
[S1] Efficient and Differentiable Shadow Computation for Inverse Problems, https://doi.org/10.1109/iccv48922.2021.01286, fetched 2026-11-01
