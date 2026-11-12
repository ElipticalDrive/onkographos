# Opacity Split for Cloning

Also called: Cloning Strategy for Gaussian Splatting, Revising Densification Opacity Initialization

## Overview
Opacity Split for Cloning is a method used during the densification phase of 3D Gaussian Splatting to clone a parent Gaussian into $N$ copies while minimizing the difference between the rasterization outcomes before and after cloning [S1]. It solves the problem where standard cloning causes the parent and child to collectively exert greater transmittance impact than the parent alone, biasing the alpha blending process [S2]. It sets itself apart from standard 3DGS cloning by explicitly deriving opacity and covariance adjustments to preserve rendering intensity, rather than simply inheriting parent parameters [S1, S2].

## How it works
**Inputs:**
*   Parent Gaussian $g_{old}$ with opacity $o_{old}$, covariance $\Sigma_{old}$, and central color $C_{old}(\mu)$ [S1].
*   Number of clones $N$ [S1].

**Outputs:**
*   $N$ cloned Gaussians $g_{new}^{(i)}$ for $i=1 \dots N$, each with opacity $o_{new}$, covariance $\Sigma_{new}$, and central color $C_{new}(\mu)$ [S1].

**Data Structures:**
*   Gaussian parameters are stored as standard 3DGS attributes: mean $\mu$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$ (logistic parameter), and spherical harmonics coefficients for color [S1].

**Steps:**
1.  **Initialize Geometry and Color:** Set the covariance of the cloned Gaussians equal to the original: $\Sigma_{new} = \Sigma_{old}$ [S1]. Set the central color values of the cloned Gaussians identical to the original: $C_{new}(\mu) = C_{old}(\mu)$ [S1].
2.  **Calculate Opacity:** Compute the new opacity $o_{new}$ for both the parent and the clones to maintain transmittance consistency. The relationship required to preserve rasterization intensity at the center $\mu$ is derived from the transmittance equation $(1 - o^{new})^N = 1 - o^{old}$ [S1]. Solving for $o_{new}$ when $N=2$ (standard cloning) yields:
    $$ o_{new} = 1 - \sqrt{1 - o_{old}} $$
    where $o_{old}$ is the parent's opacity before cloning, and $o_{new}$ is the opacity of the parent and each clone child after cloning [S2]. For general $N$, the opacity is:
    $$ o_{new} = 1 - (1 - o_{old})^{1/N} $$
    [S1, S2].
3.  **Adjust Covariance (Optional/Advanced):** The opacity adjustment alone is insufficient to preserve rasterization because Gaussians are not point sources [S1]. To minimize the error in the integrated 1D slices passing through the Gaussian centers, one must minimize:
    $$ \text{minimize } \left\| \int_{-\infty}^{\infty} \mathbf{C}^{new}(x) dx - \int_{-\infty}^{\infty} \mathbf{C}^{old}(x) dx \right\|_2^2, \quad \forall \mathcal{S} $$
    where $\mathcal{S}$ represents arbitrary slices and $x = S(x)$ denotes points along the slice [S1]. This has a simple analytical solution for adjusting the scales, ensuring $\int_{-\infty}^{\infty} \mathbf{C}^{new}(x) dx = \int_{-\infty}^{\infty} \mathbf{C}^{old}(x) dx$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clone_gaussian_with_opacity_split(gaussian, num_clones):
    """
    Clones a Gaussian, adjusting opacity and potentially covariance to preserve rasterization.
    
    Args:
        gaussian: Object containing .opacity (o_old), .covariance (Sigma_old), .color (C_old)
        num_clones: Integer N, number of clones to create.
        
    Returns:
        list: List of N new Gaussian objects.
    """
    o_old = gaussian.opacity
    # Calculate new opacity to preserve transmittance at center
    # Formula: o_new = 1 - (1 - o_old)^(1/N)
    o_new = 1.0 - (1.0 - o_old) ** (1.0 / num_clones)
    
    new_gaussians = []
    for i in range(num_clones):
        new_g = Gaussian()
        # Copy geometry and color as per [S1]
        new_g.covariance = gaussian.covariance # Sigma_new = Sigma_old
        new_g.color = gaussian.color           # C_new(mu) = C_old(mu)
        new_g.opacity = o_new
        
        # Note: Covariance adjustment for integral preservation is complex.
        # Standard implementation often keeps Sigma_new = Sigma_old [S1].
        # If full integral preservation is required, solve the 1D slice minimization [S1].
        
        new_gaussians.append(new_g)
        
    return new_gaussians
```

**Parameters:**
*   $N$: Number of clones. Typically 2 in standard 3DGS densification [S1, S2].
*   $o_{old}$: Parent opacity. Range $(0, 1)$.

**Edge Cases and Pitfalls:**
*   **Point Source Assumption:** Equation $(1 - o^{new})^N = 1 - o^{old}$ is derived assuming the Gaussian is a point source at $\mu$. Since Gaussians have extent, this equation alone is insufficient to preserve rasterization; covariance adjustment is theoretically required [S1].
*   **Transmittance Bias:** In standard 3DGS, cloned children inherit the parent's opacity. This causes the parent and child together to have greater transmittance impact than the parent alone, biasing subsequent alpha blending [S2]. Opacity splitting corrects this by reducing the opacity of both parent and child [S2].
*   **Precision:** For $o_{old}$ close to 1, $(1 - o_{old})$ may suffer from floating-point underflow. Use stable logarithmic formulations if necessary [G].

## Subtopics

### GES Optimization Strategy
Uses gradient-based densification criteria related to the number of pixels influenced, reducing densification frequency for large background Gaussians [S2].

### Temporal Sensitivity Pruning
Focuses on pruning based on temporal stability in dynamic scenes, distinct from static opacity management [Parent].

### GRay Optimization Settings
Configures optimization hyperparameters for Gaussian Splatting, including densification intervals, but does not specify the cloning opacity formula [Parent].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per clone for opacity calculation. Covariance adjustment (if implemented) is $O(1)$ if analytical solution is used [S1].
*   **Memory Complexity:** Increases memory footprint by a factor of $N$ for the cloned Gaussians, same as standard cloning [S1].
*   **Strengths:** Prevents bias in alpha blending caused by cloned Gaussians having excessive transmittance impact [S2]. Maintains visual consistency during densification [S1].
*   **Limitations:** The opacity adjustment alone does not fully preserve rasterization due to the non-point nature of Gaussians; covariance adjustment is needed for full fidelity [S1].
*   **When to use:** When using densification strategies that involve cloning Gaussians (e.g., standard 3DGS) and visual artifacts or bias in alpha blending are observed [S2].

## Background
The method originates from the observation that standard 3D Gaussian Splatting cloning inherits parent opacity, leading to transmittance bias [S2]. Bulo et al. (2024) proposed adapting the opacity initialization for newly cloned Gaussians to equally split the opacity between parent and child, keeping transmittance approximately the same [S2]. The formulation $(1 - o^{new})^N = 1 - o^{old}$ is also discussed in the context of 3D Gaussian Splatting as Markov Chain Monte Carlo, where it is noted that covariance adjustment is also required for full rasterization preservation [S1].

## Sources
[S1] 3D Gaussian Splatting as Markov Chain Monte Carlo, https://ubc-vision.github.io/3dgs-mcmc/, fetched 2026-10-26
[S2] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
