# Similarity Guided Partition

Also called: Similarity Guided Partitioning

## Overview
Similarity Guided Partition is a graph-based clustering algorithm that groups 3D Gaussians into spatially-disjoint blocks of bounded size based on spatial proximity and appearance similarity [S1, unofficial]. It solves the problem of inefficient joint pruning by ensuring that potentially redundant Gaussians are evaluated together within the same optimization block [S1, unofficial]. It sets itself apart from naive spatial partitioning by incorporating a scale-aware affinity metric that accounts for both geometric overlap and appearance consistency, and from sibling methods like Page Assignment and Merging by using a parallel greedy growth strategy on a sparse affinity graph rather than page-based memory management [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ Gaussians $G$, a maximum block size $M$, a neighbor limit $k$, and a comparison-group size limit.
*   **Output:** A set of disjoint blocks $\{C_b\}_{b=1}^B$ such that $|C_b| \le M$ for all $b$, covering all Gaussians in $G$.

**Data Structures**
*   **Gaussian:** Contains center $\mu_i \in \mathbb{R}^3$, activated scale vector $s_i \in \mathbb{R}^3$, and DC spherical-harmonic descriptor $c_i \in \mathbb{R}^3$.
*   **Affinity Graph:** A sparse graph where nodes are Gaussians and edges $(i, j)$ have weights $a_{ij}$. Each node stores at most $k$ neighbors with the highest affinity.
*   **Block:** A set of Gaussian indices.

**Algorithm Steps**
The method proceeds in three phases: building the affinity graph, growing blocks in parallel, and filling unused capacity.

1.  **Build the Graph (Two-Pass Candidate Generation)**
    To efficiently find candidate pairs without quadratic complexity, the algorithm performs two spatial passes with different grid configurations.
    *   **Pass 1:**
        1.  Assign each Gaussian center $\mu_i$ to a 3D grid cell.
        2.  Sort occupied cells by their Morton code (Z-order curve) to preserve spatial locality.
        3.  Divide the Morton-ordered Gaussian sequence into the fewest balanced groups such that each group satisfies the comparison-group size limit.
        4.  For each group, compute pairwise affinities $a_{ij}$ for all pairs within the group.
        5.  For each Gaussian, retain at most the $k$ candidates with the largest affinities.
    *   **Pass 2:**
        1.  Shift the grid origin and enlarge the grid cell size.
        2.  Repeat steps 1-5 from Pass 1 using the new grid configuration.
    *   **Merge:**
        1.  Combine the edges from both passes into a single sparse graph.
        2.  If an edge $(i, j)$ appears in both passes, keep only one instance, retaining the weight calculated from the first pass (or consistently defined, as the sources state "record their weights only once" [S1, unofficial]).

2.  **Grow Blocks in Parallel**
    *   Split the Morton-ordered sequence from the first pass into $\lceil N/M \rceil$ balanced intervals.
    *   Initialize $B = \lceil N/M \rceil$ blocks. Seed each block $C_b$ with the first Gaussian in its corresponding interval. Mark these Gaussians as assigned. All other Gaussians are unassigned.
    *   Iterate for at most 16 iterations:
        1.  For each unassigned Gaussian $i$, identify its strongest edge to an *assigned* neighbor $j$. Let $C(j)$ be the block containing $j$.
        2.  Gaussian $i$ proposes to join $C(j)$.
        3.  For each block $C_b$, collect all proposals.
        4.  Rank proposals by descending edge weight $a_{ij}$, then by ascending Gaussian index $i$ to break ties.
        5.  Accept the first $\min(\text{number of proposals}, M - |C_b|)$ Gaussians. Mark them as assigned and add them to $C_b$.
        6.  If no Gaussian was assigned in this iteration, stop.

3.  **Fill Unused Capacity**
    *   Any Gaussians that remain unassigned after the growth phase are assigned to blocks to fill them up to size $M$.
    *   Assignment follows the Morton order from the first pass. Iterate through the Morton-ordered list; if a Gaussian is unassigned, add it to the first block $C_b$ that has remaining capacity ($|C_b| < M$).

**Formula**
The scale-aware affinity $a_{ij}$ between Gaussian $i$ and candidate $j$ is defined as [S1, unofficial]:

$$
a_{ij} = \exp \left( - \sum_{d=1}^{3} \frac{(\mu_{i,d} - \mu_{j,d})^2}{s_{i,d}^2 + s_{j,d}^2 + \epsilon_g} \right) - \|c_i - c_j\|^2
$$

Where:
*   $\mu_{i,d}$ is the $d$-th component of the center of Gaussian $i$.
*   $s_{i,d}$ is the $d$-th component of the activated scale vector of Gaussian $i$.
*   $c_i$ is the three-channel DC spherical-harmonic descriptor of Gaussian $i$.
*   $\epsilon_g$ is a small constant to prevent division by zero.
*   The first term measures spatial proximity normalized by the combined scales of the two Gaussians.
*   The second term penalizes affinity between spatially close Gaussians with dissimilar appearance (high color difference).

**Parameters**
*   $M$: Maximum allowed block size. Typical value not given in sources.
*   $k$: Number of nearest neighbors to retain per Gaussian for the sparse graph. Typical value not given in sources.
*   Comparison-group size limit: Maximum size for temporary search windows during candidate generation. Typical value not given in sources.
*   $\epsilon_g$: Small constant for numerical stability. Value not given in sources.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def similarity_guided_partition(Gaussians, M, k, group_limit, epsilon_g=1e-6):
    """
    Partitions Gaussians into disjoint blocks of size <= M using similarity guided partitioning.
    
    Args:
        Gaussians: List of Gaussian objects with attributes:
            - mu: numpy array [3] center
            - s: numpy array [3] activated scale
            - c: numpy array [3] DC SH descriptor
        M: int, max block size
        k: int, max neighbors to keep
        group_limit: int, max size of temporary search groups
        epsilon_g: float, small constant for affinity formula
        
    Returns:
        blocks: List of lists of Gaussian indices
    """
    N = len(Gaussians)
    indices = list(range(N))
    
    # Helper to compute affinity
    def compute_affinity(i, j):
        mu_i, mu_j = Gaussians[i].mu, Gaussians[j].mu
        s_i, s_j = Gaussians[i].s, Gaussians[j].s
        c_i, c_j = Gaussians[i].c, Gaussians[j].c
        
        spatial_term = 0.0
        for d in range(3):
            diff = mu_i[d] - mu_j[d]
            denom = s_i[d]**2 + s_j[d]**2 + epsilon_g
            spatial_term += (diff ** 2) / denom
            
        appearance_term = np.linalg.norm(c_i - c_j) ** 2
        
        return math.exp(-spatial_term) - appearance_term

    # Phase 1: Build Graph
    # We need to perform two passes. 
    # Pass 1: Standard grid
    graph_edges = {} # (i, j) -> weight, i < j
    
    def run_pass(grid_origin, grid_cell_size):
        # Assign centers to grid cells
        # Morton order sorting
        # Split into groups
        # Compute affinities and keep top-k
        
        # 1. Assign to grid cells
        cells = {} # cell_coord -> list of gaussian_indices
        for idx in indices:
            g = Gaussians[idx]
            # Simple grid assignment (assuming normalized or known bounds, 
            # here we assume a generic hashable coord)
            # Note: Exact grid coordinate calculation depends on scene bounds, 
            # which are not specified. We assume a helper `get_cell_coord` exists.
            coord = get_cell_coord(g.mu, grid_origin, grid_cell_size)
            if coord not in cells:
                cells[coord] = []
            cells[coord].append(idx)
            
        # 2. Sort cells by Morton code
        # Morton code generation helper
        sorted_cells = sorted(cells.keys(), key=lambda c: morton_code(c))
        
        # 3. Flatten to sequence
        sequence = []
        for c in sorted_cells:
            sequence.extend(cells[c])
            
        # 4. Split into groups
        groups = []
        for i in range(0, len(sequence), group_limit):
            groups.append(sequence[i:i+group_limit])
            
        # 5. Compute affinities and keep top-k
        # For each group, compute pairwise affinities
        for group in groups:
            n_group = len(group)
            # Store potential edges for this group
            local_edges = []
            for i_idx in range(n_group):
                for j_idx in range(i_idx + 1, n_group):
                    gi = group[i_idx]
                    gj = group[j_idx]
                    w = compute_affinity(gi, gj)
                    local_edges.append((gi, gj, w))
            
            # Keep top-k for each Gaussian
            # We need to aggregate edges per Gaussian
            gaussian_candidates = {idx: [] for idx in group}
            for u, v, w in local_edges:
                gaussian_candidates[u].append((v, w))
                gaussian_candidates[v].append((u, w))
                
            for idx in group:
                # Sort by weight desc, then index asc
                candidates = sorted(gaussian_candidates[idx], key=lambda x: (-x[1], x[0]))
                top_k = candidates[:k]
                for neighbor, weight in top_k:
                    # Add to global graph if not present or update?
                    # Sources say "merge... repeated edges record their weights only once"
                    # We'll store them and merge later.
                    edge_key = (min(idx, neighbor), max(idx, neighbor))
                    if edge_key not in graph_edges:
                        graph_edges[edge_key] = weight
                    # If it exists, we keep the first one encountered (Pass 1)
                    
    # Pass 1
    run_pass(grid_origin=(0,0,0), grid_cell_size=1.0) # Placeholder values
    
    # Pass 2: Shifted grid and enlarged cell size
    # "Shift the grid origin and enlarge the cell size"
    run_pass(grid_origin=(0.5,0.5,0.5), grid_cell_size=2.0) # Placeholder values
    
    # The graph_edges now contains the sparse graph
    
    # Phase 2: Grow Blocks
    # Split first-pass Morton sequence into intervals
    # We need the Morton sequence from Pass 1. 
    # Re-running the sorting logic to get the sequence is necessary for implementation.
    # For brevity, assume `morton_sequence` is available from Pass 1.
    
    num_blocks = math.ceil(N / M)
    intervals = []
    for b in range(num_blocks):
        start = b * M
        end = min((b + 1) * M, N)
        intervals.append(morton_sequence[start:end])
        
    blocks = []
    assigned = [False] * N
    
    for interval in intervals:
        if not interval:
            continue
        seed = interval[0]
        blocks.append([seed])
        assigned[seed] = True
        
    # Parallel growth loop
    for iteration in range(16):
        new_assignments = [] # List of (gaussian_idx, block_idx, weight)
        
        for i in range(N):
            if assigned[i]:
                continue
            
            # Find strongest edge to an assigned neighbor
            best_neighbor = -1
            best_weight = -float('inf')
            best_block = -1
            
            # Check neighbors in graph
            # Graph is stored in graph_edges. We need to look up edges for i.
            # This requires an adjacency list or iterating edges.
            # For efficiency, assume adjacency list `adj` built from graph_edges.
            for j, w in adj[i]:
                if assigned[j]:
                    if w > best_weight or (w == best_weight and j < best_neighbor):
                        best_weight = w
                        best_neighbor = j
                        best_block = get_block_index(j, blocks)
            
            if best_neighbor != -1:
                new_assignments.append((i, best_block, best_weight))
                
        if not new_assignments:
            break
            
        # Process proposals per block
        proposals = {} # block_idx -> list of (gaussian_idx, weight)
        for i, b_idx, w in new_assignments:
            if b_idx not in proposals:
                proposals[b_idx] = []
            proposals[b_idx].append((i, w))
            
        for b_idx, prop_list in proposals.items():
            # Rank by desc weight, then asc index
            prop_list.sort(key=lambda x: (-x[1], x[0]))
            
            block = blocks[b_idx]
            capacity = M - len(block)
            
            if capacity > 0:
                to_add = prop_list[:capacity]
                for g_idx, _ in to_add:
                    blocks[b_idx].append(g_idx)
                    assigned[g_idx] = True
                    
    # Phase 3: Fill unused capacity
    # Assign remaining unassigned Gaussians in Morton order
    for i in morton_sequence:
        if not assigned[i]:
            # Find first block with capacity
            for b_idx in range(len(blocks)):
                if len(blocks[b_idx]) < M:
                    blocks[b_idx].append(i)
                    assigned[i] = True
                    break
                    
    return blocks

def get_block_index(gaussian_idx, blocks):
    for b_idx, block in enumerate(blocks):
        if gaussian_idx in block:
            return b_idx
    return -1

def get_cell_coord(mu, origin, cell_size):
    # Placeholder for actual grid coordinate calculation
    return (int((mu[0] - origin[0]) / cell_size), 
            int((mu[1] - origin[1]) / cell_size), 
            int((mu[2] - origin[2]) / cell_size))

def morton_code(coord):
    # Placeholder for Morton code calculation
    return 0
```

**Edge Cases and Pitfalls**
*   **Precision:** The affinity formula involves exponentials and differences of squared terms. Small $\epsilon_g$ is required to avoid division by zero in the spatial term [S1, unofficial].
*   **Boundaries:** The two-pass strategy with shifted grids and enlarged cells is designed to prevent missing pairs that lie on the boundary of a single grid's search window [S1, unofficial].
*   **Degenerate Input:** If $N < M$, a single block is formed. If $k=0$, no edges are formed, and blocks are filled strictly by Morton order after the growth phase (which will assign no one in the growth phase) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The candidate generation phase is sub-quadratic due to grid-based spatial hashing and limiting comparisons to local groups and top-$k$ neighbors. The graph construction is $O(N \cdot k)$. The block growth phase is bounded by 16 iterations, with each iteration involving proposal ranking, which is linear in the number of proposals. Overall complexity is significantly better than all-pairs $O(N^2)$ [S1, unofficial].
*   **Memory Complexity:** Stores the sparse graph with $O(N \cdot k)$ edges and the block assignments $O(N)$ [S1, unofficial].
*   **Strengths:** Ensures that spatially and appearance-similar Gaussians are grouped, which improves the effectiveness of joint pruning. The parallel growth strategy is efficient.
*   **Limitations:** Requires careful tuning of grid parameters (cell size, shift) and hyperparameters ($M, k$). The two-pass approach adds overhead compared to a single pass.
*   **When to use:** When joint pruning of Gaussians is required and naive spatial partitioning leads to suboptimal pruning results due to appearance mismatches within blocks [S1, unofficial].

## Background
Similarity Guided Partition was introduced in the paper "SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection" (2026) [S1, unofficial]. It is used as a core component in the SplatQ method to define local Gaussian sets for response modeling and quadratic selection-based pruning. It builds on the VastGaussian framework, which partitions the scene into spatially-disjoint regions [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
