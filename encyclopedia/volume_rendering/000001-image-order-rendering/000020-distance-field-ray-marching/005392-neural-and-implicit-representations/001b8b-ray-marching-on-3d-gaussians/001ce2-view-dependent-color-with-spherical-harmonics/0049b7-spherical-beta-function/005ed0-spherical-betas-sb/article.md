# Spherical Betas (SB)

Also called: SB color representation

## Overview
Spherical Betas (SB) is a directional appearance representation that extends Spherical Gaussians (SG) by enabling asymmetric and bounded-support lobes to model skewed and sharper directional variation [S1, unofficial]. It is used in Deformable Beta Splatting to model view-dependent specular color [S2]. Unlike SG, which has unbounded support requiring truncation, SB provides a compact, bounded alternative that ensures radiance continuity [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** View direction $\hat{V}$, primitive parameters (base color, lobe directions, shape parameters) [S2].
*   **Output:** View-dependent color $c(\hat{V})$ [S2].

**Data Structures:**
A 3D Beta primitive stores features that scale linearly with the number of reflection lobes $M$ [S2]. The feature dimension is $3 + 6M$ [S2]:
*   $c_0$: Diffuse/base color (3 components) [S2].
*   For each lobe $m \in \{1, \dots, M\}$:
    *   $\hat{R}_m$: Reflection direction (3 components, normalized) [S2].
    *   $c_m$: Lobe color (3 components) [S2].
    *   $b_m$: Shininess parameter (1 component, scalar) [S2].
    *   *Note:* The source specifies a dimension of $3 + 6M$ despite listing seven components per lobe; the implementation follows the stated dimension [S2].

**Formulas:**
The general Spherical Beta function for a direction $\omega$ with $K$ lobes is defined as [S1, unofficial]:
$$ f_{SB} (\omega; \alpha, \beta, s) = \sum_{k=1}^{K} (1+s_k \cdot \omega)^{\alpha_k -1} (1-s_k \cdot \omega)^{\beta_k -1} $$
where $s_k$ is the principal direction and $\alpha_k, \beta_k > 0$ control the shape [S1, unofficial].

In the specular lobe variant, appearance is modeled as a sum of a base diffuse color and specular lobes [S2]:
$$ c(\hat{V}) = c_0 + \sum_{m \in \mathcal{M}} \mathcal{B}(1 - \hat{R}_m \cdot \hat{V}; b_m) c_m $$
where $c_0$ is the diffuse/base color, $\mathcal{M}$ is the set of SB lobes, $\hat{R}_m$ is the reflection direction derived from surface normal and light source, $\hat{V}$ is the view direction, $c_m$ is the lobe color, $b_m$ is the shininess coefficient, and $\mathcal{B}$ is the Beta Kernel function [S2].

**Parameters:**
*   $M$: Number of reflection lobes per primitive. Default value is $M=2$ ($sb=2$) [S2].
*   $b_m$: Shininess parameter. Initialized to zero to make the Beta Kernel initially resemble the Gaussian Kernel [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_beta_color(base_color, lobes, view_dir):
    """
    Computes view-dependent color using Spherical Beta lobes.
    
    Args:
        base_color (vec3): Diffuse/base color c_0.
        lobes (list of dict): List of M lobes. Each lobe contains:
            - 'R': reflection direction (vec3, normalized)
            - 'c': lobe color (vec3)
            - 'b': shininess parameter (float)
        view_dir (vec3): View direction (vec3, normalized).
        
    Returns:
        vec3: Final view-dependent color.
    """
    total_specular = vec3(0.0, 0.0, 0.0)
    
    for lobe in lobes:
        R = lobe['R']
        c_m = lobe['c']
        b = lobe['b']
        
        # Calculate argument for Beta kernel: 1 - R . V
        # Source states input range is confined to [0, 1]
        cos_theta = dot(R, view_dir)
        x = 1.0 - cos_theta
        x = max(0.0, min(1.0, x))  # Clamp to [0, 1] per source claim
        
        # [Comment: The explicit mathematical formula for B(x; b) is not provided in the sources. 
        # A standard Beta distribution kernel is implemented below for completeness.]
        beta_val = beta_kernel(x, b)
        
        total_specular += beta_val * c_m
        
    return base_color + total_specular

def beta_kernel(x, b):
    """Standard Beta distribution kernel approximation."""
    # [G] Standard Beta PDF form: x^(alpha-1) * (1-x)^(beta-1)
    # Mapping shininess b to shape parameters for implementation.
    alpha = b + 1.0
    beta_shape = b + 1.0
    if x <= 0.0 or x >= 1.0:
        return 0.0
    return (x ** (alpha - 1.0)) * ((1.0 - x) ** (beta_shape - 1.0))
```

**Edge Cases and Pitfalls:**
*   **Ill-conditioned Gradients:** Extreme values of $\alpha_k$ or $\beta_k$ (or $b_m$) generate very sharp and flat regions, causing ill-conditioned gradients and stronger sensitivity to initialization compared to Spherical Gaussians [S1, unofficial].
*   **Truncation Artifacts:** Unlike Spherical Gaussians, which have unbounded support and require truncation (causing radiance discontinuities when view and reflection directions are orthogonal), Spherical Beta is inherently bounded [S2]. Its input range is confined to $[0, 1]$, ensuring radiance naturally drops to zero when $\hat{R}_m \cdot \hat{V} = 0$ [S2].

## Subtopics

### Spherical Beta
Extends SG with asymmetric, bounded-support lobes for sharper directional variation [S1, unofficial]. Harder to optimize due to gradient ill-conditioning at extreme shape parameters [S1, unofficial]. See [Spherical Voronoi](12574-spherical-beta-function.md) for comparison with other directional representations [S1, unofficial].

### Spherical Beta
Models outgoing reflected radiance by merging ambient and diffuse into a base color $c_0$ [S2]. Uses learnable bounded Beta kernels for specular lobes [S2]. Chosen for efficiency ($3+6M$ dimension) and continuity (no truncation artifacts) compared to SG [S2]. Default uses $M=2$ lobes per primitive [S2].

## Operating characteristics
*   **Time Complexity:** Evaluation is $O(M)$ per primitive [S2].
*   **Memory Complexity:** Feature dimension per primitive is $3 + 6M$ [S2]. For default $M=2$, this is 15 floats per primitive [S2].
*   **Performance:** Fully explicit and differentiable, implemented in CUDA for optimized performance [S2]. Convergence speed varies by scene, with default training at 30,000 iterations [S2].
*   **Strengths:** Bounded support eliminates truncation artifacts and radiance discontinuities at orthogonal view/reflection angles [S2]. More flexible than SG for modeling skewed/sharp lobes [S1, unofficial].
*   **Limitations:** Harder to optimize than SG due to ill-conditioned gradients and sensitivity to initialization [S1, unofficial].

## Background
Spherical Betas were introduced as an extension of Spherical Gaussians to provide asymmetric and bounded-support lobes [S1, unofficial]. They are used in Deformable Beta Splatting [S2]. The Deformable Beta Splatting implementation builds upon the 3DGS-MCMC framework and GSplat library [S2].

## Sources
[S1] Spherical Voronoi: Directional Appearance as a Differentiable Partition of the Sphere, https://doi.org/10.48550/arXiv.2512.14180, fetched 2026-10-30 (unofficial)
[S2] Deformable Beta Splatting, https://doi.org/10.1145/3721238.3730716, fetched 2026-11-08
