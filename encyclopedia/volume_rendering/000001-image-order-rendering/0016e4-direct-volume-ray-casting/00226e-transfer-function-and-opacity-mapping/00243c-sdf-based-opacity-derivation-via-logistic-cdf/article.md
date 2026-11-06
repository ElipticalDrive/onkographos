# SDF-based Opacity Derivation via Logistic CDF

Also called: Alpha compositing via logistic CDF, Alpha computation, Discrete opacity computation from SDF, Eqn. 13, NeuS Opacity Derivation, NeuS Opacity Formula

## Overview
This method computes the discrete opacity $\alpha_i$ for a ray segment $[t_i, t_{i+1}]$ in volume rendering by integrating a density function derived from a signed distance function (SDF) and the logistic distribution's cumulative distribution function (CDF) [S1, unofficial]. It solves the problem of correctly modeling surface opacity by ensuring the weight function peaks at the surface intersection, thereby avoiding the bias inherent in naive SDF-to-density mappings [S1, unofficial]. Unlike [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md), which relies on gradient magnitude for opacity, this method uses the geometric properties of the SDF relative to the ray direction to define a physically motivated density profile [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $f(\mathbf{x})$: The signed distance function, where $f(\mathbf{x}) > 0$ outside the surface and $f(\mathbf{x}) < 0$ inside [S1, unofficial].
    *   $\mathbf{p}(t)$: The ray position at distance $t$, defined as $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$, where $\mathbf{o}$ is the ray origin and $\mathbf{v}$ is the unit ray direction [S1, unofficial].
    *   $s$: The steepness parameter of the logistic distribution [S1, unofficial].
    *   $t_i, t_{i+1}$: The start and end distances of the current sample segment along the ray [S1, unofficial].
*   **Outputs:**
    *   $\alpha_i$: The opacity of the segment $[t_i, t_{i+1}]$, bounded in $[0, 1]$ [S1, unofficial].

**Data Structures:**
*   No specific complex data structures are required beyond standard scalar values for the SDF, ray parameters, and the steepness parameter. The SDF $f$ is typically queried from a neural network or a voxel grid [S1, unofficial].

**Steps:**
1.  **Evaluate SDF:** Compute the signed distance values at the segment boundaries: $d_i = f(\mathbf{p}(t_i))$ and $d_{i+1} = f(\mathbf{p}(t_{i+1}))$ [S1, unofficial].
2.  **Compute Logistic CDF:** Calculate the logistic CDF values at these distances. The CDF is defined as $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1, unofficial].
    *   Let $C_i = \Phi_s(d_i)$.
    *   Let $C_{i+1} = \Phi_s(d_{i+1})$.
3.  **Compute Opacity:** Calculate the opacity using the derived formula [S1, unofficial]:
    $$ \alpha_i = \max \left( \frac{C_i - C_{i+1}}{C_i}, 0 \right) $$
    This formula implicitly handles the two geometric cases:
    *   **Entering Surface:** If the ray is entering the surface, the SDF is decreasing ($d_i > d_{i+1}$). The density $\rho(t)$ is positive, and the opacity is computed as the normalized drop in the CDF [S1, unofficial].
    *   **Exiting Surface:** If the ray is exiting the surface, the SDF is increasing ($d_i < d_{i+1}$). The density $\rho(t)$ is defined as 0, resulting in $\alpha_i = 0$ [S1, unofficial].
    *   **Numerical Stability:** The $\max(\cdot, 0)$ term handles numerical noise or ambiguous gradients where the SDF might slightly increase when it should decrease [S1, unofficial].

**Formulas:**
*   **Logistic PDF:** $\phi_s(x) = \frac{s e^{-sx}}{(1 + e^{-sx})^2}$ [S1, unofficial].
*   **Logistic CDF:** $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1, unofficial].
*   **Density Function:** The opaque density function $\rho(t)$ is defined as [S1, unofficial]:
    $$ \rho(t) = \max \left( \frac{-\frac{d\Phi_s}{dt}(f(\mathbf{p}(t)))}{\Phi_s(f(\mathbf{p}(t)))}, 0 \right) = \max \left( \frac{-(\nabla f(\mathbf{p}(t)) \cdot \mathbf{v})\phi_s(f(\mathbf{p}(t)))}{\Phi_s(f(\mathbf{p}(t)))}, 0 \right) $$
    where $\nabla f$ is the gradient of the SDF and $\mathbf{v}$ is the ray direction [S1, unofficial].
*   **Opacity Integral:** The opacity is derived from the integral of the density [S1, unofficial]:
    $$ \alpha_i = 1 - \exp \left( - \int_{t_i}^{t_{i+1}} \rho(t) dt \right) $$
    Substituting the density definition leads to the closed-form solution [S1, unofficial]:
    $$ \alpha_i = \frac{\Phi_s(f(\mathbf{p}(t_i))) - \Phi_s(f(\mathbf{p}(t_{i+1})))}{\Phi_s(f(\mathbf{p}(t_i)))} $$
    (clamped to be non-negative) [S1, unofficial].

**Parameters:**
*   $s$: Steepness of the logistic distribution. Typical values are not explicitly given in the sources, but it controls the sharpness of the surface representation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_opacity_logistic_cdf(f, p, t_i, t_i_plus_1, s):
    """
    Computes opacity for a ray segment using SDF-based logistic CDF.
    
    Args:
        f: Function that returns the signed distance value for a point x.
        p: Function that returns the ray position for a distance t.
        t_i: Start distance of the segment.
        t_i_plus_1: End distance of the segment.
        s: Steepness parameter of the logistic distribution.
        
    Returns:
        alpha_i: Opacity of the segment.
    """
    # 1. Evaluate SDF at segment boundaries
    d_i = f(p(t_i))
    d_i_plus_1 = f(p(t_i_plus_1))
    
    # 2. Compute Logistic CDF values
    # Phi_s(x) = (1 + exp(-s * x))^-1
    c_i = 1.0 / (1.0 + math.exp(-s * d_i))
    c_i_plus_1 = 1.0 / (1.0 + math.exp(-s * d_i_plus_1))
    
    # 3. Compute opacity
    # Avoid division by zero if c_i is 0 (should not happen for finite s and x)
    if c_i == 0:
        return 0.0
        
    alpha_i = (c_i - c_i_plus_1) / c_i
    
    # Clamp to [0, 1] to handle numerical noise
    return max(0.0, min(1.0, alpha_i))
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\Phi_s(f(\mathbf{p}(t_i)))$ is zero, the formula is undefined. This occurs if $f(\mathbf{p}(t_i)) \to -\infty$ (deep inside the surface) for finite $s$. In practice, SDF values are bounded or clamped [S1, unofficial].
*   **Numerical Noise:** The $\max(\cdot, 0)$ is crucial because numerical errors in SDF evaluation or ray marching might cause $d_{i+1} > d_i$ even when the ray is technically entering the surface, leading to negative opacity without the clamp [S1, unofficial].
*   **Steepness $s$:** If $s$ is too large, the logistic CDF approaches a step function, which may cause numerical instability in the gradient-based density calculation if gradients are used explicitly (though the closed-form CDF difference is more stable) [S1, unofficial].

## Subtopics

### [Coverage-based Opacity Mapping](4519-coverage-based-opacity-mapping.md)
Coverage-based Opacity Mapping is a method to convert accumulated unnormalized Gaussian weights into per-layer coverage opacities for surface compositing. It solves the problem of jagged, aliased edges and incorrect blending in surface splatting by replacing threshold-based clamping with a smooth, monotonically increasing mapping. Unlike the single-layer ternary depth test which produces hard transitions, this method yields smooth opacity falloff against the background and gradual transitions in overlap regions. It builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md) by handling the final compositing of surface layers rather than deriving initial opacity from signed distances.

### NeUDF Discretization (Alpha Compositing)
NeUDF Discretization is a method to discretize the continuous weight function for volume rendering by dividing the sample ray into bins and computing opacity ($\alpha_i$) for each bin based on the UDF values at the bin boundaries [S2]. It differs from the standard NeuS formulation by using an unsigned distance function (UDF) and a modified alpha calculation that uses the maximum and minimum of the transformed UDF values at the segment boundaries to ensure stability [S2]. The base formula is $\alpha_i = \frac{|\varsigma_r \circ \Psi \circ p(t_i) - \varsigma_r \circ \Psi \circ p(t_{i+1})|}{\varsigma_r \circ \Psi \circ p(t_i)}$, where $\varsigma_r \circ \Psi$ represents the transformed UDF [S2]. A modified version is also used: $\alpha_i = \frac{\varsigma_i^{\max} - \varsigma_i^{\min}}{\varsigma_i^{\max}}$, where $\varsigma_i^{\max}$ and $\varsigma_i^{\min}$ are the maximum and minimum of the set $\{\varsigma_r \circ \Psi \circ p(t_i), \varsigma_r \circ \Psi \circ p(t_{i+1})\}$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample segment, assuming the SDF evaluation is $O(1)$ (e.g., neural network forward pass or grid lookup) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per segment [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of samples along the ray [S1, unofficial].
*   **Strengths:** Provides a theoretically grounded density model that peaks at the surface, reducing bias compared to naive SDF mappings [S1, unofficial]. It is differentiable, enabling gradient-based optimization of the SDF [S1, unofficial].
*   **Limitations:** Requires the SDF to be differentiable (or approximated as such) [S1, unofficial]. Performance depends on the efficiency of the SDF evaluation [S1, unofficial].
*   **When to use:** When rendering implicit surfaces defined by an SDF and high-quality, bias-free surface opacity is required [S1, unofficial].
*   **When not to use:** When the SDF is not available or not differentiable, or when simpler, non-geometric opacity transfer functions are sufficient [S1, unofficial].

## Background
The method was introduced in the paper "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction" [S1, unofficial]. It is a core component of the NeuS framework, which enables the learning of neural implicit surfaces from multi-view images by combining volume rendering with signed distance functions [S1, unofficial]. The derivation relies on the logistic distribution to model the density profile around the zero-level set of the SDF [S1, unofficial]. A variant using Unsigned Distance Functions (UDF) was later proposed in "NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering" [S2].

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
