# Gradient-based Densification Control

Also called: Densification and Gradient Threshold, Gradient-based Densification

## Overview
Gradient-based Densification Control is a mechanism for allocating 3D Gaussian primitives to regions with high reconstruction error by using gradient magnitude as a proxy for representational need [S1, unofficial]. It differs from sibling methods by explicitly decoupling the decision to densify (based on gradient magnitude) from the strategy of densification (splitting vs. cloning, based on spatial extent), allowing for dataset-specific tuning of densification aggressiveness [S1, unofficial].

## How it works
**Inputs:**
*   Per-primitive gradient magnitude $g_i$ for each active primitive $i$ [S1, unofficial].
*   Per-primitive spatial extent $s_i$ [S1, unofficial].
*   Gradient threshold parameter $thres\_grad$ [S1, unofficial].
*   Percent-dense parameter $per\_dense$ [S1, unofficial].

**Outputs:**
*   Updated primitive set with new positions, scales, and opacities [S1, unofficial].

**Data Structures:**
*   Primitive list: Each primitive $i$ stores position $\mathbf{p}_i$, covariance matrix $\Sigma_i$ (or scale/rotation/quaternion), opacity $\alpha_i$, and accumulated gradient magnitude $g_i$ [G].
*   Parameters: $thres\_grad \in [1 \times 10^{-10}, 6.5 \times 10^{-9}]$, $per\_dense \in [0.001, 0.01]$ [S1, unofficial].

**Steps:**
1. Compute gradient magnitude $g_i$ for each primitive $i$ during the backward pass [S1, unofficial].
2. Compare $g_i$ against $thres\_grad$ to decide if densification is triggered [S1, unofficial].
3. If triggered, evaluate spatial extent $s_i$ against $per\_dense$ to choose between splitting or cloning [S1, unofficial].
4. Jointly adjust $thres\_grad$ and $per\_dense$ to control densification aggressiveness per dataset and volume resolution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Primitive:
    def __init__(self, pos, cov, opacity, grad_mag):
        self.pos = pos
        self.cov = cov
        self.opacity = opacity
        self.grad_mag = grad_mag

def densify_primitives(primitives, thres_grad, per_dense):
    """
    Densify primitives based on gradient magnitude and spatial extent.
    
    Args:
        primitives: List of Primitive objects
        thres_grad: float, gradient threshold (1e-10 to 6.5e-9)
        per_dense: float, percent-dense threshold (0.001 to 0.01)
        
    Returns:
        new_primitives: List of new Primitive objects (clones or splits)
    """
    new_primitives = []
    
    for prim in primitives:
        g_i = prim.grad_mag
        s_i = prim.cov.trace() / 3  # Representative spatial extent (mean scale)
        
        # Step 2: Threshold Check
        if g_i < thres_grad:
            continue  # No densification triggered
            
        # Step 3 & 4: Extent Evaluation and Split/Clone Decision
        if s_i > per_dense:
            # Split: Create multiple smaller primitives around the original position
            splits = []
            for _ in range(2):
                new_p = Primitive(
                    pos=prim.pos[:],
                    cov=prim.cov * 0.5,
                    opacity=prim.opacity / 2,
                    grad_mag=0.0
                )
                splits.append(new_p)
            new_primitives.extend(splits)
        else:
            # Clone: Create a copy to increase density without changing individual size
            clone = Primitive(
                pos=prim.pos[:],
                cov=prim.cov.copy(),
                opacity=prim.opacity,
                grad_mag=0.0
            )
            new_primitives.append(clone)
            
    return new_primitives
```

**Edge Cases and Pitfalls:**
*   The sources do not specify additional edge cases beyond parameter tuning per dataset and volume resolution [S1, unofficial].

## Subtopics

### [Screen-Space Gradient Densification Criterion](5303-screen-space-gradient-densification-criterion.md)
This method provides a criterion for 3D primitive densification based on the mean magnitude of the gradient of the photometric loss with respect to the projected 2D screen-space center of the primitive, accumulated over all training views. It differs from alternatives like 2DGS [Huang et al. 2024] by using an analytical T-matrix backward pass to avoid conflating depth and lateral motion, which prevents gradient cancellation and ensures the signal faithfully captures the rendering residual. [Screen-Space Gradient Densification Criterion](5303-screen-space-gradient-densification-criterion.md)

### Density-Aware Splitting
A splitting/cloning strategy that compares surfel scale against local sampling density (mean neighbor distance) rather than a fixed global scene extent fraction [S2, unofficial]. It maintains a spatial index structure over surfel centers and queries the mean distance $\bar{d}$ to same-orientation nearest neighbors for each surfel to estimate local sampling density [S2, unofficial]. A global scale threshold is unsuitable because it does not adapt to local density or enforce the spatially varying overlap condition required by the partition-of-unity assumption [S2, unofficial].

## Operating characteristics
The method ensures efficient spatial allocation of primitives to regions with high reconstruction error [S1, unofficial]. Parameters are tuned depending on the dataset and volume resolution, with $thres\_grad$ ranging from $1 \times 10^{-10}$ to $6.5 \times 10^{-9}$ and $per\_dense$ from $0.001$ to $0.01$ [S1, unofficial]. These parameters jointly control the aggressiveness of densification [S1, unofficial].

## Background
Gradient-based densification follows the original 3D Gaussian Splatting (3DGS) principle, where primitives with large gradients indicate regions requiring more representational capacity and are selected for cloning or splitting [S1, unofficial]. It is used in the ESVR framework for large-scale volumetric datasets such as the hemibrain and BigBrain datasets [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
