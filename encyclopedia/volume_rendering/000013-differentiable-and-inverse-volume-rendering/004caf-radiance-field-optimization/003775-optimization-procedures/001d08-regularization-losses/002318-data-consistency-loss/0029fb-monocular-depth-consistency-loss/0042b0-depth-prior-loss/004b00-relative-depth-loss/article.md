# Relative Depth Loss

Also called: Lrel, Single-view relative depth loss

## Overview
Relative Depth Loss is a regularization term that constrains the geometry of a radiance field by aligning rendered depth with monocular or Multi-View Stereo (MVS) depth priors [S1]. It differs from absolute depth consistency losses by using a relative formulation ($D_{mvs}/D_r$), which exhibits high sensitivity to errors at closer depths [S1]. It is distinguished from its parent [Depth Prior Loss](10801-depth-prior-loss.md) by its specific mechanism of rejecting erroneous MVS values via thresholding and weighting by rendering certainty [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** MVS depth map $D_{mvs}$, rendered depth map $D_r$, rendering certainty $U$, thresholding hyperparameter $s$ [S1].
*   **Output:** Scalar loss value $L_{rel}$ [S1].

**Data Structures**
*   $D_{mvs}$: 2D array of depth values from an external MVS estimator or monocular depth network [S1].
*   $D_r$: 2D array of rendered depth values, computed per-pixel [S1].
*   $U$: 2D array of rendering certainty, computed per-pixel [S1].
*   $\omega_i$: Blending weight of the $i$-th Gaussian [S1].
*   $t_i$: Depth of the $i$-th Gaussian [S1].
*   $T_i$: Accumulated transmittance up to the $i$-th Gaussian [S1].

**Formulas**
The rendered depth $D_r$ is computed as the median depth, defined as the largest depth value $t_i$ considered visible:
$$ D_r = \max\{t_i \mid T_i > 0.5\} $$
where $T_i$ is the accumulated transmittance [S1].

Rendering certainty $U$ is defined as the sum of blending weights:
$$ U = \sum_i \omega_i $$
[S1].

The loss is calculated as:
$$ L_{rel} = 1 - \frac{D_{mvs}}{D_r} \cdot U \cdot I(|D_r - D_{mvs}| < s \cdot D_r) $$
where:
*   $D_{mvs}$ is the MVS depth [S1].
*   $D_r$ is the rendered depth [S1].
*   $s$ is the thresholding tolerance hyperparameter [S1].
*   $I(\cdot)$ is the indicator function, yielding 1 if the condition holds and 0 otherwise [S1].

**Parameters**
*   $s$: Thresholding tolerance. Typically annealed from a less restrictive to a more restrictive value during optimization [S1].

**Pitfalls and Edge Cases**
*   **Mean Depth Artifacts:** Using mean depth ($\sum \omega_i t_i / \sum \omega_i$) leads to inaccurate estimates for semi-transparent or transparent objects and introduces "flying pixel" artifacts near boundaries of opaque objects [S1].
*   **Weight Instability:** Reliance on mean depth calculated using potentially unstable weights $\omega_i$ during optimization can harm reconstruction quality [S1].
*   **Sensitivity:** The relative formulation is highly sensitive to errors at closer depths [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_relative_depth_loss(D_mvs, D_r, U, s):
    """
    Computes the Relative Depth Loss (Lrel) per pixel.
    
    Args:
        D_mvs: 2D array, MVS depth map.
        D_r: 2D array, rendered median depth map.
        U: 2D array, rendering certainty (sum of blending weights).
        s: float, thresholding tolerance hyperparameter.
        
    Returns:
        2D array, per-pixel loss values.
    """
    epsilon = 1e-6  # [G] Guard against division by zero if D_r is 0
    ratio = D_mvs / (D_r + epsilon)
    indicator = (abs(D_r - D_mvs) < s * D_r).astype(float)
    return 1.0 - ratio * U * indicator

def compute_rendered_depth(Gaussians):
    """
    Computes rendered depth D_r as the median depth.
    
    Args:
        Gaussians: List of Gaussian objects contributing to a pixel, 
                   sorted by depth. Each Gaussian has attributes:
                   - t: depth
                   - T: accumulated transmittance (pre-computed or computed during rasterization)
                   
    Returns:
        float, the median depth D_r.
    """
    visible_gaussians = [g for g in Gaussians if g.T > 0.5]
    if not visible_gaussians:
        return 0.0  # [G] Fallback for empty pixel
    return max(g.t for g in visible_gaussians)

def compute_rendering_certainty(Gaussians):
    """
    Computes rendering certainty U as the sum of blending weights.
    
    Args:
        Gaussians: List of Gaussian objects contributing to a pixel.
                   Each Gaussian has attribute:
                   - omega: blending weight
                   
    Returns:
        float, the rendering certainty U.
    """
    return sum(g.omega for g in Gaussians)
```

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Strengths:** Mitigates artifacts associated with mean depth (flying pixels, transparency issues) by using median depth [S1]. High sensitivity to close-depth errors improves geometric fidelity [S1].
*   **Limitations:** Single-view application may not regularize all scene regions effectively, as gradients primarily affect Gaussians contributing to the median depth in that specific view [S1].
*   **When to use:** When high geometric fidelity is required, especially for close-range objects, and MVS priors are available [S1]. Suitable for scenes where mean depth fails due to transparency or boundary artifacts [S1].
*   **When not to use:** Not given in the sources.

## Background
Relative Depth Loss was introduced in the context of Multiview Geometric Regularization of Gaussian Splatting for Accurate Radiance Fields [S1]. It addresses limitations of mean depth computation in 3D Gaussian Splatting, such as "flying pixel" artifacts and inaccuracies with transparent objects [S1]. The method builds on the observation that median depth provides a more robust estimate of surface location [S1].

## Sources
[S1] Multiview Geometric Regularization of Gaussian Splatting for Accurate Radiance Fields, https://arxiv.org/abs/2506.13508v1, fetched 2026-10-26
