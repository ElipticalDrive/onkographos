# Opacity-aware Non-Square Bounding Box

Also called: Level-set Bounding Box, Splat Bounding, Tight Bounding Box

## Overview
This method computes a tight, axis-aligned rectangular bounding box for a 2D Gaussian splat based on its projected opacity and covariance, replacing the original 3D Gaussian Splatting (3DGS) approach which uses a square bounding box derived from a fixed confidence radius [S1]. It reduces false positives in per-tile splat lists by accounting for the Gaussian's actual opacity and the non-square nature of its projected shape, improving culling efficiency compared to siblings like the [Opacity-Sensitive Elliptical Bounding Box](2100-opacity-sensitive-elliptical-bounding-box.md) which may still overestimate intersections [S2, unofficial].

## How it works
**Inputs:**
- $\mu_{2D} \in \mathbb{R}^2$: The projected mean position of the 2D Gaussian.
- $\Sigma_{2D} \in \mathbb{R}^{2 \times 2}$: The projected 2D covariance matrix.
- $o \in [0, 1]$: The opacity of the Gaussian splat.
- $\tau_\alpha \in [0, 1]$: The opacity threshold below which the splat is skipped (typically $\tau_\alpha = 1/255$) [S1][S2, unofficial].

**Outputs:**
- $b_{min} \in \mathbb{R}^2$: The minimum corner of the bounding box.
- $b_{max} \in \mathbb{R}^2$: The maximum corner of the bounding box.

**Data Structures:**
- The bounding box is represented by two 2D vectors, $b_{min}$ and $b_{max}$, defining the axis-aligned rectangle.

**Steps:**
1. Calculate the scaling factor $\gamma$ based on the opacity threshold and the splat's opacity:
   $$ \gamma = -2 \ln\left(\frac{\tau_\alpha}{o}\right) $$
   This factor is derived by setting the Gaussian's alpha value equation equal to $\tau_\alpha$ and solving for the exponent's numerator [S2, unofficial]. Note: If $o < \tau_\alpha$, the splat is effectively invisible and may be skipped entirely [S1].

2. Compute the half-extents of the rectangle along the x and y axes using the diagonal elements of the covariance matrix scaled by $\gamma$:
   $$ h_x = \sqrt{\gamma \cdot \Sigma_{2D,1,1}} $$
   $$ h_y = \sqrt{\gamma \cdot \Sigma_{2D,2,2}} $$
   Here, $\Sigma_{2D,1,1}$ and $\Sigma_{2D,2,2}$ are the diagonal elements of $\Sigma_{2D}$ [S2, unofficial].

3. Determine the bounding box corners using the projected mean $\mu_{2D} = (\mu_x, \mu_y)$:
   $$ b_{min} = (\mu_x - h_x, \mu_y - h_y) $$
   $$ b_{max} = (\mu_x + h_x, \mu_y + h_y) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_opacity_aware_bounding_box(mu_2d, sigma_2d, opacity, tau_alpha):
    """
    Computes a tight, non-square axis-aligned bounding box for a 2D Gaussian splat.
    
    Args:
        mu_2d: Tuple (mu_x, mu_y) representing the projected mean.
        sigma_2d: 2x2 matrix (list of lists) representing the projected covariance.
        opacity: Float representing the splat's opacity o.
        tau_alpha: Float representing the opacity threshold (default 1/255).
        
    Returns:
        Tuple (b_min, b_max) where each is a tuple (x, y).
    """
    # Step 1: Calculate scaling factor gamma
    # If opacity is below threshold, the splat is invisible. 
    # We return a degenerate box or handle culling upstream.
    if opacity < tau_alpha:
        # [G] Return a zero-sized box at the mean, or handle as "skip" in caller.
        return (mu_2d, mu_2d)
        
    gamma = -2.0 * math.log(tau_alpha / opacity)
    
    # Step 2: Compute half-extents
    # sigma_2d is [[s11, s12], [s21, s22]]
    s11 = sigma_2d[0][0]
    s22 = sigma_2d[1][1]
    
    h_x = math.sqrt(gamma * s11)
    h_y = math.sqrt(gamma * s22)
    
    # Step 3: Determine corners
    mu_x, mu_y = mu_2d
    b_min = (mu_x - h_x, mu_y - h_y)
    b_max = (mu_x + h_x, mu_y + h_y)
    
    return (b_min, b_max)
```

**Edge Cases and Pitfalls:**
- **Opacity Zero:** If $o=0$, $\ln(\tau_\alpha/o)$ is undefined. In practice, splats with $o < \tau_\alpha$ are skipped before this computation [S1].
- **Precision:** For very small opacities close to $\tau_\alpha$, $\gamma$ approaches 0, resulting in a tight box. For large opacities, $\gamma$ increases, expanding the box.
- **Covariance Degeneracy:** If $\Sigma_{2D}$ is singular (e.g., a line or point), one half-extent may be zero. The method still produces a valid (possibly degenerate) rectangle [S2, unofficial].

## Subtopics

### [Opacity-Sensitive Elliptical Bounding Box](2100-opacity-sensitive-elliptical-bounding-box.md)
Uses an elliptical bound instead of a rectangular one. More accurate but more expensive to test for tile intersection. Choose when precision is critical and performance allows.

### [Tight Ellipse AABB Derivation](2826-tight-ellipse-aabb-derivation.md)
Computes the AABB of an ellipse. Similar output to this method but derived differently. Choose when using elliptical culling logic.

## Operating characteristics
- **Time Complexity:** $O(1)$ per splat. Requires a few arithmetic operations, a logarithm, and square roots.
- **Memory Complexity:** $O(1)$ additional memory per splat for the bounding box coordinates.
- **Performance:** Reduces false positives in per-tile splat lists compared to the original 3DGS square bounding box [S2, unofficial]. This leads to fewer unnecessary rasterization operations.
- **Strengths:** Tighter fit than square boxes; accounts for opacity; simple to implement.
- **Limitations:** Still an axis-aligned approximation; may overestimate tile intersections for rotated Gaussians compared to elliptical culling [S2, unofficial].

## Background
The original 3DGS method used a square bounding box derived from a 99% confidence ellipse radius (approx. $3\sigma$) [S1]. This approach was criticized for underestimating the size of opaque, axis-aligned splats and overestimating others due to the square constraint [S2, unofficial]. This method was proposed to replace the probabilistic argument with a level-set approach based on the opacity threshold $\tau_\alpha$ [S1]. It is used in optimized 3DGS renderers to improve culling efficiency [S2, unofficial].

## Sources
[S1] Fast 3D Gaussian Splatting Rendering via Easily Integrable Improvements, https://doi.org/10.1109/lsp.2024.3521379, fetched 2026-10-26
[S2] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
