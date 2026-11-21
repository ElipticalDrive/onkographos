# Unnormalized 3D Gaussian Function

Also called: 3D Gaussian amplitude, Unnormalized Gaussian

## Overview
The Unnormalized 3D Gaussian Function is a Gaussian representation used in 3D Gaussian Splatting where the amplitude $a$ at the mean $\boldsymbol{\mu}$ equals the opacity, representing the peak capacity to block or emit light [S1]. It differs from the [Normalized 3D Gaussian Function](1064-normalized-3d-gaussian-function.md) by parameterizing the function via its peak value rather than its integral, making $a$ the intensity at the center rather than the total light capacity [S1].

## How it works
**Inputs:**
- $\mathbf{x} \in \mathbb{R}^3$: The query point in 3D space [S1].
- $a \in \mathbb{R}$: The amplitude (peak value) at the mean [S1].
- $\boldsymbol{\mu} \in \mathbb{R}^3$: The mean position (center) of the Gaussian [S1].
- $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$: The covariance matrix defining the shape and orientation [S1].

**Outputs:**
- $g \in \mathbb{R}$: The value of the unnormalized Gaussian function at $\mathbf{x}$ [S1].

**Data Structures:**
The function requires the covariance matrix $\boldsymbol{\Sigma}$ and its inverse $\boldsymbol{\Sigma}^{-1}$ [S1]. In implementation, $\boldsymbol{\Sigma}$ is often derived from a rotation matrix $\mathbf{R}$ and a scaling vector $\mathbf{s}$ such that $\boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$, where $\mathbf{S} = \text{diag}(\mathbf{s})$ [G].

**Formulas:**
The unnormalized Gaussian function $\mathcal{G}_D^\mu$ is defined as [S1]:

$$
\mathcal{G}_D^\mu(\mathbf{x}, a, \boldsymbol{\mu}, \boldsymbol{\Sigma}) = a \mathcal{I}_D(\boldsymbol{\Sigma}) \mathcal{N}_D(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma})
$$

Where $\mathcal{N}_D$ is the standard D-dimensional Gaussian probability density function (PDF) [S1]:

$$
\mathcal{N}_D(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{\Sigma}) = \frac{1}{\mathcal{I}_D(\boldsymbol{\Sigma})} e^{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})}
$$

And $\mathcal{I}_D(\boldsymbol{\Sigma})$ is the unbounded integral of the exponential part, ensuring the PDF integrates to one [S1]:

$$
\mathcal{I}_D(\boldsymbol{\Sigma}) = \sqrt{(2\pi)^D \det(\boldsymbol{\Sigma})}
$$

Substituting the PDF into the unnormalized definition, the $\mathcal{I}_D(\boldsymbol{\Sigma})$ terms cancel out, yielding the practical computation formula [S1]:

$$
\mathcal{G}_D^\mu(\mathbf{x}, a, \boldsymbol{\mu}, \boldsymbol{\Sigma}) = a e^{-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})}
$$

**Conversion to Normalized Weight:**
To convert between the unnormalized amplitude $a$ and the normalized weight $w$ (where $w$ is the integral of the function over all space), the following relationship holds [S1]:

$$
a = \frac{w}{\mathcal{I}_D(\boldsymbol{\Sigma})}
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def unnormalized_gaussian(x, a, mu, Sigma_inv):
    """
    Computes the value of the unnormalized 3D Gaussian function.
    
    Args:
        x: Query point (3,)
        a: Amplitude at the mean (scalar)
        mu: Mean position (3,)
        Sigma_inv: Inverse covariance matrix (3, 3)
        
    Returns:
        Value of the Gaussian at x
    """
    diff = [x[i] - mu[i] for i in range(3)]
    exponent = 0.0
    for i in range(3):
        row_sum = 0.0
        for j in range(3):
            row_sum += Sigma_inv[i][j] * diff[j]
        exponent += diff[i] * row_sum
    return a * math.exp(-0.5 * exponent)
```

**Edge Cases and Pitfalls:**
- **Singularity:** If $\boldsymbol{\Sigma}$ is singular (non-invertible), $\boldsymbol{\Sigma}^{-1}$ cannot be computed [G]. In practice, $\boldsymbol{\Sigma}$ is constructed from a rotation and scaling, ensuring it remains positive semi-definite [G]. Degenerate scaling (zero scale) leads to infinite amplitude for a fixed weight $w$, or zero width for a fixed amplitude $a$ [G].
- **Precision:** For points far from $\boldsymbol{\mu}$, the exponent becomes large and negative, potentially causing underflow to zero [G]. This is standard behavior for Gaussian splatting cutoffs [G].

## Subtopics

### [Normalized 3D Gaussian Function](1064-normalized-3d-gaussian-function.md)
Uses weight $w$ as the integral rather than peak amplitude. Choose when the total light capacity is the primary physical quantity of interest [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, dominated by the matrix-vector multiplication in the exponent [G].
- **Memory Complexity:** $O(1)$ auxiliary storage per Gaussian for the inverse covariance [G].
- **Strengths:** Directly parameterizes the peak intensity, which aligns with how opacity is often interpreted in rendering (peak blocking/emission) [S1].
- **Limitations:** The value at the mean depends on the spread of the Gaussian; a wider Gaussian with the same peak amplitude $a$ contributes less total energy than a narrow one [G]. This makes $a$ less intuitive for conservation of energy compared to $w$ [G].

## Background
The unnormalized Gaussian function is defined in [S1] as a Gaussian representation used in 3D Gaussian Splatting where the amplitude at the mean equals the opacity [S1]. It serves as a standard mathematical primitive for spatial distribution modeling in splatting-based rendering pipelines [G].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
