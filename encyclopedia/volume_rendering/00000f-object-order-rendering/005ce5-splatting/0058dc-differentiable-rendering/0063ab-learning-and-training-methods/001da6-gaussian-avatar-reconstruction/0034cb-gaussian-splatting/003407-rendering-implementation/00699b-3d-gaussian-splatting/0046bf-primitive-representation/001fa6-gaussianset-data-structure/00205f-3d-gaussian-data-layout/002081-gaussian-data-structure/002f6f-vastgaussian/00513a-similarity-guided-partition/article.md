# Similarity Guided Partition

Also called: Similarity Guided Partitioning

## Overview
Similarity Guided Partition is a graph-based clustering algorithm that groups 3D Gaussians into spatially-disjoint blocks of bounded size based on spatial proximity and appearance similarity [S1, unofficial]. It solves the problem of inefficient joint pruning by ensuring that potentially redundant Gaussians are evaluated together within the same optimization block [S1, unofficial]. It sets itself apart from naive spatial partitioning by incorporating a scale-aware affinity metric that accounts for both geometric overlap and appearance consistency [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ Gaussians $G$, a maximum block size $M$, a neighbor limit $k$, and a comparison-group size limit [S1, unofficial].
*   **Output:** A set of disjoint blocks $\{C_b\}_{b=1}^B$ such that $|C_b| \le M$ for all $b$, covering all Gaussians in $G$ [S1, unofficial].

**Data Structures**
*   **Gaussian:** Contains center $\mu_i \in \mathbb{R}^3$, activated scale vector $s_i \in \mathbb{R}^3$, and DC spherical-harmonic descriptor $c_i \in \mathbb{R}^3$ [S1, unofficial].
*   **Affinity Graph:** A sparse graph where nodes are Gaussians and edges $(i, j)$ have weights $a_{ij}$ [S1, unofficial]. Each node stores at most $k$ neighbors with the highest affinity [S1, unofficial].
*   **Block:** A set of Gaussian indices [S1, unofficial].

**Algorithm Steps**
The method proceeds in three phases: building the affinity graph, growing blocks in parallel, and filling unused capacity [S1, unofficial].

1.  **Build the Graph (Two-Pass Candidate Generation)**
    To efficiently find candidate pairs without quadratic complexity, the algorithm performs two spatial passes with different grid configurations [S1, unofficial].
    *   **Pass 1:**
        1.  Assign each Gaussian center $\mu_i$ to a 3D grid cell [S1, unofficial].
        2.  Sort occupied cells by their Morton code to preserve spatial locality [S1, unofficial].
        3.  Divide the Morton-ordered Gaussian sequence into the fewest balanced groups such that each group satisfies the comparison-group size limit [S1, unofficial].
        4.  For each group, compute pairwise affinities $a_{ij}$ for all pairs within the group [S1, unofficial].
        5.  For each Gaussian, retain at most the $k$ candidates with the largest affinities [S1, unofficial].
    *   **Pass 2:**
        1.  Shift the grid origin and enlarge the grid cell size [S1, unofficial].
        2.  Repeat steps 1-5 from Pass 1 using the new grid configuration [S1, unofficial].
    *   **Merge:**
        1.  Combine the edges from both passes into a single sparse graph [S1, unofficial].
        2.  If an edge $(i, j)$ appears in both passes, keep only one instance, recording its weight only once [S1, unofficial].

2.  **Grow Blocks in Parallel**
    *   Split the Morton-ordered sequence from the first pass into $\lceil N/M \rceil$ balanced intervals [S1, unofficial].
    *   Initialize $B = \lceil N/M \rceil$ blocks. Seed each block $C_b$ with the first Gaussian in its corresponding interval. Mark these Gaussians as assigned. All other Gaussians are unassigned [S1, unofficial].
    *   Iterate for at most 16 iterations:
        1.  For each unassigned Gaussian $i$, identify its strongest edge to an *assigned* neighbor $j$. Let $C(j)$ be the block containing $j$ [S1, unofficial].
        2.  Gaussian $i$ proposes to join $C(j)$ [S1, unofficial].
        3.  For each block $C_b$, collect all proposals [S1, unofficial].
        4.  Rank proposals by descending edge weight $a_{ij}$, then by ascending Gaussian index $i$ to break ties [S1, unofficial].
        5.  Accept the first $\min(\text{number of proposals}, M - |C_b|)$ Gaussians. Mark them as assigned and add them to $C_b$ [S1, unofficial].
        6.  If no Gaussian was assigned in this iteration, stop [S1, unofficial].

3.  **Fill Unused Capacity**
    *   Any Gaussians that remain unassigned after the growth phase are assigned to blocks to fill them up to size $M$ [S1, unofficial].
    *   Assignment follows the Morton order from the first pass. Iterate through the Morton-ordered list; if a Gaussian is unassigned, add it to the first block $C_b$ that has remaining capacity ($|C_b| < M$) [S1, unofficial].

**Formula**
The scale-aware affinity $a_{ij}$ between Gaussian $i$ and candidate $j$ is defined as [S1, unofficial]:

$$
a_{ij} = \exp \left( - \sum_{d=1}^{3} \frac{(\mu_{i,d} - \mu_{j,d})^2}{s_{i,d}^2 + s_{j,d}^2 + \epsilon_g} \right) - \|c_i - c_j\|^2
$$

Where:
*   $\mu_{i,d}$ is the $d$-th component of the center of Gaussian $i$ [S1, unofficial].
*   $s_{i,d}$ is the $d$-th component of the activated scale vector of Gaussian $i$ [S1, unofficial].
*   $c_i$ is the three-channel DC spherical-harmonic descriptor of Gaussian $i$ [S1, unofficial].
*   $\epsilon_g$ is a small constant to prevent division by zero [S1, unofficial].
*   The first term measures spatial proximity normalized by the combined scales of the two Gaussians [S1, unofficial].
*   The second term penalizes affinity between spatially close Gaussians with dissimilar appearance (high color difference) [S1, unofficial].

**Parameters**
*   $M$: Maximum allowed block size. Typical value not given in sources [S1, unofficial].
*   $k$: Number of nearest neighbors to retain per Gaussian for the sparse graph. Typical value not given in sources [S1, unofficial].
*   Comparison-group size limit: Maximum size for temporary search windows during candidate generation. Typical value not given in sources [S1, unofficial].
*   $\epsilon_g$: Small constant for numerical stability. Value not given in sources [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def morton_3d(x, y, z):
    # Standard 3D Morton code (bit interleaving) [G]
    x, y, z = x & 0x3FFFFFFF, y & 0x3FFFFFFF, z & 0x3FFFFFFF
    x = (x | (x << 16)) & 0x000000FF000000FF
    x = (x | (x << 8))  & 0x00000000FF0000FF
    x = (x | (x << 4))  & 0x000000000F000F00
    x = (x | (x << 2))  & 0x0000000003000300
    x = (x | (x << 1))  & 0x0000000003000300
    y = (y | (y << 16)) & 0x000000FF000000FF
    y = (y | (y << 8))  & 0x00000000FF0000FF
    y = (y | (y << 4))  & 0x000000000F000F00
    y = (y | (y << 2))  & 0x0000000003000300
    y = (y | (y << 1))  & 0x0000000003000300
    z = (z | (z << 16)) & 0x000000FF000000FF
    z = (z | (z << 8))  & 0x00000000FF0000FF
    z = (z | (z << 4))  & 0x000000000F000F00
    z = (z | (z << 2))  & 0x0000000003000300
    z = (z | (z << 1))  & 0x0000000003000300
    return x | (y << 1) | (z << 2)

def similarity_guided_partition(Gaussians, M, k, group_limit, epsilon_g=1e-6):
    N = len(Gaussians)
    indices = list(range(N))

    def compute_affinity(i, j):
        mu_i, mu_j = Gaussians[i].mu, Gaussians[j].mu
        s_i, s_j = Gaussians[i].s, Gaussians[j].s
        c_i, c_j = Gaussians[i].c, Gaussians[j].c
        spatial = sum((mu_i[d] - mu_j[d])**2 / (s_i[d]**2 + s_j[d]**2 + epsilon_g) for d in range(3))
        appearance = sum((c_i[d] - c_j[d])**2 for d in range(3))
        return math.exp(-spatial) - appearance

    def run_pass(origin, cell_size):
        cells = {}
        for idx in indices:
            mu = Gaussians[idx].mu
            coord = tuple(int((mu[d] - origin[d]) / cell_size) for d in range(3))
            cells.setdefault(coord, []).append(idx)
        sorted_cells = sorted(cells.keys(), key=lambda c: morton_3d(*c))
        sequence = [idx for c in sorted_cells for idx in cells[c]]
        groups = [sequence[i:i+group_limit] for i in range(0, len(sequence), group_limit)]
        local_edges = {}
        for group in groups:
            for i_idx in range(len(group)):
                for j_idx in range(i_idx + 1, len(group)):
                    u, v = group[i_idx], group[j_idx]
                    w = compute_affinity(u, v)
                    if u not in local_edges: local_edges[u] = []
                    if v not in local_edges: local_edges[v] = []
                    local_edges[u].append((v, w))
                    local_edges[v].append((u, w))
        top_k_edges = {}
        for idx in indices:
            if idx in local_edges:
                candidates = sorted(local_edges[idx], key=lambda x: (-x[1], x[0]))
                top_k_edges[idx] = candidates[:k]
        return sequence, top_k_edges

    # Pass 1
    seq1, edges1 = run_pass((0.0, 0.0, 0.0), 1.0)
    # Pass 2
    seq2, edges2 = run_pass((0.5, 0.5, 0.5), 2.0)

    # Merge edges
    merged_edges = {}
    for idx in indices:
        merged_edges[idx] = []
    for idx, neighbors in edges1.items():
        for n, w in neighbors:
            key = (min(idx, n), max(idx, n))
            if key not in merged_edges:
                merged_edges[key] = w
    for idx, neighbors in edges2.items():
        for n, w in neighbors:
            key = (min(idx, n), max(idx, n))
            if key not in merged_edges:
                merged_edges[key] = w

    # Build adjacency list from merged edges
    adj = {i: [] for i in indices}
    for (u, v), w in merged_edges.items():
        adj[u].append((v, w))
        adj[v].append((u, w))

    # Grow blocks
    num_blocks = math.ceil(N / M)
    intervals = [seq1[b*M:(b+1)*M] for b in range(num_blocks)]
    blocks = []
    assigned = [False] * N
    for interval in intervals:
        if interval:
            seed = interval[0]
            blocks.append([seed])
            assigned[seed] = True

    for _ in range(16):
        proposals = []
        for i in indices:
            if assigned[i]: continue
            best = None
            for n, w in adj[i]:
                if assigned[n]:
                    if best is None or w > best[1] or (w == best[1] and n < best[0]):
                        best = (n, w)
            if best:
                proposals.append((i, best[0], best[1]))
        if not proposals: break

        block_proposals = {}
        for i, n, w in proposals:
            b_idx = next(b for b, blk in enumerate(blocks) if n in blk)
            block_proposals.setdefault(b_idx, []).append((i, w))

        for b_idx, plist in block_proposals.items():
            plist.sort(key=lambda x: (-x[1], x[0]))
            cap = M - len(blocks[b_idx])
            for g_idx, _ in plist[:cap]:
                blocks[b_idx].append(g_idx)
                assigned[g_idx] = True

    # Fill unused capacity
    for i in seq1:
        if not assigned[i]:
            for b_idx in range(len(blocks)):
                if len(blocks[b_idx]) < M:
                    blocks[b_idx].append(i)
                    assigned[i] = True
                    break
    return blocks
```

**Edge Cases and Pitfalls**
*   **Precision:** The affinity formula involves exponentials and differences of squared terms. Small $\epsilon_g$ is required to avoid division by zero in the spatial term [S1, unofficial].
*   **Boundaries:** The two-pass strategy with shifted grids and enlarged cells is designed to prevent missing pairs that lie on the boundary of a single grid's search window [S1, unofficial].
*   **Degenerate Input:** If $N < M$, a single block is formed. If $k=0$, no edges are formed, and blocks are filled strictly by Morton order after the growth phase (which will assign no one in the growth phase) [S1, unofficial].

## Operating characteristics
Candidate generation is sub-quadratic because grid-based spatial hashing and top-k neighbor selection avoid all-pairs comparisons [S1, unofficial]. Block growth is bounded by 16 parallel iterations [S1, unofficial]. Memory stores the sparse graph with $O(N \cdot k)$ edges and block assignments $O(N)$ [S1, unofficial]. Strengths include grouping spatially and appearance-similar Gaussians for effective joint pruning [S1, unofficial]. Limitations include sensitivity to grid parameters and the overhead of the two-pass strategy [S1, unofficial]. Use when joint pruning requires appearance-aware grouping beyond naive spatial partitioning [S1, unofficial].

## Background
The method originates from SplatQ, a paper under review for ICLR 2027 [S1, unofficial]. It defines local Gaussian sets for response modeling and quadratic selection-based pruning within the SplatQ framework [S1, unofficial]. It builds on the VastGaussian framework, which partitions scenes into spatially-disjoint regions [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
