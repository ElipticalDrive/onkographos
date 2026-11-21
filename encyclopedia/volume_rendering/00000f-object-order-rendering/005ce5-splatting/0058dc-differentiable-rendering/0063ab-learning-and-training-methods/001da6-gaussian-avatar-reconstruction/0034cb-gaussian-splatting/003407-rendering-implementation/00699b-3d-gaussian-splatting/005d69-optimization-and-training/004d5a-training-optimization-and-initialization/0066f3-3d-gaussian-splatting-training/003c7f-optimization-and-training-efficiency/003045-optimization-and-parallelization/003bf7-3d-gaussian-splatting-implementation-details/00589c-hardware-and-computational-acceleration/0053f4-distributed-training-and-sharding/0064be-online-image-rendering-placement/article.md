# Online Image Rendering Placement

Also called: Image patch placement, Online placement of image rendering

## Overview
Online Image Rendering Placement is a per-iteration optimization step in distributed 3D Gaussian Splatting training that assigns image patches to GPUs to minimize inter-node communication and balance computational load [S1, unofficial]. It differs from static graph partitioning by dynamically solving for the optimal data placement for each small training batch using a fast heuristic solver, rather than relying on offline pre-computed assignments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** The current batch of $B$ image patches (derived from $B$ training images), the global set of 3D Gaussians $\mathcal{G}$ partitioned across $N$ GPUs, and the local shard $\mathcal{G}^{(m)}$ on each GPU $m$.
*   **Outputs:** A placement vector $W$, where $W_j$ is the GPU ID assigned to render image patch $j$.

**Data Structures:**
*   **Patch-GPU Access Pattern Matrix ($A$):** A $B \times N$ integer matrix where $A_{j,k}$ represents the number of 3D Gaussians required from GPU $k$ to render image patch $j$. This is determined via frustum culling.
*   **Placement Vector ($W$):** An array of length $B$ storing GPU indices.

**Parameters:**
*   $B$: Batch size of training images. Typical value: 16–64 [S1, unofficial].
*   $N$: Number of GPUs [S1, unofficial].
*   $p$: Granularity factor. Each $H \times W$ image is divided into $p^2$ patches of size $H_p \times W_p$. Typical $p$: 2–8 [S1, unofficial].
*   $\alpha, \beta, \gamma, \delta$: Configurable coefficients for the objective function [S1, unofficial].

**Steps:**
1.  **Frustum Culling:** Each GPU $k$ performs frustum culling on its local shard $\mathcal{G}^{(k)}$ for every image patch $j$ in the batch. It counts the number of visible Gaussians $A_{j,k}$ needed from its own memory.
2.  **Exchange Access Pattern:** All GPUs exchange their local $A_{j,k}$ values to construct the global matrix $A$.
3.  **Initial Placement (LSA):** Solve a Linear Sum Assignment (LSA) problem to minimize total communication volume. This provides an initial placement $W_{init}$.
4.  **Local Search Refinement:** Iteratively refine $W_{init}$ to balance load (send, receive, compute) using a greedy steepest-ascent strategy. Swaps are performed until convergence or a time budget is reached.
5.  **Hierarchical Assignment (Optional):** For multi-node setups, first assign patches across machines, then assign patches within a machine to local GPUs. For intra-node assignment, set $\alpha=0$ to de-prioritize communication [S1, unofficial].

**Formulas:**
The optimization aims to minimize a relaxed objective function. The loads for GPU $k$ are defined as:

$$ \text{send}_k = \sum_{j=1}^{B} [W_j \neq k] \cdot A_{j,k} $$

$$ \text{recv}_k = \sum_{j=1}^{B} [W_j = k] \cdot \left( \sum_{i=1}^{N} A_{j,i} - A_{j,k} \right) $$

$$ \text{comp}_k = \sum_{j=1}^{B} [W_j = k] \cdot \sum_{i=1}^{N} A_{j,i} $$

Where $[P]$ is the indicator function, returning 1 if predicate $P$ is true, 0 otherwise.

The relaxed objective for local search is:

$$ \beta \cdot \|\text{send}\|_p + \gamma \cdot \|\text{recv}\|_p + \delta \cdot \|\text{comp}\|_p $$

Where $\|\text{send}\|_p = \left( \sum_{k=1}^{N} \text{send}_k^p \right)^{1/p}$, and similarly for recv and comp. The parameter $p$ in the norm is distinct from the patch granularity $p$; typically $p=2$ or $p=\infty$ is used for norms to smooth the non-smooth max terms [S1, unofficial].

The original ILP formulation (solved heuristically) is:

$$ \text{minimize } \alpha \cdot \sum_{j=1}^{B} \left( \sum_{i=1}^{N} A_{j,i} - A_{j,W_j} \right) + \beta \cdot \max_k (\text{send}_k) + \gamma \cdot \max_k (\text{recv}_k) + \delta \cdot \max_k (\text{comp}_k) $$

Subject to:
$$ \sum_{j=1}^{B} [W_j = k] = \frac{B}{N}, \quad \forall k \in \{1, \dots, N\} $$

This constraint ensures each GPU renders exactly $B/N$ patches, simplifying MPI-based system development [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def online_image_rendering_placement(batch_patches, global_gaussians, gpu_shards, num_gpus, patch_granularity, alpha, beta, gamma, delta):
    """
    batch_patches: List of B image patches.
    global_gaussians: Full set of 3D Gaussians (for reference, though shards are used).
    gpu_shards: Dict mapping GPU ID to local Gaussian shard.
    num_gpus: N.
    patch_granularity: p (integer).
    alpha, beta, gamma, delta: Objective coefficients.
    
    Returns:
    W: List of GPU IDs for each patch, length B.
    """
    B = len(batch_patches)
    
    # Step 1 & 2: Frustum Culling and Exchange
    # Each GPU computes its local A_jk and exchanges to form global A
    A = exchange_access_pattern(batch_patches, gpu_shards, num_gpus)
    
    # Step 3: Linear Sum Assignment for initial placement
    # Minimize total communication: sum(A_ji - A_j,Wj)
    # This is equivalent to maximizing sum(A_j,Wj)
    # Construct cost matrix C where C[j, k] = -A[j, k]
    cost_matrix = -A.T # Shape (N, B) or (B, N) depending on solver convention
    
    # Solve LSA (e.g., using Hungarian algorithm)
    # Result is a mapping from patches to GPUs
    W_init = solve_lsa(cost_matrix, B, num_gpus)
    
    # Step 4: Local Search Refinement
    W = refine_load_balance(W_init, A, B, num_gpus, beta, gamma, delta)
    
    return W

def exchange_access_pattern(patches, shards, num_gpus):
    """
    Performs frustum culling on each GPU and gathers the access matrix A.
    A[j, k] = number of gaussians from GPU k needed for patch j.
    """
    B = len(patches)
    A = zeros((B, num_gpus), dtype=int)
    
    # Parallel execution on each GPU
    for k in range(num_gpus):
        shard = shards[k]
        for j, patch in enumerate(patches):
            # Frustum culling logic [G]
            # 1. Transform patch bounding box to world space or gaussians to clip space.
            # 2. Count gaussians in shard that intersect the patch frustum.
            count = frustum_cull_count(shard, patch)
            A[j, k] = count
            
    # All-gather A across all GPUs
    A_global = all_gather(A)
    return A_global

def solve_lsa(cost_matrix, B, N):
    """
    Solves the Linear Sum Assignment problem.
    cost_matrix[j, k] is the cost of assigning patch j to GPU k.
    Returns W where W[j] is the assigned GPU.
    """
    # Hungarian algorithm implementation [G]
    # Standard library or custom implementation of O(B^3) Hungarian method
    row_ind, col_ind = hungarian_algorithm(cost_matrix)
    
    # row_ind contains indices 0..B-1, col_ind contains assigned GPU indices 0..N-1
    W = zeros(B, dtype=int)
    for j, k in zip(row_ind, col_ind):
        W[j] = k
        
    return W

def refine_load_balance(W, A, B, N, beta, gamma, delta):
    """
    Greedy local search to balance send, recv, and comp loads.
    """
    # Define p-norm helper
    def p_norm(vec, p=2):
        return sum(v**p for v in vec)**(1/p)
    
    current_W = W.copy()
    
    # Time budget or convergence check [G]
    max_iterations = 1000 
    for iteration in range(max_iterations):
        best_swap = None
        best_cost = infinity
        
        # Calculate current loads
        loads = calculate_loads(current_W, A, B, N)
        current_cost = beta * p_norm(loads['send']) + gamma * p_norm(loads['recv']) + delta * p_norm(loads['comp'])
        
        # Try all pairwise swaps
        for i in range(B):
            for j in range(i + 1, B):
                # Swap assignments
                temp_W = current_W.copy()
                temp_W[i], temp_W[j] = current_W[j], current_W[i]
                
                # Calculate new loads
                new_loads = calculate_loads(temp_W, A, B, N)
                new_cost = beta * p_norm(new_loads['send']) + gamma * p_norm(new_loads['recv']) + delta * p_norm(new_loads['comp'])
                
                if new_cost < current_cost:
                    if new_cost < best_cost:
                        best_cost = new_cost
                        best_swap = (i, j)
        
        if best_swap is None:
            break # Convergence
            
        i, j = best_swap
        current_W[i], current_W[j] = current_W[j], current_W[i]
        
    return current_W

def calculate_loads(W, A, B, N):
    """
    Computes send, recv, and comp loads for each GPU.
    """
    send = zeros(N)
    recv = zeros(N)
    comp = zeros(N)
    
    for k in range(N):
        for j in range(B):
            assigned_gpu = W[j]
            # Total points needed for patch j
            total_points_j = sum(A[j, :])
            
            if assigned_gpu != k:
                # GPU k sends points to the assigned GPU
                send[k] += A[j, k]
            else:
                # GPU k renders patch j
                # It receives points from other GPUs
                recv[k] += (total_points_j - A[j, k])
                # It computes for all points in patch j
                comp[k] += total_points_j
                
    return {'send': send, 'recv': recv, 'comp': comp}
```

**Edge Cases and Pitfalls:**
*   **ILP Solver Speed:** Standard ILP solvers (e.g., Gurobi) are too slow and unpredictable for online optimization per iteration [S1, unofficial]. The LSA+Local Search heuristic is required.
*   **Granularity:** Using whole images limits flexibility when scaling to many GPUs. Patching (p=2–8) allows finer-grained placement [S1, unofficial].
*   **Intra-node Communication:** For multi-node setups, intra-node bandwidth is high. Setting $\alpha=0$ during intra-node assignment de-prioritizes communication volume in favor of load balancing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The LSA step using the Hungarian method has time complexity $O(B^3)$ [S1, unofficial]. The local search step depends on the number of swaps and the cost of load recalculation, but is generally fast enough for online use.
*   **Memory Complexity:** Requires storing the $B \times N$ access pattern matrix $A$.
*   **Strengths:** Reduces inter-node communication volume and balances GPU load dynamically, which is critical for large-scale distributed training where static partitioning may become suboptimal due to random batch sampling [S1, unofficial].
*   **Limitations:** Adds computational overhead per iteration due to frustum culling and the placement solver. The heuristic does not guarantee a global optimum [S1, unofficial].

## Background
Online Image Rendering Placement is part of the Gaian system for scaling point-based differentiable rendering for large-scale reconstruction [S1, unofficial]. It addresses the limitations of offline graph partitioning, which clusters points and images for the entire dataset but cannot adapt to the random small batches used in training [S1, unofficial]. By performing placement online, the system can optimize for the specific data access patterns of each training iteration [S1, unofficial].

## Sources
[S1] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
