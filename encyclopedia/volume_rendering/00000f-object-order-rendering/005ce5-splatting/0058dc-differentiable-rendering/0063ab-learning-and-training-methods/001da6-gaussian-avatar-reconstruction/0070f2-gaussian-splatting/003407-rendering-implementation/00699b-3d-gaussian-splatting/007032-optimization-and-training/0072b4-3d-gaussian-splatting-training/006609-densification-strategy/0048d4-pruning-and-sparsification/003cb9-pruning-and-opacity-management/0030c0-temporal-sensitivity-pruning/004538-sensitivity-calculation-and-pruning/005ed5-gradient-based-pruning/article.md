# Gradient-based Pruning

Also called: Gradient Magnitude Pruning

## Overview
Gradient-based Pruning is a training-time sparsification technique for 3D Gaussian Splatting that removes Gaussians contributing minimally to the optimization process by evaluating the magnitude of their associated gradients [S1]. It distinguishes itself from sibling methods, such as [Statistical Sensitivity-Driven Online Sparsification](14437-statistical-sensitivity-driven-online-sparsification.md), by directly measuring the gradient signal intensity rather than statistical variance or opacity thresholds, allowing for efficient GPU implementation while effectively eliminating visual artifacts like floaters [S1].

## How it works
**Inputs and Outputs:**
- **Input:** The set of 3D Gaussians $\mathcal{G}$ currently active in the scene, their parameters (position $\mathbf{t}$, covariance $\mathbf{\Sigma}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$), and the accumulated gradient magnitudes $\mathbf{g}$ for each Gaussian during the current training phase.
- **Output:** A pruned set of Gaussians $\mathcal{G}' \subset \mathcal{G}$ with redundant entries removed.

**Data Structures:**
- **Gradient Accumulator:** A vector or array of size $|\mathcal{G}|$ storing the cumulative gradient magnitude for each Gaussian. For EfficientGS [S1], this is a cumulative sum. For Kim et al. [S1], it may include separate components for position and SH coefficients.
- **Threshold:** A scalar value $\tau$ determining the minimum gradient magnitude required to retain a Gaussian.

**Steps:**
1. **Gradient Accumulation:** During each training iteration, after the backward pass, compute the gradient magnitude for each Gaussian.
   - For position-based pruning, calculate the Euclidean norm of the positional gradient $\nabla_{\mathbf{t}_i} L$ for Gaussian $i$.
   - For SH-based pruning (Kim et al. [S1]), calculate the norm of the gradient with respect to spherical harmonics coefficients $\nabla_{\mathbf{c}_i} L$.
   - Accumulate these magnitudes over time. EfficientGS [S1] uses a cumulative sum:
     $$ G_i^{(t)} = G_i^{(t-1)} + \|\nabla_{\mathbf{p}_i} L\| $$
     where $\mathbf{p}_i$ represents the parameters of interest (position or SH).
2. **Thresholding:** Compare the accumulated gradient magnitude $G_i$ against a threshold $\tau$.
   - If $G_i < \tau$, the Gaussian is marked for removal.
   - The threshold $\tau$ can be static or adaptive based on the distribution of gradient magnitudes in the current batch.
3. **Pruning:** Remove all marked Gaussians from the scene representation.
   - Trimming the Fat [S1] and ELMGS [S1] apply this to remove up to 75% of Gaussians.
4. **Optional: Opacity Reset:** When a Gaussian is pruned, its opacity may be reset or redistributed to neighboring Gaussians to alleviate interference with residual distribution tracking [S1].

**Parameters:**
- **Threshold $\tau$:** Not given in the sources. Typically determined empirically or via percentile-based selection (e.g., remove bottom 10-25% of Gaussians by gradient magnitude).
- **Accumulation Window:** The number of iterations over which gradients are summed. EfficientGS [S1] uses cumulative sums, implying a long-term or epoch-spanning window.

**Edge Cases and Pitfalls:**
- **Precision:** Gradient magnitudes can vary significantly across scenes. Normalization or adaptive thresholds are necessary to avoid removing too many or too few Gaussians.
- **Degenerate Input:** Gaussians with zero gradient (e.g., occluded or static background) may be pruned prematurely if the threshold is too low.
- **SH Gradients:** Incorporating SH gradients (Kim et al. [S1]) adds complexity but refines pruning by considering color/appearance contribution, not just geometry.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_pruning(gaussians, accumulated_gradients, threshold):
    """
    Prunes Gaussians with accumulated gradient magnitude below threshold.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - id: Unique identifier
            - accumulated_grad: Current cumulative gradient magnitude
        accumulated_gradients: List of floats, same length as gaussians, 
                               representing the accumulated gradient magnitude for each Gaussian.
        threshold: Float, the minimum accumulated gradient magnitude to keep a Gaussian.
        
    Returns:
        pruned_gaussians: List of Gaussian objects that survived pruning.
    """
    pruned_gaussians = []
    for i, gaussian in enumerate(gaussians):
        grad_mag = accumulated_gradients[i]
        if grad_mag >= threshold:
            pruned_gaussians.append(gaussian)
        else:
            # Gaussian is pruned. 
            # Optional: Reset opacity or redistribute weight [S1]
            pass
    return pruned_gaussians
```

## Subtopics

### [Utilization-Based Pruning](18426-utilization-based-pruning.md)
Utilization-Based Pruning (UBP) is a training-time sparsification technique for 3D Gaussian Splatting that removes Gaussians with negligible contribution to the rendered image gradient, replacing opacity reset to alleviate interference with residual distribution tracking and compress scene representation. It distinguishes itself from sibling methods by evaluating the gradient of the rendered image with respect to Gaussian positions rather than the loss gradient, allowing for efficient GPU implementation while effectively eliminating visual artifacts like floaters.

## Operating characteristics
- **Time Complexity:** $O(N)$ per iteration for gradient accumulation and pruning, where $N$ is the number of Gaussians. Cumulative sum updates are $O(1)$ per Gaussian per iteration.
- **Memory Complexity:** $O(N)$ to store accumulated gradients.
- **Measured Performance:** Trimming the Fat [S1] and ELMGS [S1] remove 75% of Gaussians while maintaining high visual quality.
- **Strengths:** Effective at removing redundant Gaussians and floaters. Efficient on GPU due to simple magnitude comparisons.
- **Limitations:** Requires careful threshold selection. May remove Gaussians that are temporarily occluded but important for later views.
- **When to Use:** When computational efficiency is critical and visual quality must be maintained with fewer Gaussians.
- **When Not to Use:** When the scene has dynamic elements where temporary occlusion is common, or when SH details are critical and position-only pruning is insufficient.

## Background
Gradient-based Pruning originated as a method to improve the efficiency of 3D Gaussian Splatting training by removing Gaussians that contribute minimally to the optimization process [S1]. It has been adopted and extended by several methods, including EfficientGS [S1], GDGS [S1], Trimming the Fat [S1], ELMGS [S1], and Kim et al. [S1]. Kim et al. [S1] extended the approach by incorporating SH gradients alongside positional gradients.

## Sources
[S1] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
