# Adaptive Monte-Carlo SH Projection

Also called: Adaptive Sampling SH, Green's SH projection, Modified Monte-Carlo Integration, SH Environment Projection via Monte Carlo Integration, SH lighting projection

## Overview
Adaptive Monte-Carlo SH Projection computes Spherical Harmonic (SH) coefficients for an arbitrary spherical environment function by integrating the product of the environment and SH basis functions over a domain that adapts to local material properties [S1][S2, unofficial]. It distinguishes itself from standard SH projection by dynamically adjusting the integration domain from a full sphere to a hemisphere (plus a transitional ring) based on voxel transparency and gradient magnitude, thereby optimizing the approximation for both translucent volumes and opaque surface-like boundaries [S1].

## How it works
**Inputs:**
- Spherical environment function $h(S)$, where $S$ is a direction on the unit sphere.
- Local material properties at position $\vec{x}$:
  - Transparency $\tau(f(\vec{x}))$ assigned via a transfer function.
  - Gradient length $|\nabla \tau(f(\vec{x}))|$ depicting medium heterogeneity [S1].
- Number of random samples $n$.
- Order of SH basis functions (determining the number of coefficients $c_l^m$).

**Outputs:**
- SH coefficients $\hat{c}_l^m$ for the local environment.

**Data Structures:**
- Random sample set $X = \{x_1, ..., x_n\}$, where each $x_i$ is a 3D unit vector uniformly distributed on the sphere $S$ [S2, unofficial].
- SH basis functions $Y_{lm}(S)$ [S2, unofficial].

**Parameters:**
- Below-surface angle $\alpha$: A parameter proportional to voxel transparency, lying between $0$ and $\pi/2$ [S1].
  - $\alpha = 0$ corresponds to an opaque, surface-like boundary (hemisphere only).
  - $\alpha = \pi/2$ corresponds to high transparency (full sphere) [S1].
- Threshold angle for sampling: $\pi/2 + \alpha$ [S1].

**Method Steps:**
1. **Determine Integration Domain:**
   - If $\tau(f(\vec{x}))$ is high and $|\nabla \tau(f(\vec{x}))|$ is small, assume the whole unit sphere affects illumination [S1].
   - If $\tau(f(\vec{x}))$ is low (opaque) and $|\nabla \tau(f(\vec{x}))|$ is large, assume only the front-facing hemisphere affects illumination [S1].
   - For intermediate cases, use the angle $\alpha$ to define a transitional region [S1].

2. **Define Probability Density Function (PDF):**
   - The SH coefficient calculation is reformulated as the expected value $E$ of a random variable $r$ using a PDF $p(r)$ [S1]:
     $$ c_j = \int_\Omega \frac{f(s)Y_j(s)}{p(r)} ds = E(r) $$
   - The PDF normalization factor depends on $\alpha$. The base value for a hemisphere is $2\pi$. The additional ring below the surface given by $\alpha$ adds $2\pi \cdot \sin \alpha$. The total normalization factor is:
     $$ Z = 2\pi(\sin \alpha + 1) $$
   - This factor equals $2\pi$ for $\alpha=0$ (hemisphere) and $4\pi$ for $\alpha=\pi/2$ (full sphere) [S1].
   - The PDF $p(r)$ is defined such that $\int_\Omega p(r) ds = 1$ [S1].

3. **Generate Samples:**
   - Generate $n$ uniformly distributed random samples $X = \{x_1, ..., x_n\}$ on the sphere $S$ [S2, unofficial].

4. **Filter Samples:**
   - During Monte-Carlo sampling, only consider rays if their angle with the gradient $\nabla \tau(f(\vec{x}))$ is less than or equal to $\pi/2 + \alpha$ [S1].
   - Let $S_{valid}$ be the set of samples satisfying this condition.

5. **Compute Coefficients:**
   - Approximate the integral using the filtered samples. The coefficient $\hat{c}_l^m$ is calculated as:
     $$ \hat{c}_l^m = \frac{Z}{n} \sum_{i \in S_{valid}} h(x_i) Y_{lm}(x_i) $$
   - Note: Standard Monte-Carlo integration over the full sphere uses $\frac{4\pi}{n}$ [S2, unofficial]. Here, the normalization factor $Z$ adapts to the domain size, and the summation is restricted to valid samples. If the sources imply uniform sampling over the *entire* sphere but weighting by the indicator function of the valid domain, the formula above holds. If sampling is restricted to the valid domain, the PDF changes. The source [S1] states "only rays are considered," implying filtering of uniform samples. The normalization $Z$ ensures the expectation is correct over the full sphere measure if the samples are drawn from the full sphere and weighted, or if the PDF is defined over the valid domain. Given the text "setting p(r) to 1/4\pi is a good choice... incorporate both... 2\pi(sin \alpha + 1)", it implies the PDF is defined over the valid domain $\Omega_\alpha$ with constant value $1/Z$. Thus, the estimator is:
     $$ \hat{c}_l^m = \frac{1}{n} \sum_{i=1}^n \frac{h(x_i) Y_{lm}(x_i)}{p(x_i)} $$
     where $p(x_i) = 1/Z$ if $x_i$ is valid, and $0$ otherwise? No, standard MC requires $p(x) > 0$ where the integrand is non-zero. If we sample from the full sphere, $p(x) = 1/4\pi$. The integrand is $h(s)Y_j(s) \cdot \mathbb{I}(valid)$. The expectation is $\int h Y \mathbb{I} ds$. The estimator is $\frac{4\pi}{n} \sum h(x_i) Y_j(x_i) \mathbb{I}(valid)$. This matches the standard form with $4\pi$ replaced by the effective area? The source says "2\pi(sin \alpha + 1) ... is equal to 2\pi for \alpha=0 and 4\pi for \alpha=\pi/2". This suggests the normalization factor $Z$ replaces $4\pi$ in the standard estimator $\frac{4\pi}{n} \sum ...$. So:
     $$ \hat{c}_l^m = \frac{Z}{n} \sum_{i=1}^n h(x_i) Y_{lm}(x_i) \cdot \mathbb{I}(\text{angle}(x_i, \nabla \tau) \le \pi/2 + \alpha) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def adaptive_monte_carlo_sh_projection(h, gradient_direction, transparency, n_samples, sh_order):
    """
    Computes SH coefficients for environment function h using adaptive Monte-Carlo integration.
    
    Args:
        h: Function returning environment value for a direction vector (3D unit vector).
        gradient_direction: Normalized gradient vector of transparency at the current voxel.
        transparency: Scalar transparency value tau(f(x)).
        n_samples: Number of random samples.
        sh_order: Order of SH basis (determines number of coefficients).
        
    Returns:
        List of SH coefficients c_l^m.
    """
    # Calculate below-surface angle alpha based on transparency
    # Source [S1]: alpha is proportional to transparency, lies between 0 and pi/2.
    # Exact mapping not specified, assume linear for implementation [G].
    alpha = transparency * (math.pi / 2)
    
    # Calculate normalization factor Z
    # Source [S1]: Z = 2 * pi * (sin(alpha) + 1)
    Z = 2 * math.pi * (math.sin(alpha) + 1)
    
    # Threshold angle for sampling
    # Source [S1]: Consider rays if angle with gradient <= pi/2 + alpha
    threshold_angle = math.pi / 2 + alpha
    
    # Precompute SH basis functions Y_lm for the sampled directions
    # This is a placeholder for the actual SH basis evaluation
    # Y_lm(s) depends on spherical coordinates of s
    sh_basis = compute_sh_basis(sh_order)
    
    coefficients = [0.0] * len(sh_basis)
    
    for _ in range(n_samples):
        # Generate uniform random sample on sphere
        # Source [S2, unofficial]: X = x1, ..., xn uniformly distributed on S
        x = random_uniform_sphere_sample()
        
        # Calculate angle between sample direction and gradient
        # Source [S1]: angle with nabla tau
        cos_angle = dot_product(x, gradient_direction)
        angle = math.acos(max(-1.0, min(1.0, cos_angle))) # Clamp for precision
        
        # Check if sample is within valid domain
        if angle <= threshold_angle:
            # Evaluate environment function
            h_val = h(x)
            
            # Update SH coefficients
            for idx, y_lm in enumerate(sh_basis):
                # Y_lm is evaluated at x
                y_val = y_lm(x)
                coefficients[idx] += h_val * y_val
    
    # Normalize by Z/n
    # Source [S2, unofficial]: c_l^m = (4pi/n) sum ...
    # Source [S1]: Adapted to Z
    for idx in range(len(coefficients)):
        coefficients[idx] = (Z / n_samples) * coefficients[idx]
        
    return coefficients

def random_uniform_sphere_sample():
    """Generates a uniformly distributed random point on the unit sphere."""
    # Standard method: uniform phi, cos(theta) uniform
    u = random.random()
    v = random.random()
    theta = 2 * math.pi * u
    phi = math.acos(2 * v - 1)
    x = math.sin(phi) * math.cos(theta)
    y = math.sin(phi) * math.sin(theta)
    z = math.cos(phi)
    return (x, y, z)

def dot_product(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def compute_sh_basis(order):
    """
    Returns a list of functions representing SH basis functions Y_lm.
    Each function takes a direction vector and returns the basis value.
    Implementation details omitted as they are standard SH math [G].
    """
    # Placeholder for actual SH basis implementation
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** Clamping the dot product to $[-1, 1]$ before `acos` is necessary to avoid NaNs due to floating-point errors [G].
- **Degenerate Input:** If the gradient is zero, the direction is undefined. A default direction (e.g., view vector) or full sphere sampling should be used [G].
- **Boundary Conditions:** The transition between hemisphere and full sphere is continuous via $\alpha$, but sharp changes in transparency between adjacent voxels may cause visual artifacts if not smoothed [S1].

## Subtopics

### Full Sphere SH Projection
Assumes the entire sphere contributes. Used for highly translucent media. Link: [Spherical Harmonic Projection](11243-spherical-harmonic-projection.md).

### Hemisphere SH Projection
Assumes only the front-facing hemisphere contributes. Used for opaque surfaces. Link: [Spherical Harmonic Projection](11243-spherical-harmonic-projection.md).

## Operating characteristics
- **Time Complexity:** $O(n \cdot L^2)$ per voxel, where $n$ is the number of samples and $L$ is the SH order (number of coefficients). The filtering step adds a constant overhead per sample [S1][S2, unofficial].
- **Memory Complexity:** $O(L^2)$ to store the SH coefficients per voxel. The random samples can be generated on the fly, requiring $O(1)$ additional memory [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with the number of samples $n$ and the square of the SH order. The adaptive nature does not significantly change the asymptotic complexity but may reduce the effective number of samples needed for convergence in opaque regions [S1].
- **Strengths:** Adapts to local material properties, providing accurate lighting for both translucent and opaque regions without requiring separate passes or assumptions [S1].
- **Limitations:** Requires evaluation of the environment function $h$ at random points, which can be expensive if $h$ is complex. The approximation error depends on the number of samples and the smoothness of $h$ [S2, unofficial].
- **When to use:** When rendering direct volume data with varying transparency and when environmental lighting (e.g., color bleeding) needs to be approximated efficiently [S1].
- **When not:** For real-time applications with very high SH orders or extremely noisy environment functions where deterministic integration might be preferred [S2, unofficial].

## Background
- **Origin:** The method was introduced by Lindemann and Ropinski in "Advanced Light Material Interaction for Direct Volume Rendering" (2010) [S1]. It builds on standard SH projection techniques [S2, unofficial] and Monte-Carlo integration methods [Green, 2003] cited in [S2, unofficial].
- **History:** The use of SH for lighting in graphics dates back to PRT (Precomputed Radiance Transfer) [S2, unofficial]. The adaptive approach specifically addresses the challenge of volume rendering where material properties vary spatially [S1].
- **Usage:** Used in direct volume rendering to compute color bleeding effects and diffuse shadowed radiance transfer [S1].

## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
[S2] PDF A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbcm.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
