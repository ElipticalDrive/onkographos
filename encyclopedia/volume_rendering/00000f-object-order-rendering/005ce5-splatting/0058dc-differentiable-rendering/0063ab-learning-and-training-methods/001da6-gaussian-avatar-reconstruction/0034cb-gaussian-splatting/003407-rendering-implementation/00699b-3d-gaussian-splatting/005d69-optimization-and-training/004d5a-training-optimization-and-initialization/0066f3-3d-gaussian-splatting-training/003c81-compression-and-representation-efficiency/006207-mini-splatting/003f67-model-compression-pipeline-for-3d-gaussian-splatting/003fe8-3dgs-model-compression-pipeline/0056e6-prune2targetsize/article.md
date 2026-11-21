# Prune2TargetSize

Also called: Top-down pruning and recovery

## Overview
Prune2TargetSize is an algorithm designed to reduce a pretrained 3D Gaussian Splatting (3DGS) model to a precise target number of splats while maintaining visual quality [S1]. It solves the problem of unpredictable model size in standard 3DGS training, where hyperparameters like cloning thresholds and densification intervals interact in complex, scene-dependent ways, making it impossible to guarantee a specific final splat count [S1]. Unlike bottom-up approaches that grow splats to a target, this method uses a top-down strategy of iterative pruning and recovery, which has been found to perform better in practice [S1].

## How it works
**Inputs:**
- $G$: A pretrained 3DGS model [S1].
- $d$: The target number of splats for the output layer [S1].
- $r$: The maximum pruning ratio per iteration (hyperparameter) [S1].
- $I$: Training view images [S1].
- $P$: Camera poses associated with images $I$ [S1].

**Outputs:**
- $G_L$: A list of $d$ splats, sorted in descending order of global significance score [S1].

**Data Structures:**
- The 3DGS model $G$ consists of a set of splats $G_i$. Each splat typically includes position, scale, rotation, opacity, and spherical harmonic coefficients, though the internal layout is inherited from the base 3DGS implementation [15] cited in [S1].
- Global significance score $S$: A scalar value for each splat, computed by `CalGS` [S1]. The exact formula for `CalGS` is not provided in the source text, but it references external work [10] for the calculation.

**Algorithm Steps:**
1. Initialize $G^* \leftarrow G$ [S1].
2. Compute current splat count $n = \text{NumOfGS}(G^*)$ [S1].
3. While $n > d$:
    a. Compute global significance scores $S$ for all splats in $G^*$ using $I$ and $P$ via `CalGS` [S1].
    b. If $n \times (1 - r) > d$:
        i. Prune the bottom $r$ fraction of splats (those with the lowest $S$) from $G^*$ [S1].
        ii. Run Recovery training on the pruned $G^*$ with `IsRefinementIteration=True` [S1]. This allows cloning, splitting, and pruning [S1].
        iii. Update $n = \text{NumOfGS}(G^*)$ [S1].
    c. Else (if $n \times (1 - r) \le d$):
        i. Prune $G^*$ to remove exactly $n - d$ splats (lowest $S$) [S1].
        ii. Run Finetune training on $G^*$ with `IsRefinementIteration=False` [S1]. This disables cloning, splitting, and pruning [S1].
        iii. Update $n = \text{NumOfGS}(G^*)$ [S1].
4. Compute final global significance scores $S$ for the pruned model $G^*$ [S1].
5. Sort splats in $G^*$ in descending order of $S$ to produce $G_L$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Prune2TargetSize(G_pretrained, target_size_d, max_pruning_ratio_r, images_I, poses_P):
    """
    Reduces a pretrained 3DGS model to a precise target number of splats.
    
    Args:
        G_pretrained: Pretrained 3DGS model.
        target_size_d: Target number of splats.
        max_pruning_ratio_r: Max fraction of splats to prune in one step.
        images_I: List of training images.
        poses_P: List of camera poses.
        
    Returns:
        G_L: List of splats sorted by significance.
    """
    # Initialize
    G_star = G_pretrained
    n = count_splats(G_star)
    
    while n > target_size_d:
        # Compute global significance scores
        # Note: CalGS implementation details are in external reference [10]
        scores = CalGS(G_star, images_I, poses_P)
        
        # Determine pruning strategy
        if n * (1 - max_pruning_ratio_r) > target_size_d:
            # Prune a fraction r
            num_to_prune = int(n * max_pruning_ratio_r)
            # Identify indices of splats with lowest scores
            sorted_indices = argsort_asc(scores)
            prune_indices = sorted_indices[:num_to_prune]
            
            # Prune splats
            G_star = remove_splats(G_star, prune_indices)
            
            # Recovery training: allows cloning, splitting, pruning
            G_star = RecoveryTraining(G_star, images_I, poses_P, is_refinement=True)
            
            # Update count
            n = count_splats(G_star)
        else:
            # Prune to exact target
            num_to_prune = n - target_size_d
            sorted_indices = argsort_asc(scores)
            prune_indices = sorted_indices[:num_to_prune]
            
            # Prune splats
            G_star = remove_splats(G_star, prune_indices)
            
            # Finetune training: disables cloning, splitting, pruning
            G_star = FinetuneTraining(G_star, images_I, poses_P, is_refinement=False)
            
            # Update count
            n = count_splats(G_star)
            
    # Final sorting
    final_scores = CalGS(G_star, images_I, poses_P)
    sorted_indices = argsort_desc(final_scores)
    G_L = [G_star[i] for i in sorted_indices]
    
    return G_L
```

**Parameters:**
- $r$: Max pruning ratio. Typical values are not specified in the sources [S1].
- $d$: Target number of splats. Scene-dependent [S1].

**Edge Cases and Pitfalls:**
- Standard 3DGS hyperparameters (e.g., $\tau_p$ for cloning/splitting, densification interval) do not allow precise control over the total splat count because their interactions are complex and scene-dependent [S1].
- The mapping between hyperparameter settings and total splat count is unclear and can change across different 3D scenes [S1].
- The algorithm assumes `CalGS` can compute a consistent global significance score for all splats [S1].

## Subtopics

### [Mini-Splatting-C](10131-mini-splatting-c.md)
A sibling compression method.

### [Hybrid Compression Framework](15040-hybrid-compression-framework.md)
A sibling compression method.

## Operating characteristics
- **Time Complexity:** Not given in the sources. The iterative nature (prune then recover) implies multiple training passes, which is computationally expensive compared to single-pass pruning [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Provides precise control over the final model size (number of splats) [S1]. The top-down approach (prune then grow) is preferred over bottom-up approaches because it worked better in practice [S1].
- **Limitations:** Requires a pretrained model as a starting point [S1]. The iterative process may be slow due to repeated training steps [S1].
- **When to use:** When hardware deployment requires a strict upper bound on model size or bandwidth, and visual quality must be preserved as much as possible [S1].
- **When not to use:** When computational resources for iterative training are limited, or when an approximate size is acceptable [S1].

## Background
Prune2TargetSize was introduced in the paper "L3GS: Layered 3D Gaussian Splats for Efficient 3D Scene Delivery" published at ACM MOBICOM ’25 in November 2025 [S1]. It is part of the L3GS pipeline, which aims to create layered 3DGS models for efficient scene delivery [S1]. The method addresses the lack of precise control over model size in standard 3DGS training [S1].

## Sources
[S1] L3GS: Layered 3D Gaussian Splats for Efficient 3D Scene Delivery, https://doi.org/10.1145/3680207.3723485, fetched 2026-10-27
