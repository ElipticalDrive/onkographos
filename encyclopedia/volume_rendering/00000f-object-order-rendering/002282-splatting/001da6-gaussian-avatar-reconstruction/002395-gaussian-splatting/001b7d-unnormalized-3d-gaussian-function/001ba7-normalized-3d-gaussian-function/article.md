# Normalized 3D Gaussian Function

Also called: Normal Distribution Helper, Normalized Gaussian, Probability Density Function Gaussian

## Overview
The Normalized 3D Gaussian Function is a probability density function scaled by a weight parameter $w$ to model the extinction coefficient in volume rendering [S1]. It serves as the fundamental building block for EWA splatting and Gaussian mixture models [S1]. Unlike the [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md), where the amplitude at the mean equals the opacity, the normalized variant integrates to $w$ over all space, making $w$ the total capacity to block or emit light rather than the peak intensity [S1].

## How it works
**Inputs:**
- $\mathbf{x} \in \mathbb{R}^3$: The query point in 3D space [S1].
- $\boldsymbol{\mu} \in \mathbb{R}^3$: The mean (center) of the Gaussian [S1].
- $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$: The covariance matrix defining the shape and orientation [S1].
- $w \in \mathbb{R}$: The weight parameter representing the total integral of the function [S1].

**Outputs:**
- $g \in \mathbb{R}$: The value of the normalized 3D Gaussian function at $\mathbf{x}$, representing the local extinction coefficient [S1].

**Data Structures:**
- $\boldsymbol{\Sigma}$: A $3 \times 3$ symmetric positive-definite matrix [S1]. In implementation, it is often stored as a $3 \times 3$ array or decomposed (e.g., via Cholesky or SVD) to ensure positive definiteness during optimization [G].
- $\boldsymbol{\mu}$: A 3-element vector [S1].
- $w$: A scalar [S1].

**Formulas:**
The normalized 3D Gaussian function $\mathcal{G}_3^n$ is defined as the product of the weight $w$ and the standard multivariate normal PDF $\mathcal{N}_3$:

$$
\mathcal{G}_3^n(\mathbf{x}, w, \boldsymbol{\mu}, \boldsymbol{\Sigma}) = w \cdot \mathcal{N}_3(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma})
$$

The PDF $\mathcal{N}_3$ is given by:

$$
\mathcal{N}_3(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = \frac{1}{\mathcal{I}_3(\boldsymbol{\Sigma})} e^{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})}
$$

The normalization factor $\mathcal{I}_3(\boldsymbol{\Sigma})$ ensures the integral of the PDF over $\mathbb{R}^3$ is 1:

$$
\mathcal{I}_3(\boldsymbol{\Sigma}) = \sqrt{(2\pi)^3 \det(\boldsymbol{\Sigma})}
$$

Combining these, the explicit formula for the normalized Gaussian is:

$$
\mathcal{G}_3^n(\mathbf{x}, w, \boldsymbol{\mu}, \boldsymbol{\Sigma}) = \frac{w}{\sqrt{(2\pi)^3 \det(\boldsymbol{\Sigma})}} e^{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})}
$$

**Parameters:**
- $\pi$: The mathematical constant Pi [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalized_3d_gaussian(x, mu, Sigma, w):
    """
    Computes the value of a normalized 3D Gaussian function at point x.
    
    Args:
        x: Query point (list/tuple of 3 floats)
        mu: Mean position (list/tuple of 3 floats)
        Sigma: Covariance matrix (list of 3 lists, each 3 floats)
        w: Weight (total integral)
        
    Returns:
        The extinction coefficient value at x.
    """
    # 1. Compute the difference vector
    diff = [x[i] - mu[i] for i in range(3)]
    
    # 2. Compute the 3x3 determinant of Sigma
    det = (Sigma[0][0] * (Sigma[1][1]*Sigma[2][2] - Sigma[1][2]*Sigma[2][1])
         - Sigma[0][1] * (Sigma[1][0]*Sigma[2][2] - Sigma[1][2]*Sigma[2][0])
         + Sigma[0][2] * (Sigma[1][0]*Sigma[2][1] - Sigma[1][1]*Sigma[2][0]))
         
    # Guard against degenerate or negative determinant
    if det <= 0:
        return 0.0
        
    # 3. Compute the 3x3 inverse (adjugate matrix divided by determinant)
    inv00 = (Sigma[1][1]*Sigma[2][2] - Sigma[1][2]*Sigma[2][1]) / det
    inv01 = (Sigma[0][2]*Sigma[2][1] - Sigma[0][1]*Sigma[2][2]) / det
    inv02 = (Sigma[0][1]*Sigma[1][2] - Sigma[0][2]*Sigma[1][1]) / det
    inv10 = (Sigma[1][2]*Sigma[2][0] - Sigma[1][0]*Sigma[2][2]) / det
    inv11 = (Sigma[0][0]*Sigma[2][2] - Sigma[0][2]*Sigma[2][0]) / det
    inv12 = (Sigma[0][2]*Sigma[1][0] - Sigma[0][0]*Sigma[1][2]) / det
    inv20 = (Sigma[1][0]*Sigma[2][1] - Sigma[1][1]*Sigma[2][0]) / det
    inv21 = (Sigma[0][1]*Sigma[2][0] - Sigma[0][0]*Sigma[2][1]) / det
    inv22 = (Sigma[0][0]*Sigma[1][1] - Sigma[0][1]*Sigma[1][0]) / det
    
    # 4. Compute the quadratic form: (x - mu)^T * Sigma^-1 * (x - mu)
    q = (diff[0]*(inv00*diff[0] + inv01*diff[1] + inv02*diff[2]) +
         diff[1]*(inv10*diff[0] + inv11*diff[1] + inv12*diff[2]) +
         diff[2]*(inv20*diff[0] + inv21*diff[1] + inv22*diff[2]))
         
    exponent_arg = -0.5 * q
    
    # 5. Compute the normalization factor: sqrt((2*pi)^3 * det(Sigma))
    normalization_factor = math.sqrt((2 * math.pi)**3 * det)
    
    # Guard against division by zero
    if normalization_factor == 0:
        return 0.0
        
    # 6. Compute the final value
    return (w / normalization_factor) * math.exp(exponent_arg)
```

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If $\det(\boldsymbol{\Sigma}) = 0$ (singular matrix), the normalization factor is zero, leading to division by zero; a guard returns 0.0 [G].
- **Positive Definiteness:** $\boldsymbol{\Sigma}$ must remain positive definite. If optimization causes it to become indefinite, the quadratic form may yield positive exponents, causing the Gaussian to explode to infinity [G].
- **Precision:** The exponential term can underflow to zero for points far from $\boldsymbol{\mu}$. Culling thresholds are typically applied in rendering to ignore Gaussians with negligible contribution [G].

## Subtopics

### [Gaussian Interpolation](2347-gaussian-interpolation.md)
Gaussian Interpolation uses the Gaussian function as an interpolation kernel to reconstruct signals from discrete samples. It is distinguished from other filters by being the only filter that is both separable and rotationally symmetric. This unique property allows it to integrate one dimension out of the kernel, enabling 2D reconstruction instead of 3D, which is particularly advantageous in point-tossing approaches like splatting.

### Gaussian Mixture Extinction Model
The Gaussian Mixture Extinction Model represents the total extinction function $f(\mathbf{x})$ as a sum of $N$ normalized 3D Gaussians, each potentially having a view-dependent radiance $c_i(\mathcal{R})$ [S1]. The extinction function is:
$$
\mathbf{f}(\mathbf{x}) = \sum_{i=0}^N \mathcal{G}_3^n(\mathbf{x}, w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)
$$
The radiance for a ray $\mathcal{R}$ is:
$$
c(\mathcal{R}, t) = \sum_{i=0}^N c_i(\mathcal{R}) \mathcal{G}_3^n(\mathcal{R}(t), w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)
$$
The final pixel intensity $I(\mathbf{p})$ is obtained by integrating along the ray $\mathcal{R}$ from $t=0$ to $\infty$:
$$
I(\mathbf{p}) = \int_0^\infty c(\mathcal{R}, t) f(\mathcal{R}(t)) e^{-\int_0^t f(\mathcal{R}(\tau)) \, d\tau} \, dt
$$
This formulation models both emission/in-scattering (via $c_i$) and attenuation (via the exponential term) [S1].

## Operating characteristics
Not given in the sources.

## Background
The normalized 3D Gaussian function originates from the field of volume rendering and splatting [S1]. It is explicitly defined as the basic building block of EWA (Elliptical Weighted Average) splatting [S1]. The formulation was later adopted and adapted for 3D Gaussian Splatting [S1]. The distinction between normalized and unnormalized Gaussians was formalized to clarify the relationship between the weight parameter $w$ and the amplitude $a$ (opacity) [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
