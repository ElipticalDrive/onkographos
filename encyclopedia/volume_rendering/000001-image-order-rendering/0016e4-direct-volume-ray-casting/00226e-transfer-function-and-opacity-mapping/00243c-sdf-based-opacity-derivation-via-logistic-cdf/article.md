# SDF-based Opacity Derivation via Logistic CDF

Also called: Alpha compositing via logistic CDF, Alpha computation, Discrete opacity computation from SDF, Eqn. 13, NeuS Opacity Derivation, NeuS Opacity Formula

## Overview
This method computes the discrete opacity $\alpha_i$ for a ray segment $[t_i, t_{i+1}]$ in volume rendering by integrating a density function derived from a Signed Distance Function (SDF) and the Cumulative Distribution Function (CDF) of the logistic distribution [S1, unofficial]. It solves the problem of correctly modeling surface opacity by ensuring the weight function peaks at the surface intersection, thereby avoiding the bias inherent in naive SDF-to-density mappings [S1, unofficial]. Unlike [Piecewise Linear Mapping and Integration](9000-piecewise-linear-mapping-and-integration.md) which uses linear approximations, this method uses the smooth, probabilistic shape of the logistic CDF to define opacity transitions [S1, unofficial].

## How it works
**Inputs:**
*   $f(\mathbf{x})$: The Signed Distance Function (SDF), where $f(\mathbf{x}) > 0$ outside the surface and $f(\mathbf{x}) < 0$ inside [S1, unofficial].
*   $\mathbf{p}(t)$: The ray position at distance $t$, defined as $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$, where $\mathbf{o}$ is the camera origin and $\mathbf{v}$ is the unit view direction [S1, unofficial].
*   $s$: The steepness parameter of the logistic distribution [S1, unofficial].
*   $t_i, t_{i+1}$: The start and end distances of the current ray segment [S1, unofficial].

**Outputs:**
*   $\alpha_i$: The opacity of the segment $[t_i, t_{i+1}]$, bounded in $[0, 1]$ [G].

**Data Structures:**
*   No specific complex data structures are required beyond scalar values for $f$, $\Phi_s$, and $\alpha_i$.

**Definitions:**
*   $\Phi_s(x)$: The CDF of the logistic distribution with steepness $s$.
    $$ \Phi_s(x) = \frac{1}{1 + e^{-sx}} $$
*   $\phi_s(x)$: The PDF of the logistic distribution (used in derivation, not directly in final opacity formula).
    $$ \phi_s(x) = \frac{s e^{-sx}}{(1 + e^{-sx})^2} $$
*   $\rho(t)$: The opaque density function along the ray [S1, unofficial].

**Steps:**
1.  Evaluate the SDF at the segment boundaries: $d_i = f(\mathbf{p}(t_i))$ and $d_{i+1} = f(\mathbf{p}(t_{i+1}))$ [S1, unofficial].
2.  Evaluate the logistic CDF at these distances: $C_i = \Phi_s(d_i)$ and $C_{i+1} = \Phi_s(d_{i+1})$ [S1, unofficial].
3.  Compute the opacity $\alpha_i$ using the following formula, which handles both entering and exiting cases via a max function to ensure non-negativity and handle numerical noise [S1, unofficial]:
    $$ \alpha_i = \max \left( \frac{C_i - C_{i+1}}{C_i}, 0 \right) $$
    Substituting the CDF definition:
    $$ \alpha_i = \max \left( \frac{\Phi_s(f(\mathbf{p}(t_i))) - \Phi_s(f(\mathbf{p}(t_{i+1})))}{\Phi_s(f(\mathbf{p}(t_i)))}, 0 \right) $$

**Derivation Logic:**
The method is derived from the continuous opacity integral $\alpha_i = 1 - \exp\left(-\int_{t_i}^{t_{i+1}} \rho(t) dt\right)$ [S1, unofficial]. The density $\rho(t)$ is defined as:
$$ \rho(t) = \max \left( \frac{-\frac{d\Phi_s}{dt}(f(\mathbf{p}(t)))}{\Phi_s(f(\mathbf{p}(t)))}, 0 \right) = \max \left( \frac{-(\nabla f(\mathbf{p}(t)) \cdot \mathbf{v})\phi_s(f(\mathbf{p}(t)))}{\Phi_s(f(\mathbf{p}(t)))}, 0 \right) $$
When the ray enters the surface (SDF decreasing, $-(\nabla f \cdot \mathbf{v}) > 0$), the integral simplifies to $-\ln(\Phi_s(f(\mathbf{p}(t_{i+1})))) + \ln(\Phi_s(f(\mathbf{p}(t_i))))$, leading to the ratio form [S1, unofficial]. When the ray exits the surface (SDF increasing), the density is defined as 0, resulting in $\alpha_i = 0$ [S1, unofficial]. The max function combines these cases [S1, unofficial].

**Parameters:**
*   $s$ (steepness): Controls the sharpness of the opacity transition. Higher values result in sharper edges. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\Phi_s(f(\mathbf{p}(t_i))) = 0$, the formula is undefined. In practice, this occurs if $f(\mathbf{p}(t_i)) \to -\infty$ (deep inside the volume). A small epsilon should be added to the denominator or the SDF clamped [G].
*   **Numerical Noise:** The max function handles cases where numerical precision causes $C_{i+1} > C_i$ in an entering segment, preventing negative opacity [S1, unofficial].
*   **Exiting Segments:** For segments where the SDF is increasing, the formula naturally yields a negative value inside the max, which is clamped to 0, correctly modeling no opacity contribution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid_cdf(x, s):
    """
    Computes the CDF of the logistic distribution.
    Phi_s(x) = 1 / (1 + exp(-s * x))
    """
    return 1.0 / (1.0 + math.exp(-s * x))

def compute_opacity_sdf_logistic(sdf_start, sdf_end, s):
    """
    Computes the discrete opacity alpha_i for a ray segment
    based on SDF values at the segment boundaries.
    
    Args:
        sdf_start: f(p(t_i)), SDF value at the start of the segment.
        sdf_end: f(p(t_{i+1})), SDF value at the end of the segment.
        s: Steepness parameter of the logistic distribution.
        
    Returns:
        alpha_i: Opacity of the segment, in [0, 1].
    """
    # Evaluate CDF at boundaries
    cdf_start = sigmoid_cdf(sdf_start, s)
    cdf_end = sigmoid_cdf(sdf_end, s)
    
    # Avoid division by zero
    if cdf_start == 0.0:
        return 0.0
        
    # Compute opacity using the derived formula
    # alpha_i = max((CDF_start - CDF_end) / CDF_start, 0)
    alpha = (cdf_start - cdf_end) / cdf_start
    
    # Clamp to [0, 1] to handle numerical noise and exiting segments
    if alpha < 0.0:
        alpha = 0.0
    elif alpha > 1.0:
        alpha = 1.0
        
    return alpha
```

## Subtopics

### [Coverage-based Opacity Mapping](4519-coverage-based-opacity-mapping.md)
Coverage-based Opacity Mapping is a method to convert accumulated unnormalized Gaussian weights into per-layer coverage opacities for surface compositing. It solves the problem of jagged, aliased edges and incorrect blending in surface splatting by replacing threshold-based clamping with a smooth, monotonically increasing mapping. Unlike the single-layer ternary depth test which produces hard transitions, this method yields smooth opacity falloff against the background and gradual transitions in overlap regions. It builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md) by handling the final compositing of surface layers rather than deriving initial opacity from signed distances.

### [SDF-based Volume Rendering](9964-sdf-based-volume-rendering.md)
SDF-based Volume Rendering is a neural rendering technique that reconstructs surfaces by optimizing a Signed Distance Field (SDF) and rendering it via volume integration. It solves the problem of differentiating through the zero-level set of an SDF during volume rendering by mapping SDF values to opacity using a logistic function. It differs from naive SDF-to-density mappings by ensuring the weight function peaks at the surface intersection, avoiding bias, and builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md).

### NeUDF Discretization (Alpha Compositing)
NeUDF Discretization is a method to discretize the continuous weight function for volume rendering by dividing the sample ray into bins and computing opacity ($\alpha_i$) for each bin based on the UDF values at the bin boundaries [S2]. It uses a modified alpha calculation to ensure stability: $\alpha_i = \frac{\varsigma_i^{\max} - \varsigma_i^{\min}}{\varsigma_i^{\max}}$, where $\varsigma_i^{\max}$ and $\varsigma_i^{\min}$ are the maximum and minimum of the set $\{\varsigma_r \circ \Psi \circ p(t_i), \varsigma_r \circ \Psi \circ p(t_{i+1})\}$ [S2]. This variant adapts the logistic CDF approach to Unsigned Distance Fields (UDF) and uses a specific normalization based on the maximum value in the bin to handle the discretization [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, as it involves evaluating the SDF and a few arithmetic operations including an exponential for the CDF [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per segment [S1, unofficial].
*   **Strengths:** Provides unbiased surface reconstruction by ensuring the weight function peaks at the surface intersection [S1, unofficial]. It is differentiable, allowing for gradient-based optimization of the SDF network [S1, unofficial]. It handles the transition between inside and outside the surface smoothly [S1, unofficial].
*   **Limitations:** Requires the SDF to be differentiable and the gradient to be available for the density derivation (though the final opacity formula only requires SDF values, the derivation assumes gradient properties) [S1, unofficial]. Performance depends on the steepness parameter $s$; very high $s$ may lead to numerical instability if not handled carefully [G].
*   **When to use:** When reconstructing surfaces from multi-view images using neural implicit representations (NeRF-like) where surface sharpness and unbiased reconstruction are critical [S1, unofficial].
*   **When not to use:** For traditional volume rendering of non-surface data (e.g., clouds, smoke) where the density is not derived from a distance field [S1, unofficial].

## Background
The method originates from the NeuS paper, which introduced learning neural implicit surfaces by volume rendering for multi-view reconstruction [S1, unofficial]. It was developed to address the bias in previous SDF-based rendering methods by using the logistic distribution's CDF to map SDF values to opacity [S1, unofficial]. The derivation ensures that the opacity is non-zero only when entering the surface, providing a physically consistent model for surface rendering [S1, unofficial]. It has been adopted and adapted in subsequent works such as NeUDF, which applies similar discretization principles to Unsigned Distance Fields [S2].

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
