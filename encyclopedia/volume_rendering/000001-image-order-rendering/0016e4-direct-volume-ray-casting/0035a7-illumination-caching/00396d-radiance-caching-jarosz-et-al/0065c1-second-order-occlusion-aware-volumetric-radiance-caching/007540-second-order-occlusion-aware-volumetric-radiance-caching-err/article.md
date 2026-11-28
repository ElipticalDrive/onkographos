# Second-Order Occlusion-Aware Volumetric Radiance Caching Error Metric and Radius Computation

Also called: Ellipsoidal valid region for radiance caching, Hessian-based Radius Computation, Hyperellipsoid Radius Computation, Second-Order Error Control for Media Radiance Extrapolation, Second-Order Error Metric, Second-order Taylor expansion error metric

## Overview
This method defines the valid spatial region around a radiance cache point where linear interpolation (or extrapolation) of volumetric radiance remains within a user-specified relative error tolerance $\epsilon$. It computes the geometry of this region—specifically a hyperellipsoid in $k$ dimensions—by analyzing the second-order derivatives (Hessian) of the radiance field. Unlike first-order methods that assume constant radiance or linear gradients, this approach accounts for curvature, allowing for larger cache radii in smooth regions and tighter bounds in high-frequency areas. It builds on [Second-Order Occlusion-Aware Volumetric Radiance Caching](19428-second-order-occlusion-aware-volumetric-radiance-caching.md) by providing the specific mathematical formulation for the cache validity bounds.

## How it works
The method determines the maximum distance a point $\mathbf{x}'$ can be from a cache point $\mathbf{x}$ such that the error introduced by approximating the radiance $L(\mathbf{x}')$ using the Taylor expansion around $\mathbf{x}$ does not exceed $\epsilon$.

### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{x}$: The cache point position in 3D space.
    *   $L(\mathbf{x})$: The radiance value at $\mathbf{x}$.
    *   $\mathbf{H}L(\mathbf{x})$: The Hessian matrix of the radiance at $\mathbf{x}$, a $3 \times 3$ symmetric matrix containing second-order partial derivatives.
    *   $\epsilon$: The desired relative error threshold (scalar, e.g., $0.05$ for 5%).
    *   $k$: The dimensionality of the space (typically 2 for surface-parallel projection or 3 for full volumetric space).
*   **Outputs**:
    *   Principal radii $R^{\lambda_i}$: The semi-axes lengths of the valid hyperellipsoid along the eigenvectors of the Hessian.
    *   Alternatively, a single scalar radius $R$ for a hyperspherical approximation.

### Data Structures
The Hessian matrix $\mathbf{H}L(\mathbf{x})$ is a $3 \times 3$ symmetric matrix. To compute the radii, it must be decomposed into its eigenvalues and eigenvectors.
*   **Eigenvalues ($\lambda_i$)**: Three scalar values representing the curvature along principal axes.
*   **Eigenvectors ($\vec{v}_{\lambda_i}$)**: Three orthogonal unit vectors defining the orientation of the ellipsoid.

### Formulas
The relative error $\hat{\varepsilon}'(\mathbf{x}')$ of the extrapolation at a point $\mathbf{x}'$ is approximated by the second-order term of the Taylor series [S1]:

$$
\hat{\varepsilon}'(\mathbf{x}') \approx \frac{|\Delta \mathbf{x}'^\top \mathbf{H}L(\mathbf{x}) \Delta \mathbf{x}'|}{2L(\mathbf{x})}
$$

where $\Delta \mathbf{x}' = \mathbf{x}' - \mathbf{x}$.

To find the valid region where $\hat{\varepsilon}'(\mathbf{x}') \le \epsilon$, the inequality is integrated over the neighborhood. The resulting valid region is a hyperellipsoid aligned with the eigenvectors of $\mathbf{H}L(\mathbf{x})$. The principal radii $R^{\lambda_i}$ along each eigenvector $\vec{v}_{\lambda_i}$ are computed as follows:

**For 2D space ($k=2$):**
$$
R_{2D}^{\lambda_i} = \sqrt[4]{\frac{4 L(\mathbf{x}) \epsilon}{\pi |\lambda_i|}}
$$

**For 3D space ($k=3$):**
$$
R_{3D}^{\lambda_i} = \sqrt[5]{\frac{15 L(\mathbf{x}) \epsilon}{4 \pi |\lambda_i|}}
$$

In these formulas:
*   $L(\mathbf{x})$ is the radiance magnitude at the cache point.
*   $\epsilon$ is the relative error tolerance.
*   $|\lambda_i|$ is the absolute value of the $i$-th eigenvalue of the Hessian $\mathbf{H}L(\mathbf{x})$.
*   The constants $\pi$, $4$, $15$, and $4$ arise from the integration of the quadratic error form over a unit circle (2D) or unit sphere (3D) [S1].

**Hyperspherical Approximation:**
For implementation simplicity, a single radius $R$ can be used instead of an ellipsoid. This is derived by using the maximum absolute eigenvalue $|\lambda_{max}| = \max_i |\lambda_i|$:

$$
R = \sqrt[k+1]{\frac{C_k L(\mathbf{x}) \epsilon}{C_{sphere} |\lambda_{max}|}}
$$

where $C_k$ and $C_{sphere}$ are the geometric constants corresponding to the dimension $k$ (e.g., for 3D, $C_3 = 15/4\pi$). This defines a hypersphere that is conservative (smaller) than the ellipsoid, ensuring the error bound is met in all directions [S1].

### Pseudocode
The following Python-like pseudocode computes the principal radii for a 3D cache point. It assumes access to standard linear algebra operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_cache_radii(hessian_matrix, radiance, epsilon, dimension=3):
    """
    Computes the principal radii of the valid region for radiance caching.
    
    Args:
        hessian_matrix (np.array): 3x3 symmetric Hessian matrix of radiance at cache point.
        radiance (float): L(x), the radiance value at the cache point.
        epsilon (float): Relative error threshold.
        dimension (int): 2 for 2D subspace, 3 for 3D volume.
        
    Returns:
        radii (list): List of principal radii R_i corresponding to eigenvectors.
    """
    # 1. Compute eigenvalues of the Hessian
    # Hessian is symmetric, so we use eigvalsh for stability
    eigenvalues, eigenvectors = np.linalg.eigh(hessian_matrix)
    
    # 2. Determine geometric constants based on dimension
    # Derived from integration of quadratic error over unit ball
    if dimension == 2:
        # Constant for 2D: 4 / pi
        # Formula: R = (4 * L * epsilon / (pi * |lambda|))^(1/4)
        numerator = 4.0 * radiance * epsilon
        denominator_const = math.pi
        power = 0.25 # 1/4
    elif dimension == 3:
        # Constant for 3D: 15 / (4 * pi)
        # Formula: R = (15 * L * epsilon / (4 * pi * |lambda|))^(1/5)
        numerator = 15.0 * radiance * epsilon
        denominator_const = 4.0 * math.pi
        power = 0.2 # 1/5
    else:
        raise ValueError("Only 2D and 3D dimensions are supported by the source formulas.")

    radii = []
    for i in range(dimension):
        # Get the i-th eigenvalue
        lam = eigenvalues[i]
        
        # Avoid division by zero if curvature is zero (flat region)
        # If lambda is 0, the error is dominated by higher order terms or noise.
        # In practice, clamp to a small epsilon or use a fallback radius.
        abs_lam = abs(lam)
        if abs_lam < 1e-10:
            # [G] Fallback: If curvature is negligible, the error metric is undefined.
            # Use a large radius or a default minimum radius based on scene scale.
            radius = float('inf') 
        else:
            # Compute radius using the derived formula
            radius = (numerator / (denominator_const * abs_lam)) ** power
        
        radii.append(radius)
        
    return radii, eigenvectors

def compute_hypersphere_radius(hessian_matrix, radiance, epsilon, dimension=3):
    """
    Computes a single conservative radius using the maximum eigenvalue.
    """
    eigenvalues, _ = np.linalg.eigh(hessian_matrix)
    
    # Find max absolute eigenvalue
    max_abs_lam = max(abs(l) for l in eigenvalues)
    
    if dimension == 2:
        numerator = 4.0 * radiance * epsilon
        denominator_const = math.pi
        power = 0.25
    elif dimension == 3:
        numerator = 15.0 * radiance * epsilon
        denominator_const = 4.0 * math.pi
        power = 0.2
    else:
        raise ValueError("Unsupported dimension.")
        
    if max_abs_lam < 1e-10:
        radius = float('inf')
    else:
        radius = (numerator / (denominator_const * max_abs_lam)) ** power
        
    return radius
```

### Parameters and Typical Values
*   **$\epsilon$**: The relative error threshold. Typical values range from $0.01$ to $0.1$ depending on the desired visual quality and performance. Lower $\epsilon$ results in smaller radii and denser caching.
*   **Dimension**: Usually 3 for volumetric media. If the cache is projected onto a 2D surface (e.g., for surface irradiance caching), use 2 [S1].

### Edge Cases and Pitfalls
1.  **Zero Curvature**: If an eigenvalue $\lambda_i$ is zero (or near zero), the denominator in the radius formula becomes zero, leading to an infinite radius. This occurs in regions where radiance is locally linear or constant. In practice, a minimum radius or a fallback strategy must be implemented [G].
2.  **Monte Carlo Noise**: The Hessian and radiance values are typically estimated using Monte Carlo tracing. This introduces variance and bias. The deterministic error metric assumes perfect knowledge of $L(\mathbf{x})$ and $\mathbf{H}L(\mathbf{x})$. In noisy scenes, the computed radii may be inaccurate, leading to either over-smoothing (if radii are too large) or artifacts (if radii are too small) [S1].
3.  **Negative Radiance**: Radiance is physically non-negative, but numerical errors or specific material models might produce negative values. The formula uses $L(\mathbf{x})$ in the numerator. If $L(\mathbf{x})$ is negative, the radius calculation becomes invalid. Ensure $L(\mathbf{x})$ is clamped to $\ge 0$ [G].
4.  **Extrapolation vs. Interpolation**: The metric is derived for extrapolation from a single point. In practice, an "overture pass" pre-populates the cache, allowing the final pass to use interpolation from multiple points. The error of the weighted sum is upper-bounded by $\hat{\epsilon}'$ if only points within the valid region are used [S1].

## Operating characteristics
*   **Time Complexity**: Computing the Hessian eigenvalues is $O(1)$ for a fixed $3 \times 3$ matrix. The dominant cost is the estimation of the Hessian itself, which requires multiple ray samples or finite difference approximations, scaling with the number of samples per cache point.
*   **Memory Complexity**: $O(1)$ per cache point to store the radiance, gradient, Hessian, and radii.
*   **Strengths**: Provides a theoretically grounded bound on interpolation error. Adapts cache density to local frequency content, reducing samples in smooth areas.
*   **Limitations**: Sensitive to noise in derivative estimation. Requires computation of second-order derivatives, which is more expensive than first-order gradients. The assumption of local quadratic behavior may break down in highly discontinuous scenes (e.g., sharp shadows, thin geometry).

## Background
This method originates from the work on radiance caching for participating media, specifically extending the theory of second-order error control from surface global illumination to volumetric rendering. The foundational work on radiance caching for media was introduced by Jarosz et al. in 2008 [S1, Ref 1]. The specific second-order error metric and radius computation for media were detailed in later work by Jarosz et al. (2012) and related publications [S1, Ref 2]. The 2D formula is analogous to the relative error metric proposed by Schwarzhaupt et al. (2012) for surface irradiance caching, where radii are computed by projecting the surface irradiance Hessian onto the 2D surface subspace [S1].

## Sources
[S1] Second-Order Occlusion-Aware Volumetric Radiance Caching, https://doi.org/10.1145/3185225, fetched 2026-10-26
