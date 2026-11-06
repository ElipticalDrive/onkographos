# Hard Pruning

Also called: Post-densification pruning, Efficient Pruning Score pruning

## Overview
Hard Pruning is a density control method in 3D Gaussian Splatting that reduces model size by a constant ratio at fixed intervals after the densification stage [S1, S2]. It differs from Soft Pruning by occurring post-densification rather than during it, relying on subsequent refinement iterations to recover performance [S1, S2].

## How it works
**Inputs:**
- Current set of 3D Gaussians $\mathcal{G}$.
- Current training iteration $i$.
- Pruning interval $\Delta i = 3000$ [S1].
- Start iteration $i_{start} = 15000$ [S1].
- Pruning ratio $r = 0.3$ (30%) [S1].
- Efficient Pruning Score for each Gaussian [S2].

**Outputs:**
- Reduced set of 3D Gaussians $\mathcal{G}'$.

**Data Structures:**
- Each Gaussian $g \in \mathcal{G}$ has an associated scalar `pruning_score` derived from the Efficient Pruning Score [S2].

**Steps:**
1. Check if current iteration $i$ satisfies the pruning condition:
   $$ i \ge i_{start} \quad \text{and} \quad (i - i_{start}) \pmod{\Delta i} = 0 $$
   If false, skip pruning [S1].
2. If true, compute the Efficient Pruning Score for all Gaussians in $\mathcal{G}$ [S2].
3. Sort $\mathcal{G}$ in ascending order of pruning score (lowest score indicates least important) [S2].
4. Calculate the number of Gaussians to remove:
   $$ N_{remove} = \lfloor |\mathcal{G}| \times r \rfloor $$
5. Remove the $N_{remove}$ Gaussians with the lowest scores from $\mathcal{G}$ [S1, S2].
6. Continue training to refine the remaining Gaussians [S1].

**Parameters:**
- $\Delta i = 3000$ iterations [S1].
- $i_{start} = 15000$ iterations [S1].
- $r = 0.3$ (30%) [S1].

**Edge Cases:**
- If $|\mathcal{G}| < N_{remove}$, remove all Gaussians (though empirically unlikely given the 10.6× reduction target) [S1].
- Precision issues in score comparison are mitigated by sorting stability [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_pruning(gaussians, current_iteration):
    # Constants from [S1]
    start_iter = 15000
    interval = 3000
    ratio = 0.3
    
    # Check pruning condition
    if current_iteration < start_iter:
        return gaussians
    
    if (current_iteration - start_iter) % interval != 0:
        return gaussians
    
    # Compute scores [S2]
    scores = compute_efficient_pruning_score(gaussians)
    
    # Sort by score ascending (lowest score = prune)
    indexed_gaussians = list(enumerate(gaussians))
    indexed_gaussians.sort(key=lambda x: scores[x[0]])
    
    # Calculate number to remove
    n_to_remove = int(len(gaussians) * ratio)
    
    # Remove lowest scoring Gaussians
    # Note: Efficient Pruning Score definition is in [S2], assumed available
    gaussians = [g for idx, g in indexed_gaussians if idx >= n_to_remove]
    
    return gaussians
```

## Subtopics

### [Soft Pruning](5590-soft-pruning.md)
Pruning performed during densification to maintain Hessian approximation. Choose when integrating pruning into the densification loop.

### [PUP 3D-GS](link)
Uses post-pruning refinement similar to Hard Pruning. Choose for comparison with other post-pruning strategies.

### [LightGaussian](link)
Uses post-pruning refinement similar to Hard Pruning. Choose for comparison with other post-pruning strategies.

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ per pruning event due to sorting, where $N$ is the number of Gaussians [S2].
- **Memory Complexity:** $O(N)$ to store scores and indexed Gaussians [S2].
- **Performance:** Reduces total Gaussians by 10.6× across scenes when paired with Soft Pruning [S1]. Increases rendering and training speed by approximately 90% reduction in primitives [S2].
- **Strengths:** Simple constant-ratio reduction; effective when combined with post-pruning refinement [S1].
- **Limitations:** Requires waiting for densification to complete; performance relies on the quality of the Efficient Pruning Score [S2].
- **When to use:** After densification stage to aggressively reduce model size [S1, S2].
- **When not to use:** During densification (use Soft Pruning instead) [S2].

## Background
Hard Pruning was introduced in Speedy-Splat as part of a sparse primitive strategy for fast 3D Gaussian Splatting [S1, S2]. It builds on the observation that model performance after densification closely matches fully-trained models, allowing pruning followed by refinement [S1]. It is similar in philosophy to PUP 3D-GS and LightGaussian, which also use post-pruning refinement [S1].

## Sources
[S1] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse Primitives, https://doi.org/10.1109/cvpr52734.2025.02006, fetched 2026-10-26
[S2] Speedy-Splat: Fast 3D Gaussian Splatting with Sparse Pixels and Sparse ..., https://speedysplat.github.io/, fetched 2026-10-28
