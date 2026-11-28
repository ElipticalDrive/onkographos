# Normalized 3D Gaussian Function

Also called: Normal Distribution Helper, Normalized Gaussian, Probability Density Function Gaussian

## Overview
The Normalized 3D Gaussian Function is a probability density function (PDF) scaled by a weight parameter to model the extinction coefficient in volume rendering [S1]. It serves as the fundamental building block for EWA splatting and Gaussian mixture models, ensuring that the total integrated capacity of a primitive to block or emit light is explicitly controlled by its weight $w$ rather than its peak amplitude [S1]. Unlike the [Unnormalized 3D Gaussian Function](1026-unnormalized-3d-gaussian-function.md), where the amplitude at the mean equals opacity, the normalized variant separates the shape normalization from the intensity scaling, making it suitable for accurate volumetric integration where the integral of the density field must equal a specific physical quantity [S1].

## How it works
**Inputs:**
- $\mathbf{x} \in \mathbb{R}^3$: The 3D position vector at which to evaluate the function [S1].
- $w \in \mathbb{R}$: The weight parameter, representing the total capacity to block/emit light, equivalent to the unbounded integral of the function [S1].
- $\boldsymbol{\mu} \in \mathbb{R}^3$: The mean position (center) of the Gaussian [S1].
- $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$: The covariance matrix defining the shape and orientation of the Gaussian [S1].

**Outputs:**
- $g \in \mathbb{R}$: The value of the normalized 3D Gaussian function at $\mathbf{x}$ [S1].

**Data Structures:**
- Exact storage layouts and parameterization schemes are Not given in the sources.

**Formulas:**
The normalized 3D Gaussian function $\mathcal{G}_3^n$ is defined as the product of the weight $w$ and the standard multivariate normal PDF $\mathcal{N}_3$ [S1]:

$$
\mathcal{G}_3^n(\mathbf{x}, w, \boldsymbol{\mu}, \boldsymbol{\Sigma}) = w \mathcal{N}_3(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma})
$$

where the PDF $\mathcal{N}_3$ is [S1]:

$$
\mathcal{N}_3(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = \frac{1}{\mathcal{I}_3(\boldsymbol{\Sigma})} e^{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})}
$$

The normalization factor $\mathcal{I}_3(\boldsymbol{\Sigma})$ ensures that $\mathcal{N}_3$ integrates to one over $\mathbb{R}^3$ [S1]:

$$
\mathcal{I}_3(\boldsymbol{\Sigma}) = \sqrt{(2\pi)^3 \det(\boldsymbol{\Sigma})}
$$

Consequently, the weight $w$ is equivalent to the unbounded integral of the function [S1]:

$$
\int_{\mathbb{R}^3} \mathcal{G}_3^n(\mathbf{x}, w, \boldsymbol{\mu}, \boldsymbol{\Sigma}) \, d\mathbf{x} = w
$$

**Steps:**
1. Compute the difference vector $\mathbf{d} = \mathbf{x} - \boldsymbol{\mu}$ [S1].
2. Compute the inverse covariance matrix $\boldsymbol{\Sigma}^{-1}$ [S1]. If $\boldsymbol{\Sigma}$ is singular or near-singular, add a small epsilon to the diagonal or use a pseudo-inverse [G].
3. Compute the quadratic form $Q = \mathbf{d}^T \boldsymbol{\Sigma}^{-1} \mathbf{d}$ [S1].
4. Compute the determinant $\det(\boldsymbol{\Sigma})$ [S1].
5. Compute the normalization constant $Z = \sqrt{(2\pi)^3 \det(\boldsymbol{\Sigma})}$ [S1].
6. Compute the exponential term $E = \exp(-0.5 \cdot Q)$ [S1].
7. Compute the PDF value $P = E / Z$ [S1].
8. Return the result $g = w \cdot P$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalized_3d_gaussian(x, w, mu, Sigma):
    """
    Evaluates the normalized 3D Gaussian function.
    
    Args:
        x: list/tuple of 3 floats, position vector.
        w: float, weight parameter (total integral).
        mu: list/tuple of 3 floats, mean vector.
        Sigma: list of lists, 3x3 covariance matrix.
        
    Returns:
        float: Value of the normalized Gaussian at x.
    """
    # 1. Difference vector
    d = [x[i] - mu[i] for i in range(3)]
    
    # 2. Inverse covariance matrix (3x3 cofactor method)
    a, b, c = Sigma[0][0], Sigma[0][1], Sigma[0][2]
    d_s, e, f = Sigma[1][0], Sigma[1][1], Sigma[1][2]
    g, h, i_val = Sigma[2][0], Sigma[2][1], Sigma[2][2]
    
    det = a*(e*i_val - f*h) - b*(d_s*i_val - f*g) + c*(d_s*h - e*g)
    if det == 0:
        return 0.0  # Singular matrix fallback
        
    inv_det = 1.0 / det
    inv = [
        [(e*i_val - f*h)*inv_det, (c*h - b*i_val)*inv_det, (b*f - c*e)*inv_det],
        [(f*g - d_s*i_val)*inv_det, (a*i_val - c*g)*inv_det, (c*d_s - a*f)*inv_det],
        [(d_s*h - e*g)*inv_det, (b*g - a*h)*inv_det, (a*e - b*d_s)*inv_det]
    ]
    
    # 3. Quadratic form: d^T * inv * d
    Q = 0.0
    for k in range(3):
        for l in range(3):
            Q += d[k] * inv[k][l] * d[l]
            
    # 4. Normalization constant: sqrt((2*pi)^3 * det)
    norm_factor = math.sqrt((2 * math.pi)**3 * det)
    
    if norm_factor == 0:
        return 0.0
        
    # 5. Exponential term
    E = math.exp(-0.5 * Q)
    
    # 6. PDF value and weighted result
    pdf_val = E / norm_factor
    return w * pdf_val
```

**Parameters:**
- $w$: Typical values depend on the scene's lighting and opacity. In EWA splatting, $w$ represents the total capacity to block and emit light [S1].

**Edge Cases and Pitfalls:**
- **Singular Covariance:** If $\det(\boldsymbol{\Sigma}) = 0$, the normalization factor is zero, leading to division by zero [S1]. This occurs if the Gaussian is degenerate (e.g., flattened to a plane or line in 3D) [G]. Regularization is required [G].
- **Numerical Stability:** For very large quadratic forms $Q$, $\exp(-0.5 Q)$ underflows to 0, which is acceptable as the contribution is negligible [G]. For very small determinants, the normalization factor becomes tiny, potentially causing overflow if not handled [G].
- **Positive Definiteness:** The inverse and determinant operations assume $\boldsymbol{\Sigma}$ is positive-definite [G]. If the matrix is not SPD (e.g., due to numerical drift in optimization), the quadratic form may yield negative values, resulting in $\exp(\text{positive})$ which explodes [G].

## Subtopics

### Gaussian Mixture Extinction Model
This variant models the extinction function $f(\mathbf{x})$ as a mixture of normalized 3D Gaussians, allowing for view-dependent radiance $c_i(\mathcal{R})$ per Gaussian [S1]. It is distinguished by its use in exact volumetric integration solutions for complex scenes, where the density field is explicitly defined as a sum of normalized primitives rather than a single blob [S1].

**Definition:**
The extinction function is a sum of normalized 3D Gaussians [S1]:
$$
\mathbf{f}(\mathbf{x}) = \sum_{i=0}^N \mathcal{G}_3^n(\mathbf{x}, w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)
$$

**Formula:**
The radiance along a ray $\mathcal{R}$ at distance $t$ is [S1]:
$$
c(\mathcal{R}, t) = \sum_{i=0}^N c_i(\mathcal{R}) \mathcal{G}_3^n(\mathcal{R}(t), w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i)
$$
where $c_i(\mathcal{R})$ is the radiance for ray $\mathcal{R}$ from the $i$-th Gaussian [S1].

The final pixel intensity $I(\mathbf{p})$ is computed via exact volumetric integration [S1]:
$$
I(\mathbf{p}) = \int_0^\infty \left( \sum_{i=0}^N c_i(\mathcal{R}) \mathcal{G}_3^n(\mathcal{R}(t), w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) \right) e^{-\int_0^t \sum_{i=0}^N \mathcal{G}_3^n(\mathcal{R}(\tau), w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) \, d\tau} \, dt
$$
This formulation explicitly models emission and attenuation using the normalized Gaussian mixture [S1].

## Operating characteristics
- **Time Complexity:** Evaluation of a single Gaussian is $O(1)$ since matrix inverse and determinant operate on constant-size $3 \times 3$ matrices [G]. For a mixture of $N$ Gaussians, evaluation is $O(N)$ [S1].
- **Memory Complexity:** $O(N)$ to store the parameters ($\boldsymbol{\mu}, \boldsymbol{\Sigma}, w, c_i$) for $N$ Gaussians [G].
- **Performance:** The normalization factor requires computing a determinant and inverse, which is more expensive than the unnormalized variant where these terms cancel each other out [S1]. However, it provides physically meaningful weights for integration [S1].
- **Strengths:** Ensures the total integral of the density is exactly $w$, which is crucial for accurate volumetric rendering and extinction modeling [S1]. It is the standard building block for EWA splatting and Gaussian mixture models [S1].
- **Limitations:** More computationally intensive than unnormalized Gaussians due to the normalization factor calculation [S1]. Requires careful handling of singular covariance matrices [G].
- **When to use:** When accurate volumetric integration is required, or when the total mass/optical depth of a primitive must be explicitly controlled independent of its shape [S1].
- **When not to use:** When performance is critical and approximate rendering is acceptable, or when using methods like standard 3DGS that prefer the unnormalized variant because the normalization factors cancel out during evaluation [S1].

## Background
The normalized 3D Gaussian function originates from the field of volume rendering and splatting, specifically as a building block for EWA (Elliptical Weighted Average) splatting [S1]. It is formally defined in the context of Gaussian data models for volume rendering, where it is paired with a weight parameter to model extinction [S1]. The concept is rooted in the probability density function of the multivariate normal distribution, adapted for computer graphics to represent particle densities or extinction coefficients [S1]. It is used in EWA splatting [ZPVBG02] and is a fundamental component in the theoretical underpinnings of 3D Gaussian Splatting [KKLD23], although 3DGS often employs the unnormalized variant for efficiency [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
