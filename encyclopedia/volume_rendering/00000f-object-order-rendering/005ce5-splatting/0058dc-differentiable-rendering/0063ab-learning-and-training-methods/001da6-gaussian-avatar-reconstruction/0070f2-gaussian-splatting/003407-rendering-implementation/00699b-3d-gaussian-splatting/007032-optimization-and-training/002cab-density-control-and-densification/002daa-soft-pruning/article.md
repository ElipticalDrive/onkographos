# Soft Pruning

Also called: Hessian-aware pruning, L1 loss pruning

## Overview
Soft Pruning is a density control method in 3D Gaussian Splatting that reduces the number of Gaussian primitives by removing those with low contribution to the reconstruction loss, specifically to maintain a robust Hessian approximation during optimization [S1, S2]. It differs from its sibling [Hard Pruning](5591-hard-pruning.md) by occurring *during* the densification stage (immediately before opacity resets) rather than after, allowing the subsequent training iterations to refine the remaining primitives [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of $N$ 3D Gaussians, each with position $\mathbf{p}$, covariance $\mathbf{S}$ (or scale/rotation), opacity $\alpha$, and accumulated L1 loss $\mathcal{L}_{L1}$ [G].
*   **Output:** A reduced set of Gaussians with indices of low-loss primitives removed [S1].

**Data Structures:**
*   Each Gaussian $i$ maintains a scalar metric: the accumulated L1 loss $\mathcal{L}_{L1}^{(i)}$ over the current training phase [G].
*   A global counter or flag tracks the current iteration number $t$ [G].

**Parameters:**
*   **Pruning Ratio ($r$):** The fraction of Gaussians to remove. Typical value: $0.8$ (80% pruning) [S1].
*   **Reset Iterations:** Specific iteration numbers where pruning is triggered. Typical values: $6000, 9000, 12000$ [S1].

**Steps:**
1.  **Check Trigger:** At the start of an iteration $t$, check if $t$ matches a predefined reset iteration (e.g., $6000, 9000, 12000$) [S1].
2.  **Compute Score:** If triggered, calculate the pruning score for each Gaussian $i$. The score is based on the accumulated L1 loss $\mathcal{L}_{L1}^{(i)}$. Gaussians with smaller loss are considered less important [S1].
3.  **Rank and Select:** Sort all $N$ Gaussians by their L1 loss in ascending order. Remove the fraction $r$ of Gaussians with the lowest loss [S1].
4.  **Remove:** Delete the selected Gaussians from the scene representation [S1].
5.  **Proceed:** Continue with the standard densification pipeline (cloning/splitting) and then the opacity reset [S1].

**Edge Cases and Pitfalls:**
*   **Timing:** Pruning must occur *before* the opacity reset. If performed after, it becomes "Hard Pruning," which relies on later refinement rather than maintaining the Hessian approximation during the critical densification phase [S1, S2].
*   **Loss Convergence:** The method relies on the observation that L1 loss converges quickly and remains low except after resets. Pruning when loss is uniformly low is safe [S1].
*   **High Pruning Ratios:** Visual fidelity is preserved at 80% pruning [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_pruning(gaussians, current_iteration, reset_iterations, pruning_ratio):
    """
    gaussians: List of Gaussian objects, each with attribute 'l1_loss'
    current_iteration: int
    reset_iterations: List[int], e.g., [6000, 9000, 12000]
    pruning_ratio: float, e.g., 0.8
    """
    if current_iteration not in reset_iterations:
        return gaussians

    # Step 1: Sort Gaussians by L1 loss (ascending)
    # Gaussians with lower loss are less important
    sorted_gaussians = sorted(gaussians, key=lambda g: g.l1_loss)

    # Step 2: Determine number of Gaussians to prune
    num_to_prune = int(len(sorted_gaussians) * pruning_ratio)

    # Step 3: Keep the top (1 - ratio) Gaussians with highest loss
    # The bottom 'num_to_prune' are discarded
    gaussians = sorted_gaussians[num_to_prune:]

    return gaussians
```

## Subtopics

### [Hard Pruning](5591-hard-pruning.md)
A density control method that reduces model size by a constant ratio at fixed intervals after the densification stage. It differs from Soft Pruning by occurring post-densification rather than during it, relying on subsequent refinement iterations to recover performance. [Hard Pruning](5591-hard-pruning.md)

## Operating characteristics
*   **Time Complexity:** Sorting the Gaussians by loss takes $O(N \log N)$, where $N$ is the number of primitives. This is performed infrequently (e.g., 3 times in a typical training run) [S1].
*   **Memory Complexity:** Reduces memory footprint by removing primitives. Soft and Hard Pruning together reduce the number of Gaussians by approximately 90%, increasing rendering and training speed while maintaining image quality [S2].
*   **Strengths:** Maintains Hessian approximation quality by removing only low-contribution primitives before opacity resets. Allows for aggressive pruning (80%) without visual degradation [S1].
*   **Limitations:** Requires tracking accumulated L1 loss per Gaussian. Timing is critical; incorrect placement in the pipeline degrades performance [S1, S2].

## Background
Soft Pruning was introduced in the context of "Speedy-Splat," a method for fast 3D Gaussian Splatting [S1, S2]. It addresses the issue of Hessian approximation degradation as training progresses and L1 loss becomes small [S1]. By pruning before opacity resets, it maintains a robust Hessian approximation [S1].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
