# Soft Pruning

Also called: Hessian-aware pruning, L1 loss pruning

## Overview
Soft Pruning is a density control method in 3D Gaussian Splatting that removes redundant Gaussian primitives during the densification stage to maintain a robust Hessian approximation [S1, S2]. It differs from its sibling [Hard Pruning](5591-hard-pruning.md) by occurring *during* the densification phase (specifically before opacity resets) rather than after, allowing the model to recover performance through subsequent refinement iterations [S1, S2].

## How it works
Soft Pruning integrates into the standard 3D Gaussian Splatting training pipeline by triggering removals based on the convergence of the L1 loss [S1, S2].

**Inputs and Outputs:**
- **Input:** The current set of 3D Gaussians $\mathcal{G}$, the current training iteration $t$, and the accumulated L1 loss metric.
- **Output:** A reduced set of Gaussians $\mathcal{G}'$ with a specified pruning ratio.

**Data Structures:**
The method operates on the existing Gaussian primitive structure defined in the parent [Density Control and Densification](5337-density-control-and-densification.md): position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients $\mathbf{c}_i$ [S2].

**Steps:**
1. **Monitor L1 Loss:** Track the L1 loss during training. The method relies on the observation that L1 loss converges quickly and remains small after iteration 6000, except immediately following an opacity reset [S1].
2. **Identify Pruning Intervals:** Schedule pruning operations immediately before opacity resets. The standard intervals are iterations 6000, 9000, and 12000 [S1].
3. **Evaluate Primitives:** At each scheduled iteration, evaluate each Gaussian $g_i$ using an "Efficient Pruning Score" [S2]. The sources do not provide the explicit mathematical formula for this score, only that it is used to determine which Gaussians to remove [S2].
4. **Remove Gaussians:** Remove the Gaussians with the lowest pruning scores (highest redundancy) until the target pruning ratio is reached [S1, S2].
5. **Proceed to Opacity Reset:** Immediately after pruning, perform the opacity reset as per the standard densification pipeline [S1].

**Parameters:**
- **Pruning Ratio:** The fraction of Gaussians to remove. Experiments show that visual fidelity is preserved even at high ratios, specifically 80% [S1].
- **Iterations:** The specific training steps where pruning occurs. Standard values are 6000, 9000, and 12000 [S1].

**Edge Cases and Pitfalls:**
- **Timing Sensitivity:** Pruning must occur *before* the opacity reset to ensure the Hessian approximation remains robust [S1]. Pruning after the reset may lead to suboptimal convergence.
- **Loss Convergence:** The method assumes L1 loss has converged. Applying it too early (before iteration 6000) may remove primitives that are still necessary for learning [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_pruning(gaussians, current_iteration, pruning_ratio=0.8):
    """
    Perform soft pruning on the set of Gaussians.
    
    Args:
        gaussians: List of Gaussian primitives.
        current_iteration: Current training step.
        pruning_ratio: Fraction of Gaussians to remove (e.g., 0.8 for 80%).
        
    Returns:
        pruned_gaussians: List of remaining Gaussian primitives.
    """
    
    # Standard pruning intervals defined in [S1]
    pruning_iterations = [6000, 9000, 12000]
    
    if current_iteration in pruning_iterations:
        # Calculate pruning score for each Gaussian
        # Note: The exact formula for the "Efficient Pruning Score" is not provided in sources [S2]
        # [G] Implementation requires a score function that correlates with redundancy,
        # likely involving opacity and gradient magnitude as per general densification principles [S2].
        scores = calculate_pruning_scores(gaussians)
        
        # Sort Gaussians by score (ascending, assuming lower score = more redundant)
        sorted_indices = argsort(scores)
        
        # Determine number of Gaussians to remove
        num_to_remove = int(len(gaussians) * pruning_ratio)
        
        # Remove the bottom-scoring Gaussians
        pruned_gaussians = [g for i, g in enumerate(gaussians) if i not in sorted_indices[:num_to_remove]]
        
        return pruned_gaussians
    else:
        return gaussians
```

## Subtopics

### [Hard Pruning](5591-hard-pruning.md)
Hard Pruning is a density control method in 3D Gaussian Splatting that reduces model size by a constant ratio at fixed intervals after the densification stage. It differs from Soft Pruning by occurring post-densification rather than during it, relying on subsequent refinement iterations to recover performance.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Reduces the number of Gaussians by approximately 90% when combined with Hard Pruning, significantly decreasing memory usage [S2].
- **Measured Performance:** Preserves visual fidelity at 80% pruning ratio [S1]. Increases rendering and training speed while maintaining image quality [S2].
- **Strengths:** Maintains robust Hessian approximation by timing pruning with L1 loss convergence [S1]. Allows for high compression ratios without significant quality loss [S1, S2].
- **Limitations:** Requires careful scheduling of pruning intervals relative to opacity resets [S1].
- **When to use:** When training 3D Gaussian Splatting models and aiming to reduce computational load during the densification phase without sacrificing final image quality [S1, S2].
- **When not:** When the L1 loss has not yet converged (before iteration 6000) or when opacity resets are not part of the training pipeline [S1].

## Background
Soft Pruning was introduced in the "Speedy-Splat" paper [S1, S2]. It was developed to address the need for faster rendering and training speeds in 3D Gaussian Splatting by reducing the number of primitives. The method leverages the observation that L1 loss converges quickly, allowing for aggressive pruning without harming the model's ability to learn [S1].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
