# Opacity Split for Cloning

Also called: Cloning Strategy for Gaussian Splatting, Revising Densification Opacity Initialization

## Overview
Opacity Split for Cloning is a method used during the densification phase of 3D Gaussian Splatting to clone existing Gaussians into multiple copies while preserving the rendered image intensity [S1]. It solves the problem where naive cloning (copying parameters directly) alters the alpha-blending transmittance, causing visible artifacts or bias in the optimization [S2]. It sets apart from standard cloning by explicitly adjusting the opacity of both the parent and the clones to maintain transmittance consistency, ensuring the rasterization outcome remains unchanged immediately after the split [S1, S2].

## How it works
The method involves cloning a single Gaussian primitive into $N$ copies. The core challenge is that Gaussians are not point sources; simply copying the covariance and color is insufficient to preserve the rasterization integral [S1]. The method adjusts opacity and covariance to minimize the difference between the pre- and post-cloning rasterization outcomes [S1].

### Inputs and Outputs
*   **Input:** A parent Gaussian $g_{old}$ with opacity $o_{old}$, covariance $\Sigma_{old}$, and mean color $\mathbf{C}_{old}(\boldsymbol{\mu})$.
*   **Input:** The number of clones $N$ (typically $N=2$ in standard implementations, but the formula generalizes) [S1].
*   **Output:** $N$ new child Gaussians and the modified parent Gaussian (if the parent is retained) with new opacity $o_{new}$, covariance $\Sigma_{new}$, and color $\mathbf{C}_{new}$.

### Data Structures
The Gaussian primitive retains the standard 3DGS layout:
*   $\boldsymbol{\mu} \in \mathbb{R}^3$: Mean position.
*   $\Sigma \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often stored as rotation quaternion $\mathbf{q}$ and scale vector $\mathbf{s}$).
*   $\alpha \in [0, 1]$: Opacity.
*   $\mathbf{C} \in \mathbb{R}^3$: Mean color (SH coefficients omitted for brevity, assumed constant).

### Steps

1.  **Initialize Parameters:**
    Set the mean position, covariance, and color of the cloned Gaussians identical to the parent:
    $$ \boldsymbol{\mu}_{new} = \boldsymbol{\mu}_{old} $$
    $$ \Sigma_{new} = \Sigma_{old} $$
    $$ \mathbf{C}_{new}(\boldsymbol{\mu}) = \mathbf{C}_{old}(\boldsymbol{\mu}) $$
    [S1]

2.  **Calculate Target Opacity:**
    Determine the new opacity $o_{new}$ for the clones. Two common formulations exist in the literature:
    
    *   **General N-cloning (MCMC approach):** To preserve the rasterization intensity at the center $\boldsymbol{\mu}$, the relationship is derived from the transmittance equation:
        $$ (1 - o^{new})^N = 1 - o^{old} $$
        Solving for $o^{new}$:
        $$ o^{new} = 1 - (1 - o^{old})^{1/N} $$
        [S1]
        
    *   **Binary Split (Bulo et al. approach):** If $N=2$ and the parent is retained with the same opacity as the child:
        $$ o_{new} = 1 - \sqrt{1 - o_{old}} $$
        [S2]

3.  **Adjust Covariance (Covariance Splitting):**
    Equation (11) alone is insufficient because Gaussians have spatial extent [S1]. To minimize the error in the rasterization integral, the method minimizes the mean squared error of integrated 1D slices passing through the Gaussian centers [S1].
    
    The optimization objective is:
    $$ \text{minimize } \left\| \int_{-\infty}^{\infty} \mathbf{C}^{new}(x) dx - \int_{-\infty}^{\infty} \mathbf{C}^{old}(x) dx \right\|_2^2, \quad \forall \mathcal{S} $$
    where $\mathcal{S}$ represents arbitrary slices and $x$ denotes points along the slice [S1].
    
    Since the full 3D integral (Eq. 12 in [S1]) has no simple analytical form, the method uses the 1D slice approximation. For a Gaussian with mean $\boldsymbol{\mu}=0$, the integral of the color along a slice is proportional to the opacity and the inverse of the scale in the direction of the slice. To preserve the integral $\int \mathbf{C} dx$, if the opacity is reduced (or split), the spatial extent (covariance/scale) must be adjusted.
    
    Specifically, for $N$ clones with opacity $o^{new}$, the scale factors $\mathbf{s}$ of the cloned Gaussians are typically scaled by a factor related to $\sqrt{N}$ or derived from the ratio of opacities to maintain the same integrated density. A common implementation (derived from the slice constraint) sets the new scale $\mathbf{s}_{new}$ such that:
    $$ \mathbf{s}_{new} = \mathbf{s}_{old} \cdot \sqrt{\frac{o_{old}}{N \cdot o^{new}}} $$
    [G] *Note: The exact scaling factor depends on the specific derivation in the source. The MCMC paper [S1] derives an update equation (Eq. 9 in source) for the scales based on this integral constraint. If the source does not provide the explicit closed-form scale update, it is computed by solving the 1D slice integral equality.*

4.  **Assign Opacity:**
    Set the opacity of all $N$ clones (and the parent, if retained) to $o_{new}$ [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def opacity_split_for_cloning(gaussian, num_clones):
    """
    Clones a Gaussian into num_clones copies, adjusting opacity and covariance
    to preserve rasterization intensity.
    
    Args:
        gaussian: Object with attributes .opacity, .covariance (or .scale, .rotation), .mean, .color.
        num_clones: Integer N, the number of copies to create.
        
    Returns:
        list: A list of N new Gaussian objects.
    """
    o_old = gaussian.opacity
    N = num_clones
    
    # Step 2: Calculate new opacity
    # Formula: (1 - o_new)^N = 1 - o_old
    # o_new = 1 - (1 - o_old)^(1/N)
    o_new = 1.0 - (1.0 - o_old) ** (1.0 / N)
    
    # Step 1: Initialize clones with same mean, color, and initial covariance
    clones = []
    for i in range(N):
        new_g = Gaussian()
        new_g.mean = gaussian.mean.copy()
        new_g.color = gaussian.color.copy()
        # Initial covariance copy; will be adjusted in Step 3
        new_g.covariance = gaussian.covariance.copy() 
        clones.append(new_g)
        
    # Step 3: Adjust Covariance/Scales
    # To preserve the integral of the 1D slice, the spatial extent must change
    # if the opacity changes.
    # The integral of a Gaussian slice is proportional to alpha / scale.
    # We require: sum(integral_new) = integral_old
    # N * (o_new / scale_new) = o_old / scale_old  (assuming isotropic for simplicity)
    # scale_new = scale_old * (o_new / (o_old / N)) = scale_old * (N * o_new / o_old)
    
    # However, the MCMC paper [S1] suggests minimizing the error. 
    # A robust implementation derived from the slice constraint:
    # If we assume the Gaussian is unnormalized and we want to preserve the 
    # integrated color, the scale must be adjusted.
    
    # Extract scale vector s from covariance Sigma
    # Sigma = R * diag(s^2) * R^T
    s_old = gaussian.get_scale_vector() # Helper to extract scale from covariance
    
    # Calculate scaling factor for the new Gaussians
    # Based on preserving the 1D integral: Integral ~ alpha * sigma_1D
    # We want Integral_new_total = Integral_old
    # N * (o_new * sigma_new) = o_old * sigma_old
    # sigma_new = (o_old / (N * o_new)) * sigma_old
    
    scale_factor = (o_old / (N * o_new))
    s_new = s_old * (scale_factor ** 0.5) # Scale is linear, variance is squared
    
    # Update covariance for all clones
    for clone in clones:
        clone.opacity = o_new
        # Update scale
        clone.set_scale_vector(s_new)
        # Recompute covariance from new scale and original rotation
        clone.covariance = clone.compute_covariance()
        
    return clones
```

### Edge Cases and Pitfalls
*   **Point Source Assumption:** Equation (11) alone is insufficient because Gaussians are not point sources [S1]. Failing to adjust the covariance leads to rasterization errors.
*   **Transmittance Bias:** In standard 3DGS, cloned children inherit the parent's opacity. This increases the total transmittance impact of the parent-child pair compared to the parent alone, biasing the alpha blending process [S2]. Opacity splitting counteracts this [S2].
*   **Precision:** When $o_{old}$ is close to 1, $(1 - o_{old})$ approaches 0, potentially causing numerical instability in the root calculation. Guard against negative values due to floating-point errors.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per clone, as it involves basic arithmetic operations and covariance updates.
*   **Memory Complexity:** $O(1)$ additional memory per clone for the new Gaussian parameters.
*   **Strengths:** Preserves the rendered image immediately after cloning, preventing optimization instability and visual artifacts [S1, S2].
*   **Limitations:** Requires careful handling of covariance updates to maintain the integral constraint [S1]. The analytical solution for the full 3D integral is complex, so the 1D slice approximation is used, which may not perfectly preserve the 3D rasterization in all orientations [S1].
*   **When to use:** During the densification phase of 3D Gaussian Splatting when a Gaussian is deemed significant enough to be split into multiple copies [S1, S2].

## Background
The method originates from the need to improve the densification strategy in 3D Gaussian Splatting. Standard 3DGS cloning simply copies parameters, which alters the transmittance [S2]. Bulo et al. (2024) proposed revising the opacity initialization to equally split opacity between parent and child to maintain transmittance consistency [S2]. Independently, the MCMC formulation of 3DGS [S1] derived a cloning strategy that minimizes the difference in rasterization outcomes by adjusting both opacity and covariance based on 1D slice integrals. Both approaches aim to solve the same problem: ensuring that cloning does not bias the rendering or the optimization landscape.

## Sources

[S1] 3D Gaussian Splatting as Markov Chain Monte Carlo, https://ubc-vision.github.io/3dgs-mcmc/, fetched 2026-10-26
[S2] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
