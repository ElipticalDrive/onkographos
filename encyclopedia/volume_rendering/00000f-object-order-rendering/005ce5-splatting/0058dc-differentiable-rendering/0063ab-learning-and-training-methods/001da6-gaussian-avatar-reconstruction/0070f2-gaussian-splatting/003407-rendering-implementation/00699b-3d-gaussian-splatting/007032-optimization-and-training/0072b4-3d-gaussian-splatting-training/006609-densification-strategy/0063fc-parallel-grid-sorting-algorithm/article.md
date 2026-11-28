# Parallel Grid Sorting Algorithm

Also called: 3DGS densification sorting, self-organizing grid sorting

## Overview
The Parallel Grid Sorting Algorithm is a post-densification optimization step that reorders 3D Gaussian primitives into separate 2D attribute grids to maximize spatial locality and compressibility [S1]. Unlike sibling methods that focus on geometric refinement or probabilistic sampling, this method specifically targets the data layout for efficient storage and transmission by minimizing the variance of absolute differences between neighboring grid elements [S1]. It is distinct from standard rasterization sorting as it operates on the entire attribute set (position, color, opacity, scale, rotation) simultaneously to prepare the data for subsequent compression codecs [S1].

## How it works
The algorithm executes after every densification step to reorder the set of $N$ Gaussians $\mathcal{G}$ into a 2D grid structure for each attribute [S1].

**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_k$ has attributes: mean position $\mu_k \in \mathbb{R}^3$, opacity $\alpha_k \in \mathbb{R}$, scaling $s_k \in \mathbb{R}^3$, rotation $q_k \in \mathbb{R}^4$, and spherical harmonic coefficients $\mathbf{c}_k$ [Parent, S1].
*   **Output:** A set of reordered 2D grids for each attribute, such that spatially and attribute-similar Gaussians are adjacent in the grid [S1].

**Data Structures:**
The algorithm maintains five separate 2D grids, one for each attribute type: position, color/SH DC, opacity, scaling, and rotation [S1]. Each grid has dimensions $L \times L$, where $L$ is the side length sufficient to hold all Gaussians [S1]. The grid is initialized with the current Gaussian attributes [S1].

**Parameters:**
*   **Break Threshold ($\tau$):** A relative L2 reduction threshold controlling convergence, fixed to $10^{-4}$ in all training experiments [S1].
*   **Sorting Weights ($w$):** Separate multipliers for each attribute used to guide sorting priority, with position, color, and scale typically weighted higher due to their larger impact on PSNR [S1].

**Algorithm Steps:**
1.  **Initialization:** Map the $N$ Gaussians to five separate 2D grids of size $L \times L$ [S1]. Initialize the grids with the current Gaussian attributes [S1].
2.  **Iteration Loop:**
    a.  **Compute Current Cost:** Calculate the total L2 distance cost $E_{\text{curr}}$ of the current grid configuration [S1].
    b.  **Generate Permutation:** Apply a parallel sorting heuristic to generate a candidate permutation of the grid indices [S1].
    c.  **Evaluate Candidate:** Compute the cost $E_{\text{next}}$ for the permuted grid [S1].
    d.  **Update:** If $E_{\text{next}} < E_{\text{curr}}$, accept the permutation and update the grid [S1].
    e.  **Convergence Check:** Compute the relative reduction in L2 distance: $\Delta = \frac{|E_{\text{curr}} - E_{\text{next}}|}{E_{\text{curr}}}$ [S1]. If $\Delta < \tau$, stop sorting for the current configuration or proceed to the next permutation/level [S1].
3.  **Finalization:** Once convergence is reached or the maximum iterations are exceeded, the grid is finalized for compression [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def parallel_grid_sorting(gaussians, weights, threshold=1e-4, max_iterations=10000):
    """
    Reorders 3D Gaussians into separate 2D grids for compression.
    
    Args:
        gaussians: List of Gaussian objects with attributes (pos, color, opacity, scale, rot).
        weights: Dict mapping attribute names to sorting weights.
        threshold: Relative L2 reduction threshold for convergence.
        max_iterations: Maximum number of sorting iterations.
        
    Returns:
        grids: Dict mapping attribute names to 2D numpy-like arrays of Gaussians.
    """
    N = len(gaussians)
    L = math.ceil(math.sqrt(N))
    
    # Initialize five separate 2D grids for each attribute
    grids = {attr: [[None for _ in range(L)] for _ in range(L)] for attr in weights.keys()}
    for i, g in enumerate(gaussians):
        y, x = divmod(i, L)
        for attr in weights.keys():
            grids[attr][y][x] = g
            
    current_cost = compute_l2_cost(grids, weights, L)
    
    for iteration in range(max_iterations):
        # The exact parallel heuristic (e.g., row/column swaps, block moves) is not specified in the sources [G].
        # A standard implementation would apply parallelizable permutation operations here.
        candidate_grids = apply_parallel_heuristic(grids, weights, L)
        
        next_cost = compute_l2_cost(candidate_grids, weights, L)
        
        # Check for improvement
        if next_cost < current_cost:
            relative_reduction = abs(current_cost - next_cost) / max(current_cost, 1e-9)
            
            if relative_reduction < threshold:
                # Convergence reached
                break
            
            grids = candidate_grids
            current_cost = next_cost
        else:
            # No improvement, try different heuristic or break if stuck
            pass
            
    return grids

def compute_l2_cost(grids, weights, L):
    """
    Computes the sum of squared differences between neighboring cells,
    weighted by attribute importance.
    """
    total_cost = 0.0
    
    for attr, grid in grids.items():
        weight = weights[attr]
        for y in range(L):
            for x in range(L):
                current_g = grid[y][x]
                if current_g is None:
                    continue
                
                # Right neighbor
                if x + 1 < L:
                    right_g = grid[y][x + 1]
                    if right_g is not None:
                        total_cost += weight * sum((current_g[attr][c] - right_g[attr][c])**2 for c in range(len(current_g[attr])))
                
                # Down neighbor
                if y + 1 < L:
                    down_g = grid[y + 1][x]
                    if down_g is not None:
                        total_cost += weight * sum((current_g[attr][c] - down_g[attr][c])**2 for c in range(len(current_g[attr])))
                        
    return total_cost

def apply_parallel_heuristic(grids, weights, L):
    """
    Placeholder for the unspecified parallel sorting heuristic.
    The sources do not provide the exact permutation logic [G].
    """
    # Return a deep copy of grids as a fallback
    import copy
    return copy.deepcopy(grids)
```

**Edge Cases and Pitfalls:**
*   **Grid Size:** The grid side length $L$ must be sufficient to hold all Gaussians; for the largest tested scene (Garden), $N=4.37\text{M}$ requires $L=2091$ [S1].
*   **Smoothing Regularization:** Applying sorting and compression only after vanilla 3DGS training (without smoothing regularization) drastically reduces visual quality [S1]. The smoothing regularization influences the training process to make attributes more amenable to sorting [S1].
*   **Convergence:** The break threshold $\tau$ controls the trade-off between sorting quality and runtime; a lower $\tau$ yields better compression but increases runtime [S1].

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative and depends on the number of Gaussians $N$ and the grid size $L \approx \sqrt{N}$ [S1]. Each iteration involves computing the cost $O(N)$ and applying heuristics [S1]. The total runtime depends on the convergence speed, which is controlled by the break threshold [S1].
*   **Memory Complexity:** $O(N)$ to store the five $L \times L$ grids and attributes [S1].
*   **Measured Performance:**
    *   On an Nvidia RTX 4090, sorting a random $512 \times 512 \times 3$ grid takes 5.7 seconds after 8015 reorders, achieving a Variance of Absolute Differences (VAD) of 4.02 (shuffled data VAD: 3607.45) [S1].
    *   On an AMD Ryzen Threadripper PRO 5955WX CPU, the FLAS algorithm takes 131 seconds for the same grid, achieving a VAD of 3.53 [S1].
    *   The largest tested scene (Garden, 4.37M Gaussians, $L=2091$) sorts in well below one minute [S1].
*   **Strengths:** High parallelism leads to significantly reduced runtime compared to CPU-based state-of-the-art methods like FLAS, while maintaining comparable sorting quality (VAD 4.02 vs 3.53) [S1].
*   **Limitations:** Requires smoothing regularization during training to maintain visual quality; otherwise, compression degrades rendering [S1].

## Background
The Parallel Grid Sorting Algorithm was introduced in the context of compact 3D scene representation via self-organizing Gaussian grids [S1]. It addresses the need for efficient storage and transmission of 3D Gaussian Splatting models by leveraging spatial locality in attribute space [S1]. The method builds on the observation that sorting Gaussians based on their attributes (position, color, scale, etc.) can significantly improve compression ratios [S1].

## Sources
[S1] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
