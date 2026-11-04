# SDF-based Opacity Derivation via Logistic CDF

Also called: Alpha compositing via logistic CDF, Alpha computation, Discrete opacity computation from SDF, Eqn. 13, NeuS Opacity Derivation, NeuS Opacity Formula

## Overview
This method computes the discrete opacity $\alpha_i$ for a ray segment $[t_i, t_{i+1}]$ in volume rendering by integrating a density function derived from a signed distance function (SDF) and the logistic distribution's probability density function (PDF) [S1, unofficial]. It solves the problem of correctly modeling surface opacity by ensuring the weight function peaks at the surface intersection, thereby avoiding the bias inherent in naive SDF-to-density mappings [S1, unofficial]. Unlike [Piecewise Density Approximation for Reparameterized Sampling](1139-piecewise-density-approximation-for-reparameterized-sampling.md) which approximates density piecewise, this method analytically integrates the logistic CDF to ensure the opacity accumulates only when the ray enters the surface (SDF decreasing) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $f(\mathbf{x})$: The signed distance function, where $f(\mathbf{x}) > 0$ outside the surface and $f(\mathbf{x}) < 0$ inside [S1, unofficial].
    *   $\mathbf{p}(t)$: The ray position at distance $t$, defined as $\mathbf{o} + t\mathbf{v}$, where $\mathbf{o}$ is the origin and $\mathbf{v}$ is the unit direction vector.
    *   $s$: The steepness parameter of the logistic distribution.
    *   $t_i, t_{i+1}$: The start and end distances of the current ray segment.
*   **Outputs:**
    *   $\alpha_i$: The opacity (alpha value) for the segment $[t_i, t_{i+1}]$.

**Data Structures:**
No specific complex data structures are required beyond scalar values for the SDF evaluations and the steepness parameter. The method operates on per-segment basis.

**Formulas:**
The cumulative distribution function (CDF) of the logistic distribution with steepness $s$ is defined as [S1, unofficial]:
$$ \Phi_s(x) = \frac{1}{1 + e^{-sx}} $$

The discrete opacity $\alpha_i$ is computed as [S1, unofficial]:
$$ \alpha_i = \max \left( \frac{\Phi_s(f(\mathbf{p}(t_i))) - \Phi_s(f(\mathbf{p}(t_{i+1})))}{\Phi_s(f(\mathbf{p}(t_i)))}, 0 \right) $$

**Parameters:**
*   $s$: Steepness of the logistic CDF. Typical values are not explicitly given in the sources, but it controls the sharpness of the surface transition [S1, unofficial].

**Step-by-Step Method:**
1.  Evaluate the SDF at the start of the segment: $d_i = f(\mathbf{p}(t_i))$.
2.  Evaluate the SDF at the end of the segment: $d_{i+1} = f(\mathbf{p}(t_{i+1}))$.
3.  Compute the logistic CDF at the start: $C_i = \Phi_s(d_i) = \frac{1}{1 + e^{-s d_i}}$.
4.  Compute the logistic CDF at the end: $C_{i+1} = \Phi_s(d_{i+1}) = \frac{1}{1 + e^{-s d_{i+1}}}$.
5.  Calculate the numerator: $N = C_i - C_{i+1}$.
6.  Calculate the opacity: $\alpha_i = \max(N / C_i, 0)$.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $C_i = 0$ (which occurs if $d_i \to -\infty$), the formula is undefined. In practice, $C_i$ should be clamped to a small epsilon or the segment should be skipped if the ray is deep inside the surface where opacity is already accumulated [G].
*   **Numerical Noise:** The $\max(\cdot, 0)$ term handles cases where numerical noise causes $C_{i+1} > C_i$ (i.e., SDF increasing) or results in negative opacity [S1, unofficial].
*   **Exiting Surface:** When the ray exits the surface, the SDF increases ($d_{i+1} > d_i$), leading to $C_{i+1} > C_i$ and thus $\alpha_i = 0$, which is correct as no new surface opacity is accumulated [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def logistic_cdf(x: float, s: float) -> float:
    """Compute the CDF of the logistic distribution."""
    # Avoid overflow in exp for large negative s*x
    if s * x > 500:
        return 1.0
    if s * x < -500:
        return 0.0
    return 1.0 / (1.0 + math.exp(-s * x))

def compute_opacity(
    sdf_start: float,
    sdf_end: float,
    s: float
) -> float:
    """
    Compute discrete opacity for a ray segment based on SDF values.
    
    Args:
        sdf_start: SDF value at t_i
        sdf_end: SDF value at t_{i+1}
        s: Steepness parameter of the logistic distribution
        
    Returns:
        Opacity alpha_i for the segment
    """
    # Compute logistic CDF values
    cdf_start = logistic_cdf(sdf_start, s)
    cdf_end = logistic_cdf(sdf_end, s)
    
    # Avoid division by zero
    if cdf_start < 1e-8:
        return 0.0
        
    # Compute opacity
    numerator = cdf_start - cdf_end
    alpha = numerator / cdf_start
    
    # Ensure non-negative opacity
    return max(alpha, 0.0)
```

## Subtopics

### [Coverage-based Opacity Mapping](4519-coverage-based-opacity-mapping.md)
Coverage-based Opacity Mapping is a method to convert accumulated unnormalized Gaussian weights into per-layer coverage opacities for surface compositing. It solves the problem of jagged, aliased edges and incorrect blending in surface splatting by replacing threshold-based clamping with a smooth, monotonically increasing mapping. Unlike the single-layer ternary depth test which produces hard transitions, this method yields smooth opacity falloff against the background and gradual transitions in overlap regions. It builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md) by handling the final compositing of surface layers rather than deriving initial opacity from signed distances.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ray segment, as it involves a constant number of arithmetic operations and one exponential evaluation per segment.
*   **Memory Complexity:** $O(1)$ auxiliary memory per segment.
*   **Strengths:** Analytically integrates the density function, ensuring unbiased surface representation. It naturally handles the entering/exiting logic of the surface without explicit thresholding.
*   **Limitations:** Requires the SDF to be differentiable or at least continuous. The choice of steepness $s$ affects the sharpness of the surface; too low $s$ results in blurry surfaces, too high $s$ may cause numerical instability [S1, unofficial].
*   **When to Use:** When rendering neural implicit surfaces (like NeuS) where accurate surface opacity is critical and the SDF is available.
*   **When Not to Use:** When the SDF is noisy or discontinuous, as the derivative-based density assumption may fail.

## Background
The method originates from the NeuS paper, which introduced learning neural implicit surfaces by volume rendering [S1, unofficial]. It was designed to address the bias in previous SDF-based volume rendering methods that used naive density mappings. The derivation relies on the properties of the logistic distribution to model the surface density field.

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
