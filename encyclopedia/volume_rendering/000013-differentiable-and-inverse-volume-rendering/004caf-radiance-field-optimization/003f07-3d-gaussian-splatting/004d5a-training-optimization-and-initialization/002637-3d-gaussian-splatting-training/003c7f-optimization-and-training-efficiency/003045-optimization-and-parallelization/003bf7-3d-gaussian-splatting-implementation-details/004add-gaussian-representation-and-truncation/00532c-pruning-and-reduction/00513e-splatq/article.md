# SplatQ

Also called: SplatQ: Post-training Gaussian pruning via QUBO

## Overview
SplatQ is a post-training Gaussian pruning framework that partitions Gaussians by spatial proximity and appearance similarity into size-limited blocks, smooths importance-based block retention quotas to prevent budget concentration, and constructs a quadratic approximation of joint deletion effects to jointly select Gaussians under a fixed budget [S1, unofficial]. It differs from global Top-K methods by decomposing the selection into subproblems of specific size and solving a combinatorial optimization problem within each block, rather than ranking all Gaussians globally by independent scores [S1, unofficial].

## How it works
**Inputs and outputs:**
*   **Input:** A pretrained 3D Gaussian Splatting (3DGS) model $G$ containing $N$ Gaussians, and a target retention count $K$ [S1, unofficial].
*   **Output:** A pruned set of $K$ Gaussians with updated attributes [S1, unofficial].

**Data structures:**
The model is represented as $G = \{G_i = (\mu_i, s_i, r_i, c_i, h_i, o_i)\}_{i=1}^N$, where:
*   $\mu_i$: Center position [S1, unofficial].
*   $s_i$: Scale [S1, unofficial].
*   $r_i$: Rotation quaternion [S1, unofficial].
*   $c_i$: Zeroth-order (DC) spherical-harmonic coefficients [S1, unofficial].
*   $h_i$: Higher-order spherical-harmonic coefficients [S1, unofficial].
*   $o_i$: Opacity [S1, unofficial].

Gaussians are partitioned into blocks $\{C_b\}$, where each block $C_b$ contains at most $M$ members [S1, unofficial]. For each block, a response matrix $R_b$ is computed [S1, unofficial].

**Parameters:**
*   $N$: Total number of Gaussians in the pretrained model [S1, unofficial].
*   $K$: Exact number of Gaussians to retain after pruning [S1, unofficial].
*   $M$: Maximum size (number of members) of each block $C_b$ [S1, unofficial].
*   $K_b$: Retention quota assigned to block $b$, representing the number of Gaussians to retain within that block [S1, unofficial].
*   $R_b$: Response matrix for block $b$ [S1, unofficial].

**Steps:**

1.  **Partitioning:** Divide the $N$ Gaussians into several size-limited blocks $\{C_b\}$ of at most $M$ members each, based on spatial proximity and appearance similarity [S1, unofficial]. This is achieved using a 3D grid and Morton ordering to efficiently group related Gaussians without quadratic pairwise comparison costs [S1, unofficial].
2.  **Opacity Response Modeling:** For each block $C_b$, compute a response matrix $R_b$ using sampled pixels from multiple views to estimate individual deletion effects and pairwise interactions [S1, unofficial]. The matrix approximates joint deletion effects using the responses of representative pixel colors to changes in Gaussian opacity [S1, unofficial].
3.  **Budget Allocation:** Assign a retention quota $K_b$ to every block such that $\sum_b K_b = K$ [S1, unofficial]. This is done by deriving an initial quota from importance estimates and adjusting the distribution to reduce the risk of concentrating the budget in a few blocks [S1, unofficial].
4.  **Quadratic Selection:** For each block, formulate a Quadratic Unconstrained Binary Optimization (QUBO) objective incorporating individual responses, pairwise interactions, and quota penalties [S1, unofficial]. Solve the QUBO using Simulated Annealing (SA) or a Coherent Ising Machine (CIM) to determine which $K_b$ block members to retain [S1, unofficial].
5.  **Fine-tuning:** Fine-tune the attributes of the retained Gaussians without changing their count to recover rendering quality [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatq_prune(G, N, K, M, views):
    """
    G: List of N Gaussian objects, each with attributes (mu, s, r, c, h, o).
    N: Total number of Gaussians.
    K: Target number of Gaussians to retain.
    M: Maximum block size.
    views: List of camera poses for sampling pixels.
    """
    
    # Stage 1: Partitioning
    # Group Gaussians into blocks based on spatial proximity and appearance similarity.
    # Uses 3D grid and Morton ordering for efficiency.
    blocks = partition_gaussians(G, M)
    
    # Stage 2: Opacity Response Modeling
    # Compute response matrix Rb for each block to estimate deletion effects.
    response_matrices = {}
    for block in blocks:
        # Sample pixels from multiple views to estimate individual and pairwise effects.
        # The exact formula for Rb is derived from pixel color changes w.r.t opacity.
        Rb = compute_response_matrix(block, views)
        response_matrices[block] = Rb
        
    # Stage 3: Budget Allocation
    # Assign retention quota Kb to each block such that sum(Kb) = K.
    # Smooth importance-based quotas to prevent concentration in high-score regions.
    quotas = allocate_budget(blocks, response_matrices, K)
    
    # Stage 4: Quadratic Selection
    retained_gaussians = []
    for block, Rb in response_matrices.items():
        Kb = quotas[block]
        
        # Formulate QUBO objective:
        # Minimize: sum(w_i * x_i) + sum(w_ij * x_i * x_j) + penalty(x)
        # where x_i is binary retention variable (1=retain, 0=delete).
        # w_i: individual response (from Rb).
        # w_ij: pairwise interaction (from Rb).
        # penalty: enforces sum(x_i) == Kb.
        
        # Solve QUBO using Simulated Annealing (SA) or Coherent Ising Machine (CIM).
        # Returns binary selection vector for the block.
        selection = solve_qubo(Rb, Kb)
        
        # Select Gaussians based on selection vector
        retained_gaussians.extend([G[i] for i, keep in enumerate(selection) if keep])
        
    # Stage 5: Fine-tuning
    # Fine-tune attributes of retained Gaussians without changing count.
    final_model = fine_tune(retained_gaussians, views)
    
    return final_model

def partition_gaussians(G, M):
    """
    Partitions Gaussians into blocks of size <= M using spatial and appearance similarity.
    Implementation detail: Uses 3D grid hashing and Morton Z-order curve sorting.
    """
    # Assign each Gaussian to a 3D grid cell based on mu_i.
    grid_cells = {}
    for i, g in enumerate(G):
        cell_id = get_grid_cell(g.mu)
        if cell_id not in grid_cells:
            grid_cells[cell_id] = []
        grid_cells[cell_id].append(i)
        
    # Sort cells using Morton order to preserve spatial locality.
    sorted_cells = sort_cells_by_morton_order(grid_cells.keys())
    
    blocks = []
    current_block = []
    
    for cell_id in sorted_cells:
        indices = grid_cells[cell_id]
        for idx in indices:
            current_block.append(idx)
            if len(current_block) >= M:
                blocks.append(current_block)
                current_block = []
        # If block is not full but next cell is far, could split, but Morton order
        # generally keeps nearby items together.
        
    if current_block:
        blocks.append(current_block)
        
    return blocks

def compute_response_matrix(block_indices, views):
    """
    Computes the response matrix Rb for a block of Gaussians.
    Rb[i, j] represents the interaction between Gaussian i and Gaussian j.
    Diagonal elements represent individual deletion impact.
    """
    # Sample a set of representative pixels from views.
    # For each pixel, render with current Gaussians.
    # Then, perturb opacity of Gaussian i and measure color change (individual effect).
    # Perturb opacity of Gaussians i and j together and measure change (pairwise effect).
    # The quadratic approximation uses these changes to estimate joint deletion effects.
    # Note: Exact mathematical formulation of Rb entries is not provided in the abstract/method summary.
    # [G] Typically, Rb[i,i] is the squared error change if Gaussian i is removed alone.
    # Rb[i,j] is the interaction term, often derived from the difference between joint removal error
    # and sum of individual errors.
    pass

def allocate_budget(blocks, response_matrices, K):
    """
    Assigns retention quota Kb to each block.
    """
    # Calculate initial importance score for each block (e.g., sum of diagonal elements in Rb).
    scores = [sum(response_matrices[b][i][i] for i in range(len(b))) for b in blocks]
    
    # Smooth scores to prevent budget concentration in high-score regions.
    # Normalize scores to get initial ratios.
    total_score = sum(scores)
    quotas = {}
    for i, b in enumerate(blocks):
        # Initial quota proportional to score
        q = scores[i] / total_score * K
        # Apply smoothing factor to reduce variance between blocks
        # [G] Specific smoothing formula not given; typically involves averaging with uniform distribution.
        quotas[b] = q
        
    # Adjust quotas to ensure sum(Kb) = K and Kb are integers.
    # [G] Rounding and redistribution logic required.
    return quotas

def solve_qubo(Rb, Kb):
    """
    Solves the QUBO problem for a block.
    """
    # Construct QUBO matrix Q from Rb and quota penalty.
    # Q = Rb + lambda * I_quota
    # Solve using Simulated Annealing (SA) or Coherent Ising Machine (CIM).
    # Return binary vector x where sum(x) = Kb.
    pass

def fine_tune(retained_gaussians, views):
    """
    Fine-tunes attributes of retained Gaussians.
    """
    # Run standard 3DGS optimization steps (position, scale, rotation, SH coefficients)
    # for a few iterations, keeping the set of Gaussians fixed.
    pass
```

**Edge cases and pitfalls:**
*   **Spatial Quality Imbalance:** Global ranking can lead to spatial quality imbalance where the limited budget concentrates in regions with consistently higher scores, leaving other regions with insufficient representational capacity [S1, unofficial]. SplatQ addresses this via blockwise retention ratio allocation [S1, unofficial].
*   **Joint Effects:** Independent scoring fails to capture joint rendering effects because multiple Gaussians may jointly support the same pixel or make overlapping contributions, meaning degradation from removing multiple Gaussians is not strictly additive [S1, unofficial]. SplatQ models pairwise interactions via a quadratic objective [S1, unofficial].
*   **Complexity:** Formulating the optimization problem over the entire model requires $N$ binary variables and an $N \times N$ matrix, which is impractical for millions of Gaussians [S1, unofficial]. Partitioning into blocks of size $M$ reduces this to manageable subproblems [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The partitioning step uses 3D grid and Morton ordering to avoid quadratic pairwise comparisons [S1, unofficial]. The QUBO solving step is NP-hard, but is solved per block of size $M$, making it feasible where global QUBO would not be [S1, unofficial].
*   **Memory Complexity:** Requires storing the response matrix $R_b$ for each block, which is $O(M^2)$ per block, significantly less than $O(N^2)$ for the global case [S1, unofficial].
*   **Strengths:** Ensures balanced reconstruction quality across a scene by coordinating regional budget allocation [S1, unofficial]. Captures joint deletion effects through quadratic approximation [S1, unofficial].
*   **Limitations:** Requires solving a combinatorial optimization problem (QUBO) for each block, which may be slower than simple thresholding methods [S1, unofficial]. Relies on accurate estimation of the response matrix [S1, unofficial].
*   **When to use:** When balanced quality across different regions of a scene is critical and the computational cost of QUBO solving is acceptable [S1, unofficial].
*   **When not to use:** When real-time pruning is required or when scene regions have highly uniform importance [S1, unofficial].

## Background
SplatQ was proposed by Hongbin Ye, Haixu Liu, Shenwei Kang, Wenxiu Wei, Tinglan Wang, Hongdong Zhu, Fan Tang, Qi Gao, Yin Ma, Kai Wen, Hai Wei, Nuo Qian, Wei-Shi Zheng, Shaobo Chen, Fa-Ting Hong, and Jingke Meng [S1, unofficial]. It was presented as a preprint on arXiv in October 2026 [S1, unofficial]. The method addresses limitations of existing 3D Gaussian Splatting pruning methods that rely on global Top-K selection based on independent importance scores, which lack regional budget control and ignore joint deletion effects [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
