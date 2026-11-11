# Soft Pruning

Also called: Hessian-aware pruning, L1 loss pruning

## Overview
Soft Pruning is a density control method in 3D Gaussian Splatting that removes Gaussians with low training loss contribution to maintain a robust Hessian approximation [S1, S2]. It differs from [Hard Pruning](5591-hard-pruning.md) by executing during the densification stage immediately before opacity resets, rather than after [S1, S2].

## How it works
Soft Pruning is integrated into the densification pipeline to remove Gaussians when the L1 loss is small, ensuring stable gradient signals for Hessian-based optimization [S1].

**Inputs and Outputs:**
*   **Input:** The current set of 3D Gaussians $\{G_i\}$, the current training iteration $t$, the pruning ratio $R$, and the predefined reset iterations [S1].
*   **Output:** A reduced set of Gaussians $\{G_i'\}$ after removing the lowest-scoring fraction [S1].

**Data Structures:**
*   Each Gaussian $G_i$ retains standard attributes: mean position $\mathbf{p}_i$, covariance matrix $\Sigma_i$ (scale/rotation), opacity $\alpha_i$, and spherical harmonic coefficients [G].
*   The pruning decision relies on an "Efficient Pruning Score" [S2], though the exact mathematical formulation is not provided in the sources [S2]. Pruning is triggered when the L1 loss is small [S1].

**Steps:**
1.  **Check Pruning Condition:** Determine if the current iteration $t$ matches a predefined opacity reset iteration [S1].
2.  **Evaluate Pruning Score:** For each Gaussian $G_i$, compute its pruning score [S2]. Gaussians with low scores (low loss contribution) are identified as candidates for removal [S2].
3.  **Apply Pruning Ratio:** Remove the bottom $R$ percent of Gaussians based on the computed scores [S1].
4.  **Remove Gaussians:** Delete the selected Gaussians from the scene representation [S1].
5.  **Continue Training:** Proceed with the opacity reset and subsequent densification steps (cloning/splitting) as defined in the parent [Density Control and Densification](5337-density-control-and-densification.md) [S1, S2].

**Formulas:**
Not given in the sources. The pruning score is referred to as an "Efficient Pruning Score" [S2].

**Parameters:**
*   **Pruning Ratio ($R$):** The fraction of Gaussians to remove. Typical value: 0.8 (80%) [S1].
*   **Reset Iterations:** The iterations at which Soft Pruning is applied. Typical values: 6000, 9000, 12000 [S1].

**Edge Cases and Pitfalls:**
*   **Premature Pruning:** Pruning too early, before the L1 loss converges, may remove Gaussians that are still learning [S1].
*   **Opacity Reset Interaction:** Soft Pruning is performed *immediately before* opacity resets to maintain the Hessian approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_pruning(gaussians, current_iteration, reset_iterations, pruning_ratio):
    """
    Perform Soft Pruning on the set of Gaussians.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - p: mean position
                   - alpha: opacity
                   - score: accumulated pruning score (e.g., L1 loss or gradient magnitude)
        current_iteration: int, current training iteration
        reset_iterations: list of int, iterations where opacity resets occur
        pruning_ratio: float, fraction of Gaussians to remove (e.g., 0.8)
        
    Returns:
        pruned_gaussians: List of Gaussian objects after pruning
    """
    # Step 1: Check if pruning is required at this iteration
    if current_iteration not in reset_iterations:
        return gaussians
    
    # Step 2: Evaluate pruning score
    # [G] The exact "Efficient Pruning Score" formula is not provided in the sources [S2].
    # Implementers should rank Gaussians by accumulated L1 loss or gradient magnitude.
    # Gaussians with lower scores are pruned.
    for g in gaussians:
        g.pruning_score = g.score 
        
    # Step 3: Sort Gaussians by pruning score (ascending, so lowest score first)
    gaussians.sort(key=lambda g: g.pruning_score)
    
    # Step 4: Calculate number of Gaussians to remove
    num_to_remove = int(len(gaussians) * pruning_ratio)
    
    # Step 5: Remove the bottom R percent
    pruned_gaussians = gaussians[num_to_remove:]
    
    return pruned_gaussians
```

## Subtopics

### [Hard Pruning](5591-hard-pruning.md)
Hard Pruning is a density control method in 3D Gaussian Splatting that reduces model size by a constant ratio at fixed intervals after the densification stage. It differs from Soft Pruning by occurring post-densification rather than during it, relying on subsequent refinement iterations to recover performance.

## Operating characteristics
*   **Time Complexity:** The pruning step is performed only at fixed iterations (e.g., 6000, 9000, 12000), so the amortized cost per iteration is low [S1].
*   **Memory Complexity:** Together with Hard Pruning, the modalities reduce the Gaussian count by approximately 90% [S2].
*   **Performance:** Improves rendering and training speed by reducing the number of primitives [S2]. Visual fidelity is preserved even with high pruning ratios (80%) [S1].
*   **Strengths:** Maintains Hessian approximation quality by pruning when loss is low [S1]. Integrates seamlessly into the existing densification pipeline [S2].
*   **Limitations:** Requires careful timing relative to opacity resets [S1]. The specific "Efficient Pruning Score" formula is not detailed, which may require empirical tuning [S2].

## Background
Soft Pruning was introduced in the "Speedy-Splat" paper [S1, S2]. It was developed to address the need for faster training and rendering in 3D Gaussian Splatting by reducing the number of Gaussians without sacrificing quality [S2]. The method leverages the observation that L1 loss converges quickly and remains low after iteration 6000, allowing for aggressive pruning [S1].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
