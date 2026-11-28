# Online Image Rendering Placement

Also called: Image patch placement, Online placement of image rendering

## Overview
Online Image Rendering Placement is a per-iteration optimization step in distributed 3D Gaussian Splatting training that assigns image patches to GPUs to minimize inter-node communication and balance computational load [S1, unofficial]. Unlike offline graph partitioning which clusters both points and images for the entire dataset, this method dynamically determines rendering assignments for a small, randomly selected batch of images at each training step [S1, unofficial]. It distinguishes itself by operating at the granularity of image patches rather than whole images, enabling finer-grained load balancing and reduced communication overhead in large-scale multi-GPU setups [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A batch of $B$ image patches (derived from $B$ training images), the current 3D Gaussian point cloud partitioned across $N$ GPUs, and configuration coefficients $\alpha, \beta, \gamma, \delta$.
- **Outputs:** A placement vector $W$ where $W_j$ is the GPU ID assigned to render image patch $j$.

**Data Structures:**
- **Access Pattern Matrix $A$:** A $B \times N$ matrix where $A_{j,k}$ is the number of 3D Gaussian points required from GPU $k$ to render image patch $j$. This is determined by frustum culling on each GPU.
- **Placement Vector $W$:** An array of length $B$, where each element is an integer in $\{0, \dots, N-1\}$.

**Steps:**
1. **Frustum Culling:** Each GPU $k$ receives the entire batch of $B$ image patches. It performs frustum culling against its locally stored 3D Gaussians to determine which points are visible for each patch.
2. **Exchange Access Patterns:** All GPUs exchange their culling results to construct the global access pattern matrix $A$, where $A_{j,k}$ represents the count of points needed from GPU $k$ for patch $j$ [S1, unofficial].
3. **Initial Placement via Linear Sum Assignment (LSA):** Solve a Linear Sum Assignment problem to minimize total communication volume. The objective is to minimize $\sum_{j=1}^{B} \sum_{k \neq W_j} A_{j,k}$, subject to the constraint that each GPU is assigned exactly $B/N$ patches. This yields an initial placement $W_{init}$ [S1, unofficial].
4. **Local Search Refinement:** Iteratively refine $W_{init}$ to balance load. In each step, swap the GPU assignments of two image patches if it reduces the relaxed objective function. Continue until convergence or a time budget is exhausted [S1, unofficial].
5. **Hierarchical Assignment (Optional):** If nodes contain multiple GPUs, first assign patches across machines, then assign a machine's patches among its local GPUs. For intra-node assignment, set $\alpha=0$ to de-prioritize communication [S1, unofficial].

**Formulas:**
The ILP formulation aims to minimize:
$$ \alpha \cdot \sum_{j=1}^{B} \sum_{k \neq W_j} A_{j,k} + \beta \cdot \max_k (\text{send}_k) + \gamma \cdot \max_k (\text{recv}_k) + \delta \cdot \max_k (\text{comp}_k) $$
Subject to:
$$ \sum_{j=1}^{B} [W_j = k] = \frac{B}{N}, \quad \forall k \in \{1, \dots, N\} $$

Where the loads for GPU $k$ are:
$$ \text{send}_k = \sum_{j=1}^{B} [W_j \neq k] \cdot A_{j,k} $$
$$ \text{recv}_k = \sum_{j=1}^{B} [W_j = k] \cdot \left( \sum_{i=1}^{N} A_{j,i} - A_{j,k} \right) $$
$$ \text{comp}_k = \sum_{j=1}^{B} [W_j = k] \cdot \sum_{i=1}^{N} A_{j,i} $$

To avoid plateaus in local search, the max terms are replaced with $p$-norms in the relaxed objective:
$$ \beta \cdot \|\text{send}\|_p + \gamma \cdot \|\text{recv}\|_p + \delta \cdot \|\text{comp}\|_p $$
where $\|\text{send}\|_p = \left( \sum_{k} \text{send}_k^p \right)^{1/p}$ [S1, unofficial].

**Parameters:**
- $B$: Batch size of training images. Typical value: 16–64 [S1, unofficial].
- $N$: Number of GPUs.
- $p$: Power for the $p$-norm relaxation. Not given in the sources.
- $\alpha, \beta, \gamma, \delta$: Configurable coefficients. $\alpha$ weights communication, $\beta$ send imbalance, $\gamma$ receive imbalance, $\delta$ compute imbalance [S1, unofficial].
- $p_{grid}$: Grid subdivision factor. Each $H \times W$ image is divided into $p_{grid}^2$ patches. Typical $p_{grid}$: 2–8 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def online_image_rendering_placement(batch_patches, point_clouds, num_gpus, alpha, beta, gamma, delta, p_norm=2, time_budget=1.0):
    """
    batch_patches: List of B image patches.
    point_clouds: List of N point clouds, one per GPU.
    num_gpus: N.
    Returns: W, list of GPU IDs for each patch.
    """
    B = len(batch_patches)
    
    # Step 1 & 2: Frustum Culling and Access Matrix Construction
    # Each GPU k determines which points from its local cloud are needed for each patch
    # A[j, k] = number of points from GPU k needed for patch j
    A = zeros((B, num_gpus))
    for k in range(num_gpus):
        for j in range(B):
            # Perform frustum culling for patch j against point_clouds[k]
            visible_points = frustum_cull(batch_patches[j], point_clouds[k])
            A[j, k] = len(visible_points)
            
    # Step 3: Linear Sum Assignment for initial placement
    # Minimize total communication: sum over j of sum over k!=W[j] of A[j,k]
    # Constraint: Each GPU gets exactly B/N patches
    # This is a standard LSA problem. We construct a cost matrix C where C[j, k] = A[j, k]
    # But we want to minimize communication, which is sum_{k != W[j]} A[j,k].
    # Note: sum_{k != W[j]} A[j,k] = (sum_{all k} A[j,k]) - A[j, W[j]]
    # Minimizing sum_{j} (Total_A[j] - A[j, W[j]]) is equivalent to Maximizing sum_{j} A[j, W[j]]
    # So we maximize the diagonal elements in the assignment.
    
    # Construct cost matrix for LSA (maximization problem converted to minimization)
    # Cost[j, k] = -A[j, k]
    cost_matrix = -A
    
    # Solve LSA using Hungarian algorithm
    # Returns row_indices, col_indices where col_indices[j] is the GPU assigned to patch j
    row_indices, col_indices = hungarian_algorithm(cost_matrix)
    W_init = col_indices
    
    # Step 4: Local Search Refinement
    W = W_init.copy()
    start_time = time.time()
    
    while time.time() - start_time < time_budget:
        improved = False
        # Try swapping assignments of two patches
        for i in range(B):
            for j in range(i + 1, B):
                # Calculate current objective
                current_obj = compute_relaxed_objective(W, A, num_gpus, beta, gamma, delta, p_norm)
                
                # Try swap
                W[i], W[j] = W[j], W[i]
                new_obj = compute_relaxed_objective(W, A, num_gpus, beta, gamma, delta, p_norm)
                
                if new_obj < current_obj:
                    improved = True
                    # Keep the swap
                else:
                    # Revert swap
                    W[i], W[j] = W[j], W[i]
                    
        if not improved:
            break
            
    return W

def compute_relaxed_objective(W, A, num_gpus, beta, gamma, delta, p_norm):
    B = len(W)
    send = zeros(num_gpus)
    recv = zeros(num_gpus)
    comp = zeros(num_gpus)
    
    for k in range(num_gpus):
        for j in range(B):
            if W[j] != k:
                send[k] += A[j, k]
            if W[j] == k:
                total_points_for_patch = sum(A[j, :])
                recv[k] += (total_points_for_patch - A[j, k])
                comp[k] += total_points_for_patch
                
    # Compute p-norms
    send_norm = (sum(s**p_norm for s in send))**(1/p_norm)
    recv_norm = (sum(r**p_norm for r in recv))**(1/p_norm)
    comp_norm = (sum(c**p_norm for c in comp))**(1/p_norm)
    
    return beta * send_norm + gamma * recv_norm + delta * comp_norm
```

**Edge Cases and Pitfalls:**
- **ILP Solver Performance:** Standard ILP solvers like Gurobi are too slow and unpredictable for online optimization within a training iteration [S1, unofficial]. The heuristic LSA+local search approach is necessary.
- **Granularity:** Using whole images for placement severely limits scalability when the number of GPUs is large, as the batch size $B$ is small (16–64). Patching allows finer-grained assignment [S1, unofficial].
- **Intra-node Communication:** For GPUs within the same node, communication bandwidth is high. Setting $\alpha=0$ for intra-node assignment de-prioritizes communication volume, focusing on load balancing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The LSA step using the Hungarian method has a time complexity of $O(B^3)$ [S1, unofficial]. The local search step's complexity depends on the number of swaps performed and the time budget, but is generally kept small to fit within the training iteration time.
- **Memory Complexity:** Requires storing the access pattern matrix $A$ of size $B \times N$ and the placement vector $W$ of size $B$. This is negligible compared to the point cloud data.
- **Strengths:** Reduces inter-node communication by co-locating rendering with data access. Balances computational load across GPUs. Works online, adapting to the random batch selection in each training step.
- **Limitations:** The heuristic may not find the global optimum. The local search requires a time budget, which adds latency to each training iteration if not carefully tuned.
- **When to Use:** Large-scale distributed training of 3D Gaussian Splatting with many GPUs and limited inter-node bandwidth.
- **When Not to Use:** Small-scale setups with few GPUs where communication overhead is minimal, or when the overhead of the placement algorithm exceeds the communication savings.

## Background
Online Image Rendering Placement was introduced in the Gaian system for scaling point-based differentiable rendering for large-scale reconstruction [S1, unofficial]. It addresses the limitations of offline graph partitioning, which is static and does not adapt to the dynamic nature of training batches. The method leverages Linear Sum Assignment and local search to provide a fast, effective solution for dynamic resource allocation in distributed rendering.

## Sources
[S1] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
