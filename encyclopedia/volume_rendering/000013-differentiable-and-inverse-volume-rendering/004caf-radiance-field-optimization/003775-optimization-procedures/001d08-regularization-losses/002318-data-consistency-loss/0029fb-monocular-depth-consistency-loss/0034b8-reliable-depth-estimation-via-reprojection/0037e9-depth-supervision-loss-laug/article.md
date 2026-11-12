# Depth Supervision Loss (Laug)

Also called: Depth Loss with Adaptive Masking, Laug

## Overview
Depth Supervision Loss (Laug) is a data consistency regularization term used in differentiable volume rendering to constrain the rendered depth to match ground-truth depth estimates. It addresses the ambiguity of monocular depth estimation by applying a dynamic reliability mask that stabilizes training, particularly in the early stages. It differs from its siblings, [Depth Consistency Check](9363-depth-consistency-check-depth-outlier-removal.md) and [Multi-view Consistency Depth Filtering](12319-multi-view-consistency-depth-filtering.md), by operating as a direct pixel-wise regression loss with adaptive masking rather than a post-hoc filtering or consistency check mechanism.

## How it works
The method computes a weighted L1 reconstruction loss between the warped rendered depth and the ground-truth depth. It employs a two-phase strategy: an initial phase with a static scene-specific mask, followed by a dynamic phase that replaces masked ground-truth values and updates the mask based on depth comparisons.

### Inputs and Outputs
*   **Inputs:**
    *   $d_{render}$: Warped rendered depth map.
    *   $d_{gt}$: Ground-truth depth map.
    *   $m_{init}$: Initial scene-specific binary mask (1 for valid pixels, 0 for invalid).
    *   $I$: Current training iteration index.
    *   $I_{total}$: Total number of training iterations ($opt.iterations$).
*   **Outputs:**
    *   $L_{depth}$: Scalar loss value.

### Data Structures
*   **Depth Maps:** 2D arrays of floating-point values, shape $(H, W)$.
*   **Masks:** 2D boolean or binary float arrays, shape $(H, W)$.

### Algorithm Steps
1.  **Initialize Mask:** Set the adaptive mask $m_{gt}$ to the initial scene-specific mask $m_{init}$.
2.  **Check Iteration Threshold:** Determine if the current iteration $I$ exceeds one-third of the total iterations ($I > I_{total} / 3$).
3.  **Adaptive Masking (Post-Threshold):**
    *   If $I > I_{total} / 3$:
        *   Replace masked ground-truth depth values with a constant value $C = 0.75$. Specifically, for pixels where $m_{gt} == 0$, set $d_{gt}[i] = 0.75$.
        *   Update the mask: $m_{gt} = (d_{render} < d_{gt}) \land m_{init}$. This selects pixels where the rendered depth is closer than the ground truth, restricted to the initial valid region.
    *   Else:
        *   Keep $m_{gt} = m_{init}$.
        *   Keep $d_{gt}$ unchanged.
4.  **Compute Loss:** Calculate the masked L1 loss and scale by 2.0.

### Formula
The loss is defined as:
$$ L_{depth} = 2 \cdot \frac{1}{N} \sum_{i=1}^{N} |d_{render, i} \cdot m_{gt, i} - d_{gt, i} \cdot m_{gt, i}| $$
Where:
*   $N$ is the total number of pixels.
*   $d_{render, i}$ is the rendered depth at pixel $i$.
*   $d_{gt, i}$ is the ground-truth depth at pixel $i$ (potentially modified in step 3).
*   $m_{gt, i}$ is the adaptive mask value at pixel $i$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_depth_supervision_loss(d_render, d_gt, m_init, iteration, total_iterations):
    """
    Computes the Depth Supervision Loss (Laug) with adaptive masking.
    
    Args:
        d_render: numpy array (H, W), rendered depth.
        d_gt: numpy array (H, W), ground truth depth.
        m_init: numpy array (H, W), initial binary mask (0 or 1).
        iteration: int, current training iteration.
        total_iterations: int, total number of training iterations.
        
    Returns:
        float: The computed loss value.
    """
    # Step 1: Initialize adaptive mask
    m_gt = m_init.copy()
    
    # Step 2: Check iteration threshold
    threshold = total_iterations / 3.0
    
    if iteration > threshold:
        # Step 3a: Replace masked GT depth values with 0.75
        # Note: The source implies replacing values where the mask is 0,
        # or potentially all masked-out regions. Based on "replace masked GT depth values",
        # we assume pixels where m_gt is 0 are set to 0.75.
        d_gt[m_gt == 0] = 0.75
        
        # Step 3b: Update mask to (depth < gt_depth) * mask[0:1]
        # m_init is used as the base constraint.
        # Condition: d_render < d_gt
        condition = d_render < d_gt
        m_gt = (condition.astype(float) * m_init)
    else:
        # Step 3c: Early training phase, no modification
        pass
        
    # Step 4: Compute L1 loss
    # Masked rendered depth
    masked_render = d_render * m_gt
    # Masked GT depth
    masked_gt = d_gt * m_gt
    
    # Absolute difference
    abs_diff = np.abs(masked_render - masked_gt)
    
    # Mean over N pixels (N is total pixels, not just valid ones, as per formula 1/N sum)
    loss = np.mean(abs_diff)
    
    # Scale by 2.0
    loss = 2.0 * loss
    
    return loss
```

### Parameters
*   `opt.iterations`: Total training iterations. The adaptive masking logic triggers at $I > opt.iterations / 3$.
*   Constant replacement value: $0.75$.
*   Loss scaling factor: $2.0$.

### Edge Cases and Pitfalls
*   **Precision:** Ensure $d_{render}$ and $d_{gt}$ are in the same coordinate system and scale (e.g., meters). The constant $0.75$ is unit-dependent; if depths are in millimeters, this value is invalid. The source does not specify the unit, but $0.75$ suggests meters.
*   **Boundary Conditions:** The mask update $(d_{render} < d_{gt})$ may result in empty masks if the network is poorly initialized. This is mitigated by the initial mask $m_{init}$.
*   **Degenerate Input:** If $m_{init}$ is all zeros, the loss is zero. If $d_{gt}$ contains NaNs, they should be masked out in $m_{init}$.

## Subtopics

### Robust Depth Supervision
A variant that identifies a reliable subset of depth points $D_{reliable}^m$ using scheduled parameters $\epsilon_t^m$ (threshold) and $\epsilon_o^m$ (offset) to filter out occluded or distant points. It computes a Lidar Depth Loss $L_{ds} = L_{depth} + L_{sight}$. $L_{depth}$ is an L2 loss between rendered and ground truth depth. $L_{sight}$ enforces the weight distribution $w(t)$ along the ray to resemble a Gaussian $N(\hat{D}_i, \epsilon_n)$ centered at the ground truth depth $\hat{D}_i$. The probability mass $N_i$ is computed exactly using the Gaussian CDF rather than mid-point approximation. The reliable subset is defined as $D_{reliable}^m = \{D_i | D_i \le \epsilon_t^m, D_i \le \hat{D}_i + \epsilon_o^m, D_i \in D\}$. The parameters $\epsilon_t^m$ and $\epsilon_o^m$ are scheduled exponentially: $\epsilon_t^m = \min\{\alpha_t \epsilon_t^{m-1}, \epsilon_t^0\}$ with $\alpha_t > 1$, and $\epsilon_o^m = \max\{\alpha_o \epsilon_o^{m-1}, \epsilon_o^0\}$ with $\alpha_o < 1$.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of pixels, due to element-wise operations.
*   **Memory Complexity:** $O(N)$ for storing the depth maps and masks.
*   **Strengths:** Stabilizes training by ignoring unreliable depth estimates early on. The adaptive mask prevents the network from overfitting to noisy or occluded ground-truth depth.
*   **Limitations:** Relies on the quality of the initial mask $m_{init}$ and the ground-truth depth. The constant $0.75$ is heuristic and may not generalize across different scenes or units.
*   **When to use:** When training with sparse views or noisy monocular depth estimates where early training instability is a concern.
*   **When not to use:** When high-precision, metric depth is available and reliable for all pixels, or when computational resources are extremely limited (though the cost is low).

## Background
The Depth Supervision Loss (Laug) with adaptive masking was introduced in the context of Autonomous Vehicle Multi-View Stereo (AVM-GS) [S1, unofficial]. It is designed to handle the challenges of training neural radiance fields with limited view coverage and noisy depth data. The method builds on the idea of dynamic masking to improve convergence stability.

## Sources
[S1] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S2] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S3] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
