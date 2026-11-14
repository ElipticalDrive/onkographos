# SplatQ

Also called: SplatQ: Post-training Gaussian pruning via QUBO

## Overview
SplatQ is a post-training Gaussian pruning framework that partitions Gaussians by spatial proximity and appearance similarity into size-limited blocks, smooths importance-based block retention quotas to prevent budget concentration, and constructs a quadratic approximation of joint deletion effects to jointly select Gaussians under a fixed budget [S1, unofficial]. It differs from global Top-K methods by solving a combinatorial optimization problem within each block rather than ranking all Gaussians globally by independent scores, and it differs from methods that only consider individual deletion errors by modeling pairwise interactions and joint effects via a quadratic objective [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pretrained 3D Gaussian Splatting (3DGS) model $G$ consisting of $N$ Gaussians, a target retention count $K$, and a maximum block size $M$.
*   **Output:** A pruned set of $K$ Gaussians with updated attributes after fine-tuning.

**Data Structures:**
The model is represented as a set of Gaussians $G = \{G_i\}_{i=1}^N$. Each Gaussian $G_i$ is parameterized by:
*   $\mu_i$: Center position.
*   $s_i$: Scale.
*   $r_i$: Rotation quaternion.
*   $c_i$: Zeroth-order (DC) spherical-harmonic coefficients.
*   $h_i$: Higher-order spherical-harmonic coefficients.
*   $o_i$: Opacity [S1, unofficial].

The algorithm partitions these $N$ Gaussians into blocks $\{C_b\}$. Each block $C_b$ contains at most $M$ members. For each block, a response matrix $R_b$ is computed to estimate deletion effects.

**Steps:**

1.  **Partitioning:** Divide the $N$ Gaussians into several size-limited blocks $\{C_b\}$ of at most $M$ members each, based on spatial proximity and appearance similarity [S1, unofficial]. This is typically achieved using a 3D grid and Morton ordering to efficiently group related Gaussians without quadratic pairwise comparison costs [S1, unofficial].
2.  **Opacity Response Modeling:** For each block $C_b$, compute a response matrix $R_b$ using sampled pixels from multiple views. This matrix estimates individual deletion effects and pairwise interactions by observing changes in rendered pixel colors relative to changes in Gaussian opacity [S1, unofficial].
3.  **Budget Allocation:** Assign a retention quota $K_b$ to every block such that $\sum_b K_b = K$. The initial quota is derived from importance estimates, and the distribution is adjusted (smoothed) to reduce the risk of concentrating the budget in a few blocks, thereby preventing spatial quality imbalance [S1, unofficial].
4.  **Quadratic Selection:** For each block, formulate a Quadratic Unconstrained Binary Optimization (QUBO) objective. This objective incorporates individual responses, pairwise interactions, and quota penalties. Solve the QUBO using Simulated Annealing (SA) or a Coherent Ising Machine (CIM) to determine which $K_b$ block members to retain [S1, unofficial].
5.  **Fine-tuning:** Fine-tune the attributes ($\mu, s, r, c, h, o$) of the retained Gaussians without changing their count to recover rendering quality [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial import cKDTree # Standard library for KD-tree operations

def splatq_pruning(G, N, K, M, num_views=10, num_pixels_per_view=1000):
    """
    G: List of N Gaussian objects, each with attributes mu, s, r, c, h, o.
    N: Total number of Gaussians.
    K: Target number of Gaussians to retain.
    M: Maximum size of each block.
    num_views: Number of random camera views to sample for response modeling.
    num_pixels_per_view: Number of random pixels to sample per view.
    
    Returns:
    pruned_G: List of K Gaussian objects.
    """
    
    # Stage 1: Partitioning
    # Group Gaussians by spatial proximity and appearance similarity.
    # Implementation detail: Use a 3D grid or KD-tree to find neighbors,
    # then cluster them into blocks of size <= M.
    blocks = partition_gaussians(G, M)
    
    # Stage 2: Opacity Response Modeling
    # For each block, estimate how deleting Gaussians affects rendered pixels.
    response_matrices = {}
    for b_idx, block in enumerate(blocks):
        R_b = compute_response_matrix(block, num_views, num_pixels_per_view)
        response_matrices[b_idx] = R_b
        
    # Stage 3: Budget Allocation
    # Assign retention quota K_b to each block such that sum(K_b) = K.
    # Quotas are smoothed to prevent concentration in high-score regions.
    quotas = allocate_budget(response_matrices, K)
    
    # Stage 4: Quadratic Selection
    # Solve QUBO for each block to select exactly K_b Gaussians.
    selected_indices_global = []
    global_offset = 0
    
    for b_idx, block in enumerate(blocks):
        R_b = response_matrices[b_idx]
        K_b = quotas[b_idx]
        
        # Define QUBO coefficients based on R_b
        # Linear term (bias) and Quadratic term (interaction)
        # The specific formulation minimizes rendering error subject to retention count.
        # Solve using Simulated Annealing or CIM.
        # Returns a binary mask of length len(block), with sum == K_b.
        mask = solve_qubo(R_b, K_b)
        
        # Map local block indices to global indices
        for local_idx, retained in enumerate(mask):
            if retained:
                selected_indices_global.append(global_offset + local_idx)
                
        global_offset += len(block)
        
    # Stage 5: Fine-tuning
    # Retain only the selected Gaussians and fine-tune their attributes.
    pruned_G = [G[i] for i in selected_indices_global]
    pruned_G = fine_tune_attributes(pruned_G)
    
    return pruned_G

def partition_gaussians(G, M):
    """
    Partitions Gaussians into blocks based on spatial proximity and appearance.
    Uses Morton ordering or grid-based clustering.
    """
    # [G] Implementation note: 
    # 1. Compute a feature vector for each Gaussian (e.g., center + DC color).
    # 2. Build a KD-tree or use a 3D grid hash.
    # 3. Cluster neighbors into groups of size <= M.
    # The exact clustering algorithm (e.g., DBSCAN, hierarchical) is not specified 
    # in the source, but the goal is to group spatially/appearance-similar Gaussians.
    # For the purpose of this implementation, we assume a placeholder function 
    # that returns a list of lists (blocks).
    blocks = []
    # Placeholder for the actual partitioning logic described as "3D grid and Morton ordering"
    # Since the source does not provide the exact clustering code, we mark this as [G].
    # In a real implementation, one would:
    # - Quantize centers to grid cells.
    # - Sort cells by Morton code.
    # - Assign Gaussians to blocks ensuring size <= M and similarity.
    raise NotImplementedError("Exact partitioning algorithm (Morton ordering details) not provided in source.")

def compute_response_matrix(block, num_views, num_pixels_per_view):
    """
    Computes the response matrix R_b for a block.
    R_b captures individual deletion effects and pairwise interactions.
    """
    # [G] Implementation note:
    # 1. Render the scene with the current model.
    # 2. Sample random pixels from num_views random camera poses.
    # 3. For each Gaussian in the block, perturb its opacity (or delete it) and measure
    #    the change in pixel color (L2 loss or similar).
    # 4. For pairs of Gaussians, perturb both and measure joint effect.
    # 5. Construct matrix R_b where entries represent sensitivity/interaction strength.
    # The source states R_b is used to estimate "individual deletion effects and pairwise interactions".
    raise NotImplementedError("Exact response matrix computation formula not provided in source.")

def allocate_budget(response_matrices, K):
    """
    Assigns retention quotas K_b to blocks such that sum(K_b) = K.
    Smooths importance-based quotas to prevent concentration.
    """
    # [G] Implementation note:
    # 1. Estimate importance for each block (e.g., sum of individual scores in R_b).
    # 2. Assign initial K_b proportional to importance.
    # 3. Apply smoothing to the distribution of K_b to ensure no block gets 0 or 
    #    disproportionately high quota, preventing spatial imbalance.
    # The source mentions "smooths importance-based block retention quotas".
    raise NotImplementedError("Exact smoothing algorithm for budget allocation not provided in source.")

def solve_qubo(R_b, K_b):
    """
    Solves the QUBO problem for a block.
    Minimizes quadratic objective subject to sum(x) = K_b.
    """
    # [G] Implementation note:
    # The QUBO objective is: min x^T Q x + c^T x
    # where x is a binary vector of length len(block).
    # Q contains pairwise interaction terms from R_b.
    # c contains individual deletion costs from R_b.
    # A penalty term enforces sum(x) = K_b.
    # Solved via Simulated Annealing (SA) or Coherent Ising Machine (CIM).
    # Here we use a placeholder for SA.
    n = len(R_b)
    x = np.random.randint(0, 2, size=n)
    # Ensure initial solution satisfies constraint if possible, or use penalty method
    # Standard SA loop:
    #   propose flip
    #   compute delta_energy
    #   accept/reject based on temperature
    # Return binary mask with sum == K_b
    raise NotImplementedError("Exact QUBO formulation and SA parameters not provided in source.")

def fine_tune_attributes(pruned_G):
    """
    Fine-tunes the attributes of the retained Gaussians.
    """
    # [G] Standard 3DGS fine-tuning:
    # Render the pruned set and optimize parameters (mu, s, r, c, h, o) using SGD
    # to minimize rendering loss against ground truth images.
    # The count of Gaussians remains K.
    raise NotImplementedError("Fine-tuning hyperparameters not provided in source.")
```

**Parameters:**
*   $N$: The total number of Gaussians in the pretrained 3DGS model [S1, unofficial].
*   $K$: The exact number of Gaussians to retain after pruning [S1, unofficial].
*   $M$: The maximum size (number of members) of each block $C_b$ [S1, unofficial].
*   $K_b$: The retention quota assigned to block $b$, representing the number of Gaussians to retain within that block [S1, unofficial].
*   $R_b$: The response matrix for block $b$, used to estimate individual deletion effects and pairwise interactions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Spatial Quality Imbalance:** Global ranking methods can lead to situations where the limited budget concentrates in regions with consistently higher scores, leaving other regions with insufficient representational capacity [S1, unofficial]. SplatQ mitigates this via blockwise budget allocation [S1, unofficial].
*   **Joint Deletion Effects:** Independent scoring fails to capture joint rendering effects because multiple Gaussians may jointly support the same pixel or make overlapping contributions. Degradation from removing multiple Gaussians is not strictly additive [S1, unofficial]. SplatQ addresses this by modeling pairwise interactions [S1, unofficial].
*   **Scalability:** Formulating the optimization over the entire model requires $N$ binary variables and an $N \times N$ matrix, which is impractical for millions of Gaussians. Partitioning into blocks of size $M$ reduces this complexity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The partitioning step uses Morton ordering and grid-based methods to avoid quadratic pairwise comparisons, making it more efficient than all-pairs similarity search [S1, unofficial]. The QUBO solving step is NP-hard, but is restricted to blocks of size $M$, making it tractable. The overall complexity depends on the QUBO solver (SA or CIM) and the block size $M$ [S1, unofficial].
*   **Memory Complexity:** The response matrix $R_b$ for each block requires memory proportional to $M^2$. Since $M \ll N$, this is significantly less than storing an $N \times N$ matrix [S1, unofficial].
*   **Strengths:** Ensures balanced reconstruction quality across a scene by controlling regional budget allocation. Captures joint deletion effects through quadratic approximation, avoiding redundancy from overlapping Gaussians [S1, unofficial].
*   **Limitations:** Requires multiple views and pixel sampling for response modeling, adding overhead. The QUBO solution is approximate (unless using exact solvers for small $M$). Fine-tuning is required post-pruning to recover quality [S1, unofficial].
*   **When to use:** When balanced quality across the entire scene is critical and the scene contains significant Gaussian overlap/redundancy. When global Top-K pruning results in spatial artifacts.
*   **When not to use:** When computational resources for QUBO solving are extremely limited and $M$ cannot be kept small. When real-time pruning is required without fine-tuning.

## Background
SplatQ was proposed by Hongbin Ye, Haixu Liu, Shenwei Kang, Wenxiu Wei, Tinglan Wang, Hongdong Zhu, Fan Tang, Qi Gao, Yin Ma, Kai Wen, Hai Wei, Nuo Qian, Wei-Shi Zheng, Shaobo Chen, Fa-Ting Hong, and Jingke Meng. It was presented as an arXiv preprint (2610.13503) in October 2026, under review for ICLR 2027 [S1, unofficial]. The method addresses limitations in existing 3D Gaussian Splatting pruning methods, which primarily rely on global Top-K selection based on independent importance scores, lacking regional budget control and ignoring joint deletion effects [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
