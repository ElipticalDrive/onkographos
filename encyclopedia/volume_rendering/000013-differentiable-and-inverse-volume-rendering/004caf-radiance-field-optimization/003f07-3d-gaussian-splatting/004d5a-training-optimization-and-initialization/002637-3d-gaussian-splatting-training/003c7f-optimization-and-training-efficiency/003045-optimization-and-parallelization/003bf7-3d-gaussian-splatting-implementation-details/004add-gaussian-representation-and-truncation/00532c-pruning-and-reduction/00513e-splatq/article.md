# SplatQ

Also called: SplatQ: Post-training Gaussian pruning via QUBO

## Overview
SplatQ is a post-training pruning framework for 3D Gaussian Splatting (3DGS) that addresses the spatial quality imbalance and loss of joint rendering effects inherent in global Top-K selection methods [S1, unofficial]. It partitions Gaussians into size-limited blocks based on spatial and appearance similarity, allocates retention quotas to these blocks to ensure balanced budget distribution, and solves a Quadratic Unconstrained Binary Optimization (QUBO) problem within each block to jointly select Gaussians that minimize reconstruction error under a fixed global count [S1, unofficial].

## How it works
**Inputs:** A pretrained 3DGS model with $N$ Gaussians $\mathcal{G} = \{G_i\}_{i=1}^N$ and a target retention count $K$.
**Outputs:** A pruned set of $K$ Gaussians.

**Data Structures:**
*   **Gaussian $G_i$:** Defined by center $\mu_i$, scale $s_i$, rotation quaternion $r_i$, zeroth-order SH coefficients $c_i$, higher-order SH coefficients $h_i$, and opacity $o_i$ [S1, unofficial].
*   **Block $C_b$:** A subset of Gaussians with size $|C_b| \le M$, where $M$ is the maximum block size [S1, unofficial].
*   **Response Matrix $R_b$:** A matrix for block $C_b$ estimating individual deletion effects and pairwise interactions, derived from sampled pixel responses [S1, unofficial].
*   **Retention Quota $K_b$:** An integer representing the number of Gaussians to retain in block $b$, such that $\sum_b K_b = K$ [S1, unofficial].

**Algorithm Steps:**

1.  **Partitioning:** Divide the $N$ Gaussians into blocks $\{C_b\}$ of at most $M$ members each. Partitioning is guided by spatial proximity and appearance similarity to ensure potentially redundant Gaussians are evaluated together. This is implemented efficiently using a 3D grid and Morton ordering to avoid quadratic pairwise comparisons [S1, unofficial].
2.  **Opacity Response Modeling:** For each block $C_b$, compute the response matrix $R_b$. This involves sampling pixels from multiple views to estimate how the rendered image changes when Gaussians are deleted. The matrix captures individual deletion effects and pairwise interactions (joint effects) [S1, unofficial].
3.  **Budget Allocation:** Assign a retention quota $K_b$ to every block $b$ such that $\sum_b K_b = K$. The initial quota is derived from importance estimates (derived from $R_b$), and the distribution is smoothed to prevent the budget from concentrating in regions with consistently high scores, which would leave other regions with insufficient representational capacity [S1, unofficial].
4.  **Quadratic Selection:** For each block $C_b$, formulate a QUBO objective function. The objective incorporates individual responses, pairwise interactions, and a penalty for deviating from the retention quota $K_b$. The binary variable $x_i \in \{0, 1\}$ indicates if Gaussian $i$ is retained ($1$) or deleted ($0$). The problem is solved using Simulated Annealing (SA) or a Coherent Ising Machine (CIM) to determine the optimal set of $K_b$ members to retain [S1, unofficial].
5.  **Fine-tuning:** Fine-tune the attributes ($\mu, s, r, c, h, o$) of the retained Gaussians without changing their count to recover rendering quality lost during pruning [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatq_prune(G, N, K, M, num_views=10, num_pixels_per_view=100):
    """
    G: List of N Gaussian objects, each with attributes mu, s, r, c, h, o.
    N: Total number of Gaussians.
    K: Target number of Gaussians to retain.
    M: Maximum size of each block.
    """
    
    # Stage 1: Partitioning
    # Use 3D grid and Morton ordering to group Gaussians by spatial proximity
    # and appearance similarity.
    blocks = partition_by_morton_ordering(G, M)
    
    # Stage 2: Opacity Response Modeling
    # Compute response matrix Rb for each block to estimate joint deletion effects.
    response_matrices = {}
    for b, block in enumerate(blocks):
        # Sample pixels from multiple views
        sampled_pixels = sample_pixels_from_views(num_views, num_pixels_per_view)
        
        # Estimate individual and pairwise effects on pixel colors
        # Rb[i, j] represents the interaction between Gaussian i and j in block b
        Rb = compute_response_matrix(block, sampled_pixels)
        response_matrices[b] = Rb
        
    # Stage 3: Budget Allocation
    # Assign retention quota Kb to each block to balance spatial quality.
    quotas = allocate_budget(response_matrices, blocks, K)
    
    # Stage 4: Quadratic Selection
    pruned_gaussians = []
    for b, block in enumerate(blocks):
        Kb = quotas[b]
        Rb = response_matrices[b]
        
        # Formulate QUBO objective:
        # Minimize: sum_i (individual_cost_i * x_i) + sum_{i<j} (interaction_ij * x_i * x_j)
        # Subject to: sum_i x_i = Kb
        # x_i in {0, 1}
        
        # Solve using Simulated Annealing or CIM
        # Returns binary selection vector for this block
        selection = solve_qubo(Rb, Kb)
        
        # Select Gaussians based on selection vector
        retained_block = [G[i] for i, keep in enumerate(selection) if keep]
        pruned_gaussians.extend(retained_block)
        
    # Stage 5: Fine-tuning
    # Recover quality by optimizing attributes of retained Gaussians
    final_model = fine_tune(pruned_gaussians, original_training_data)
    
    return final_model

def partition_by_morton_ordering(G, M):
    """
    Groups Gaussians into blocks of size <= M based on 3D grid and Morton ordering.
    """
    # 1. Assign each Gaussian to a 3D grid cell based on mu
    grid_cells = assign_to_grid(G)
    
    # 2. Sort cells and Gaussians within cells using Morton order (Z-order curve)
    # This preserves spatial locality in the linear sequence
    sorted_gaussians = sort_by_morton_order(G)
    
    # 3. Split into chunks of size M
    blocks = []
    for i in range(0, len(sorted_gaussians), M):
        blocks.append(sorted_gaussians[i:i+M])
        
    return blocks

def compute_response_matrix(block, sampled_pixels):
    """
    Estimates individual and pairwise deletion effects.
    """
    n = len(block)
    Rb = np.zeros((n, n))
    
    for pixel in sampled_pixels:
        # Render image with all Gaussians in block
        original_render = render_with_gaussians(block, pixel)
        
        for i in range(n):
            # Render with Gaussian i removed
            render_i_removed = render_with_gaussians(block_without_i(block, i), pixel)
            
            # Individual effect (diagonal)
            Rb[i, i] += loss(original_render, render_i_removed)
            
            for j in range(i + 1, n):
                # Render with both i and j removed
                render_ij_removed = render_with_gaussians(block_without_i_and_j(block, i, j), pixel)
                
                # Pairwise interaction (off-diagonal)
                # Interaction is the difference between joint removal and sum of individual removals
                # This captures non-additive effects
                interaction = loss(original_render, render_ij_removed) - \
                              (loss(original_render, render_i_removed) + loss(original_render, render_j_removed))
                Rb[i, j] += interaction
                Rb[j, i] += interaction
                
    return Rb

def allocate_budget(response_matrices, blocks, K):
    """
    Assigns quotas Kb to blocks to prevent budget concentration.
    """
    # Calculate initial importance score for each block (e.g., sum of diagonal elements in Rb)
    block_importances = [np.trace(Rb) for Rb in response_matrices.values()]
    
    # Smooth quotas to ensure no block gets zero or disproportionately high quota
    # Simple smoothing: distribute K proportionally but with a minimum floor
    total_importance = sum(block_importances)
    quotas = []
    for imp in block_importances:
        # Proportional allocation
        q = K * (imp / total_importance)
        # Apply smoothing (e.g., ensure at least 10% of average block size or similar heuristic)
        # Exact smoothing formula not specified in sources, but goal is balance [S1, unofficial]
        quotas.append(max(1, int(round(q))))
        
    # Adjust quotas to sum exactly to K
    quotas = adjust_quotas_to_sum(quotas, K)
    return quotas

def solve_qubo(Rb, Kb):
    """
    Solves the QUBO problem for a block.
    """
    # QUBO formulation:
    # Minimize x^T Q x
    # Where Q is derived from Rb and Kb penalty
    # Penalty term: lambda * (sum(x) - Kb)^2
    
    # Construct Q matrix
    n = len(Rb)
    Q = Rb.copy()
    # Add penalty for quota violation
    # This is a simplified representation; actual QUBO construction involves expanding the penalty
    # into quadratic terms for binary variables
    penalty_lambda = 1000.0 # Heuristic weight
    for i in range(n):
        Q[i, i] += penalty_lambda
    for i in range(n):
        for j in range(n):
            if i != j:
                Q[i, j] += penalty_lambda # Cross terms from (sum x - K)^2
                
    # Solve using Simulated Annealing
    # Initialize random binary vector x
    x = np.random.randint(0, 2, n)
    
    # Simulated Annealing loop
    best_x = x
    best_energy = energy(x, Q)
    
    for t in range(max_iterations):
        # Perturb x
        x_new = x.copy()
        idx = np.random.randint(n)
        x_new[idx] = 1 - x_new[idx]
        
        # Check quota constraint (or use penalty in energy)
        # If using hard constraint, reject moves that violate Kb
        if np.sum(x_new) != Kb:
            continue
            
        new_energy = energy(x_new, Q)
        
        # Acceptance criterion
        if new_energy < best_energy or np.random.rand() < acceptance_probability(best_energy, new_energy, t):
            best_x = x_new
            best_energy = new_energy
            x = x_new
            
    return best_x

def energy(x, Q):
    return x.T @ Q @ x

def fine_tune(pruned_gaussians, training_data):
    """
    Optimizes attributes of retained Gaussians.
    """
    # Standard 3DGS training loop, but only optimizing attributes of pruned_gaussians
    # No new Gaussians are added or removed
    model = Model(pruned_gaussians)
    optimizer = Adam(model.parameters())
    
    for epoch in range(fine_tune_epochs):
        for batch in training_data:
            loss = render_and_compute_loss(model, batch)
            loss.backward()
            optimizer.step()
            
    return model
```

**Parameters:**
*   $N$: Total number of Gaussians in the pretrained model [S1, unofficial].
*   $K$: Exact number of Gaussians to retain [S1, unofficial].
*   $M$: Maximum size of each block [S1, unofficial].
*   $K_b$: Retention quota for block $b$ [S1, unofficial].
*   $R_b$: Response matrix for block $b$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Spatial Quality Imbalance:** Global ranking methods concentrate the budget in regions with high scores, leaving other regions with insufficient capacity [S1, unofficial]. SplatQ mitigates this via blockwise quota allocation.
*   **Non-Additive Degradation:** Removing multiple overlapping Gaussians does not have an additive effect on rendering error. Independent scoring fails to capture this, potentially retaining redundant Gaussians [S1, unofficial]. SplatQ models this via pairwise interactions in $R_b$.
*   **Computational Cost:** Solving QUBO is NP-hard. The block size $M$ must be small enough for SA/CIM to be feasible, but large enough to capture interactions.

## Operating characteristics
*   **Time Complexity:** Partitioning is $O(N \log N)$ or $O(N)$ with grid/Morton ordering. Response matrix computation is $O(B \cdot M^2 \cdot P \cdot V)$, where $B$ is number of blocks, $P$ is pixels per view, $V$ is views. QUBO solving is exponential in $M$ but constant for fixed $M$. Fine-tuning is $O(N_{retained} \cdot E \cdot I)$, where $E$ is epochs, $I$ is iterations.
*   **Memory Complexity:** $O(N)$ for Gaussians, $O(B \cdot M^2)$ for response matrices.
*   **Strengths:** Ensures balanced reconstruction quality across the scene; captures joint deletion effects; post-training (no additional learning required for selection).
*   **Limitations:** Requires solving QUBO, which can be slow for large $M$; sensitive to the choice of $M$ and smoothing parameters in budget allocation.
*   **When to use:** When balanced spatial quality is critical and the scene has varying density; when post-training pruning is preferred over training-time pruning.
*   **When not to use:** When real-time pruning is required; when $N$ is very large and $M$ cannot be kept small.

## Background
SplatQ was proposed by Hongbin Ye et al. in 2026 [S1, unofficial]. It addresses limitations of existing 3DGS pruning methods that rely on global Top-K selection based on independent importance scores, which fail to ensure balanced reconstruction quality and ignore joint deletion effects [S1, unofficial]. The method uses QUBO optimization, a technique common in combinatorial optimization, applied to Gaussian pruning for the first time in this context.

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
