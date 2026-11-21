# Spherical Beta Function

Also called: SB color representation

## Overview
The Spherical Beta (SB) function is a directional appearance representation used to model view-dependent effects, such as specular reflections, in 3D Gaussian splatting. It replaces spherical harmonics (SH) to capture higher-frequency directional variations by using asymmetric, bounded-support lobes [S1, S2, unofficial]. Unlike SH, which uses global basis functions, SB uses localized kernels that can model skewed and sharper directional variation, offering greater flexibility at the cost of increased optimization difficulty [S2, unofficial].

## How it works
The SB function represents the color or intensity of a Gaussian surfel as a sum of $K$ directional lobes. Each lobe is defined by a principal direction and two shape parameters that control the asymmetry and sharpness of the lobe.

**Inputs and Outputs:**
- **Input:** Viewing direction $\omega \in \mathbb{R}^3$ (unit vector), number of lobes $K$, set of principal directions $\{s_k\}_{k=1}^K$, and shape parameters $\{\alpha_k, \beta_k\}_{k=1}^K$.
- **Output:** Scalar value $f_{SB}(\omega)$ representing the directional intensity or color component.

**Data Structures:**
For each Gaussian primitive, the view-dependent color is stored as:
- $K$: Integer, number of lobes (typically small, e.g., 1-4).
- $s_k$: 3D vector, principal direction of the $k$-th lobe. Normalized to unit length.
- $\alpha_k$: Scalar, shape parameter $> 0$. Controls the "rise" side of the lobe.
- $\beta_k$: Scalar, shape parameter $> 0$. Controls the "fall" side of the lobe.

**Formula:**
The SB function is defined as [S2, unofficial]:

$$
f_{SB} (\omega; \alpha, \beta, s) = \sum_{k=1}^{K} (1+s_k \cdot \omega)^{\alpha_k -1} (1-s_k \cdot \omega)^{\beta_k -1}
$$

Where:
- $s_k \cdot \omega$ is the dot product between the principal direction and the viewing direction.
- $\alpha_k, \beta_k > 0$.
- The term $(1+s_k \cdot \omega)$ is non-negative because $s_k \cdot \omega \in [-1, 1]$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_beta_function(omega, lobes):
    """
    Compute the Spherical Beta function value for a given viewing direction.
    
    Args:
        omega: numpy array, shape (3,), unit viewing direction vector.
        lobes: list of dicts, each containing:
            - 's': numpy array, shape (3,), principal direction (unit vector).
            - 'alpha': float, shape parameter > 0.
            - 'beta': float, shape parameter > 0.
            
    Returns:
        float: The SB function value.
    """
    total = 0.0
    for lobe in lobes:
        s = lobe['s']
        alpha = lobe['alpha']
        beta = lobe['beta']
        
        # Dot product s . omega
        dot = np.dot(s, omega)
        
        # Compute terms
        # Note: (1 + dot) and (1 - dot) are always in [0, 2]
        term1 = (1.0 + dot) ** (alpha - 1.0)
        term2 = (1.0 - dot) ** (beta - 1.0)
        
        total += term1 * term2
        
    return total
```

**Parameters:**
- $K$: Number of lobes. Not given in the sources.
- $\alpha_k, \beta_k$: Optimized during training. Typical values are not given, but they must be $> 0$.

**Edge Cases and Pitfalls:**
- **Ill-conditioned Gradients:** Extreme values of $\alpha_k$ or $\beta_k$ generate very sharp and flat regions, causing ill-conditioned gradients and stronger sensitivity to initialization compared to Spherical Gaussians [S2, unofficial].
- **Overfitting:** The flexibility of SB increases the risk of overfitting, as observed in DBS [S1, unofficial].
- **Boundary Conditions:** The function is well-defined for all $\omega$ on the unit sphere since $1 \pm s_k \cdot \omega \ge 0$.

## Operating characteristics
- **Time Complexity:** $O(K)$ per evaluation, where $K$ is the number of lobes. This is comparable to SH of similar order but may be more expensive if $K$ is large.
- **Memory Complexity:** Stores $K$ directions (3 floats each) and $2K$ shape parameters (2 floats each). Total $5K$ floats per Gaussian.
- **Performance:** In DP-GES, using SB increased FLIP error slightly compared to GES (mainly due to higher-frequency effects being captured, which FLIP interprets as warping error) but improved view-dependent appearance [S1, unofficial]. Reverting to SH reduced FPS from 472 to 480 and PSNR from 28.11 to 27.72 on MipNeRF360, indicating SB is slightly more expensive but yields higher quality [S1, unofficial].
- **Strengths:** Can model skewed and sharper directional variation than SG or SH [S2, unofficial].
- **Limitations:** Harder to optimize, sensitive to initialization, prone to overfitting [S1, S2, unofficial].

## Background
The Spherical Beta function was introduced in the context of Gaussian-Enhanced Surfel Rendering to model higher-frequency view-dependent effects [S1, unofficial]. It extends the concept of Spherical Gaussians (SG) by allowing asymmetric lobes [S2, unofficial]. It is used in DP-GES (Depth Peeling for High-Fidelity Gaussian-Enhanced Surfel Rendering) [S1, unofficial].

## Sources
[S1] Depth Peeling for High-Fidelity Gaussian-Enhanced Surfel Rendering, https://doi.org/10.48550/arXiv.2605.25345, fetched 2026-10-30 (unofficial)
[S2] Spherical Voronoi: Directional Appearance as a Differentiable Partition of the Sphere, https://doi.org/10.48550/arXiv.2512.14180, fetched 2026-10-30 (unofficial)
