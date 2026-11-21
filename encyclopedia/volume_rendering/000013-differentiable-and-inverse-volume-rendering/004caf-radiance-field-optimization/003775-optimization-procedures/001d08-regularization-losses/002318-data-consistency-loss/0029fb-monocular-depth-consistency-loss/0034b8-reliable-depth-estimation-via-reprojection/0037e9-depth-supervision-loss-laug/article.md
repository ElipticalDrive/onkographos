# Depth Supervision Loss (Laug)

Also called: Depth Loss with Adaptive Masking, Laug

## Overview
Depth Supervision Loss (Laug) enforces geometric consistency by minimizing the discrepancy between rendered depth maps and ground-truth depth estimates during radiance field optimization. It distinguishes itself from sibling methods by combining a dynamic adaptive masking strategy to handle unreliable depth regions with a robust, gradient-stopped reliability weighting mechanism that prevents error propagation from noisy depth priors [S1, unofficial][S2].

## How it works
The method computes a weighted L1 reconstruction loss between the warped rendered depth and the ground-truth depth. It employs two distinct mechanisms for handling reliability: a time-dependent adaptive mask for standard depth consistency [S1, unofficial] and a reliability-based indicator mask for robust supervision [S2].

**Inputs:**
- $d_{render}$: Warped rendered depth map.
- $d_{gt}$: Ground-truth depth map.
- $m_{gt}$: Initial ground-truth mask (binary, 1 for valid, 0 for invalid).
- $z_m$: Multi-view consistent depth estimate.
- $z_a$: Single-view depth estimate.
- $m_a, m_m$: Reliability masks for single-view and multi-view depths, respectively.
- $N$: Number of pixels.
- $I$: Current training iteration.
- $I_{total}$: Total number of training iterations.

**Data Structures:**
- Depth maps are 2D arrays of floating-point values.
- Masks are 2D binary arrays (0 or 1).

**Parameters:**
- `opt.iterations`: Total training iterations. The adaptive masking logic triggers at $I_{total} / 3$ [S1, unofficial].
- Scaling factor: 2.0 [S1, unofficial].

**Steps:**

1. **Adaptive Masking (Standard L_depth):**
   - Load $d_{gt}$ and $m_{gt}$.
   - If $I > I_{total} / 3$:
     - Replace masked-out values in $d_{gt}$ with 0.75.
     - Update $m_{gt}$ to: $m_{gt} = (d_{render} < d_{gt}) \cdot m_{gt}$. This selects pixels where the rendered depth is closer than the ground truth, effectively masking out occluded or unreliable rear surfaces [S1, unofficial].
   - Compute the loss:
     $$ L_{depth} = 2 \cdot \frac{1}{N} \sum_{i=1}^{N} |d_{render, i} \cdot m_{gt, i} - d_{gt, i} \cdot m_{gt, i}| $$
     where $d_{render}$ is the warped rendered depth and $d_{gt}$ is the ground-truth depth [S1, unofficial].

2. **Robust Supervision (Laug):**
   - Use reliability masks $m_a$ and $m_m$ to weight the depth consistency between single-view ($z_a$) and multi-view ($z_m$) estimates.
   - Apply stop-gradient ($\hat{\nabla}$) to the reference terms to prevent gradient flow through unreliable depth estimates [S2].
   - Compute the loss:
     $$ L_{aug} = \mathbb{1}_{\{m_a=1\}} \odot ||z_m - \hat{\nabla}(z_a)||^2 + \mathbb{1}_{\{m_m=1\}} \odot ||\hat{\nabla}(z_m) - z_a||^2 $$
     where $\odot$ is element-wise multiplication, $\mathbb{1}$ is the indicator function, and $\hat{\nabla}$ denotes the stop-gradient operator [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_depth_loss(d_render, d_gt, m_gt, z_m, z_a, m_a, m_m, current_iter, total_iters):
    """
    Computes Depth Supervision Loss (Laug) combining adaptive masking and robust reliability weighting.
    """
    # --- Part 1: Adaptive Masking for L_depth [S1, unofficial] ---
    # Apply scene-specific mask logic if past 1/3 of training
    if current_iter > total_iters / 3:
        # Replace masked GT depth values with 0.75
        d_gt_masked = d_gt.clone()
        d_gt_masked[m_gt == 0] = 0.75
        
        # Update mask: keep pixels where rendered depth is closer than GT depth
        # Note: Source [S1, unofficial] implies this is applied to the valid region
        m_gt_new = (d_render < d_gt_masked).float() * m_gt
        m_gt = m_gt_new
        
        # Compute L1 loss with scaling factor 2.0
        # Only sum over valid pixels defined by m_gt
        valid_pixels = m_gt.sum()
        if valid_pixels > 0:
            l_depth = 2.0 * (torch.abs(d_render * m_gt - d_gt_masked * m_gt).sum() / valid_pixels)
        else:
            l_depth = torch.tensor(0.0)
    else:
        # Standard L1 loss without adaptive replacement/mask update
        valid_pixels = m_gt.sum()
        if valid_pixels > 0:
            l_depth = 2.0 * (torch.abs(d_render * m_gt - d_gt * m_gt).sum() / valid_pixels)
        else:
            l_depth = torch.tensor(0.0)

    # --- Part 2: Robust Supervision for Laug [S2] ---
    # Apply stop-gradient to reference terms
    z_a_stopped = z_a.detach()
    z_m_stopped = z_m.detach()
    
    # Term 1: Multi-view vs Single-view (weighted by single-view reliability)
    term1 = (m_a == 1).float() * (z_m - z_a_stopped).pow(2)
    
    # Term 2: Single-view vs Multi-view (weighted by multi-view reliability)
    term2 = (m_m == 1).float() * (z_m_stopped - z_a).pow(2)
    
    # Combine losses
    l_aug = term1.sum() + term2.sum()
    
    return l_depth, l_aug
```

**Edge Cases and Pitfalls:**
- **Precision:** The comparison $d_{render} < d_{gt}$ in the adaptive mask can be sensitive to depth quantization errors. [S1, unofficial]
- **Boundaries:** The adaptive mask update at $I_{total}/3$ may cause sudden changes in the loss landscape if the rendered depth is significantly inaccurate at that stage. [S1, unofficial]
- **Degenerate Input:** If $m_{gt}$ is all zeros, the L1 loss division by $N$ (or valid pixels) must handle division by zero. [G]

## Subtopics

### [Robust Depth Supervision](12870-robust-depth-supervision.md)
Handles defocus blur ambiguity by sampling front-most depth values and using exact Gaussian CDFs, avoiding the blending of front/rear depths.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels $N$ for both L1 and L2 loss computations [S1, unofficial][S2].
- **Memory Complexity:** Requires storage for depth maps and masks, proportional to image resolution [S1, unofficial][S2].
- **Strengths:** The adaptive masking stabilizes training by ignoring unreliable rear-surface depths early on, while the reliability masks in Laug prevent noisy depth estimates from corrupting the optimization [S1, unofficial][S2].
- **Limitations:** The method relies on the quality of the initial depth estimates and masks. If the reliability masks are inaccurate, the loss may guide the model incorrectly [S2]. The adaptive mask logic is specific to the iteration schedule and may not generalize to all training regimes [S1, unofficial].

## Background
The method originates from implementations in AVM-GS [S1, unofficial] and Simple-RF [S2]. It is used in differentiable volume rendering and radiance field optimization to regularize depth consistency, particularly in sparse-view settings where depth estimation is ambiguous [S2].

## Sources
[S1] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
