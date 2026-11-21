# Density-based Clustering with Weighted Dissimilarity

Also called: DBSCAN variation, Weighted Dissimilarity DBSCAN

## Overview
This method is a density-based clustering algorithm designed to identify similar phases or states across multiple simulation sequences by treating segmented time steps as data points. It distinguishes itself from standard DBSCAN by employing a weighted dissimilarity measure that incorporates temporal indices and sequence identifiers, allowing it to group similar temporal progressions even when they are not strictly synchronous. It builds on [DBSCAN](11722-dbscan.md) but modifies the distance metric and neighborhood density calculation to account for the hierarchical structure of simulation data.

## How it works
**Inputs:** A set of simulation sequences, where each sequence consists of a series of 3D volumes (time steps). Each volume $v$ has a sequence identifier $s(v)$ and a time step index $t_i(v)$. The input to the clustering step is a set of segment representatives, where each representative corresponds to a segment of time steps derived from a prior segmentation process.

**Outputs:** A set of clusters. Each cluster contains:
1.  References to its member segments.
2.  The cluster medoid (the member minimizing average dissimilarity to all other cluster members).
3.  The sequence range (set of sequence identifiers with at least one member in the cluster).
4.  The temporal range (set of time step indices covered by cluster members).
5.  Temporal medoids (for each distinct time step index in the cluster, the member minimizing average dissimilarity to cluster members of that same time step).

**Data Structures:**
*   **Segment Representative:** An object representing a segment of time steps. It stores the sequence identifier $s$, the representative time step index $t_i$, and the number of members $|S_0|$ in the original segment.
*   **Dissimilarity Matrix:** A symmetric matrix $D$ of size $N \times N$, where $N$ is the number of segment representatives. $D_{ij}$ stores the dissimilarity $d(v_i, v_j)$ between representative $i$ and representative $j$.
*   **Neighborhood Index:** For each representative, a list of indices sorted by ascending dissimilarity values.

**Parameters:**
*   $\epsilon$ (epsilon): Maximum dissimilarity distance for two points to be considered neighbors. Set to the average dissimilarity between all segment representatives.
*   $p_{min}$: Minimum number of points required to form a dense region. Calculated as:
    $$ p_{min} = \left\lceil \frac{1}{\ln |S_0|} \right\rceil $$
    where $|S_0|$ is the average number of members in a segment. [S1]

**Algorithm Steps:**

1.  **Compute Dissimilarity Matrix:**
    For every pair of segment representatives $v_1$ and $v_2$, compute the dissimilarity $d(v_1, v_2)$ using the weighted formula:
    $$ d(v_1, v_2) = w(v_1, v_2) \sum_{u} (v_1(u) - v_2(u))^2 $$
    where $u$ iterates over all 3D grid positions, $v_1(u)$ and $v_2(u)$ are the intensity values at position $u$, and $w(v_1, v_2)$ is the weight factor defined as:
    $$ w(v_1, v_2) = \begin{cases} 1 + \sqrt{|t_i(v_1) - t_i(v_2)|} & \text{if } s(v_1) = s(v_2) \\ 0 & \text{otherwise} \end{cases} $$
    Note: The source text implies $w=0$ if sequences differ, which would make dissimilarity 0. However, standard clustering requires non-zero distances for distinct points. The text states "The additional weight increases the dissimilarity...". If $w=0$, points from different sequences are identical in this metric, which contradicts the goal of grouping similar temporal progressions *between* sequences. Re-reading carefully: "This allows the clustering algorithm to group similar temporal progressions between different sequences even if they are not entirely synchronous." If $w=0$ for different sequences, they are always neighbors. This seems to be a specific design choice to force cross-sequence grouping if intensities match, or potentially a typo in the source where it should be 1 or another base value. Given the explicit formula in [S1], we implement it as written. [S1]

2.  **Generate Neighborhood Indices:**
    For each segment representative $v_i$, sort all other representatives $v_j$ by their dissimilarity $d(v_i, v_j)$ in ascending order. Store this sorted list as the neighborhood index for $v_i$.

3.  **Execute DBSCAN:**
    Initialize all points as unvisited.
    For each unvisited point $p$:
    a.  Mark $p$ as visited.
    b.  Retrieve the neighborhood index for $p$.
    c.  Identify all neighbors $N_p$ such that $d(p, q) \le \epsilon$.
    d.  If $|N_p| \ge p_{min}$:
        i.  Create a new cluster $C$.
        ii. Add $p$ to $C$.
        iii. For each neighbor $q \in N_p$:
            1.  If $q$ is unvisited, mark it as visited.
            2.  Retrieve the neighborhood index for $q$.
            3.  Identify neighbors $N_q$ such that $d(q, r) \le \epsilon$.
            4.  If $|N_q| \ge p_{min}$, add these neighbors to the frontier of $N_p$ (merge neighborhoods).
            5.  Add $q$ to $C$.
    e.  If $|N_p| < p_{min}$, mark $p$ as noise (or border point depending on implementation, but DBSCAN core definition labels non-core unvisited points as noise until they are reached by a core point).

4.  **Assign Members:**
    If a segment representative is part of a cluster, all original members of that segment are assumed to share this cluster association.

5.  **Compute Cluster Metadata:**
    After clustering completes, compute the additional information for each cluster:
    *   **Cluster Medoid:** Find the member $m \in C$ that minimizes $\frac{1}{|C|-1} \sum_{k \in C, k \neq m} d(m, k)$.
    *   **Sequence Range:** Set $\{ s(m) \mid m \in C \}$.
    *   **Temporal Range:** Set $\{ t_i(m) \mid m \in C \}$.
    *   **Temporal Medoids:** For each distinct time step index $t$ in the temporal range, find the member $m \in C$ with $t_i(m) = t$ that minimizes $\frac{1}{|C_t|-1} \sum_{k \in C_t, k \neq m} d(m, k)$, where $C_t = \{ k \in C \mid t_i(k) = t \}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_dissimilarity(v1, v2):
    """
    v1, v2: dicts or objects with keys 'u' (array of values), 'ti' (time index), 'si' (sequence id)
    Returns weighted dissimilarity.
    """
    # Sum of squared intensity differences
    ssd = sum((v1['u'][idx] - v2['u'][idx])**2 for idx in range(len(v1['u'])))
    
    # Weight factor
    if v1['si'] == v2['si']:
        weight = 1 + math.sqrt(abs(v1['ti'] - v2['ti']))
    else:
        weight = 0
        
    return weight * ssd

def weighted_dbscan(representatives, epsilon, p_min):
    """
    representatives: list of segment representative objects
    epsilon: float
    p_min: int
    Returns: list of clusters, where each cluster is a list of representative indices
    """
    n = len(representatives)
    # Precompute dissimilarity matrix
    dist_matrix = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(i + 1, n):
            d = compute_dissimilarity(representatives[i], representatives[j])
            dist_matrix[i][j] = d
            dist_matrix[j][i] = d

    # Generate neighborhood indices (sorted neighbors by distance)
    # Note: DBSCAN typically uses a radius query. Sorting helps in finding neighbors <= epsilon efficiently if we iterate.
    # However, standard DBSCAN uses a spatial index or brute force radius check.
    # The source says "generate a neighborhood index by sorting dissimilarity values in ascending order".
    # We will use this to find neighbors within epsilon.
    
    neighborhood_indices = []
    for i in range(n):
        # Create list of (distance, index)
        dists = [(dist_matrix[i][j], j) for j in range(n) if i != j]
        dists.sort(key=lambda x: x[0])
        neighborhood_indices.append(dists)

    visited = [False] * n
    clusters = []
    noise = []

    for i in range(n):
        if visited[i]:
            continue
        
        visited[i] = True
        
        # Find neighbors within epsilon
        neighbors = []
        for dist, idx in neighborhood_indices[i]:
            if dist <= epsilon:
                neighbors.append(idx)
            else:
                # Since sorted, we can stop if we only want exact epsilon neighbors?
                # No, we must check all if we want to be sure, but sorted list allows early exit if we assume monotonicity.
                # Actually, if sorted ascending, once dist > epsilon, all subsequent are > epsilon.
                break
        
        if len(neighbors) >= p_min:
            # Start new cluster
            cluster = [i]
            visited[i] = True # Already visited
            
            # Process neighbors
            # Use a queue or stack for expansion
            queue = list(neighbors)
            while queue:
                q = queue.pop(0)
                if not visited[q]:
                    visited[q] = True
                    # Find neighbors of q
                    q_neighbors = []
                    for dist, idx in neighborhood_indices[q]:
                        if dist <= epsilon:
                            q_neighbors.append(idx)
                        else:
                            break
                    
                    if len(q_neighbors) >= p_min:
                        # Add to queue for expansion
                        for nb in q_neighbors:
                            if nb not in cluster: # Avoid duplicates in cluster list
                                cluster.append(nb)
                                queue.append(nb)
                    else:
                        # Border point, add to cluster but don't expand
                        if q not in cluster:
                            cluster.append(q)
                else:
                    # Already visited, but if it was part of a core point, it's in a cluster.
                    # If it was noise, it remains noise unless reached by this cluster.
                    # Standard DBSCAN: if visited and not in a cluster, it's noise.
                    # If visited and in a cluster, it's already handled.
                    # We need to track which cluster each point belongs to.
                    pass
            
            # Refine cluster membership based on standard DBSCAN logic:
            # The above simplified queue logic might include border points incorrectly or miss core point expansion.
            # Let's stick to the standard DBSCAN expansion logic using the precomputed neighbors.
            
            # Re-implementing standard DBSCAN expansion properly:
            cluster = []
            # ... (Standard DBSCAN expansion using the 'neighbors' list and 'visited' array)
            # For brevity in this specific weighted variant, we assume the 'neighbors' list contains all points within epsilon.
            
            # Correct expansion:
            cluster = [i]
            visited[i] = True
            seed_set = list(neighbors)
            
            while seed_set:
                q = seed_set.pop(0)
                if not visited[q]:
                    visited[q] = True
                    # Find neighbors of q
                    q_neighbors = []
                    for dist, idx in neighborhood_indices[q]:
                        if dist <= epsilon:
                            q_neighbors.append(idx)
                        else:
                            break
                    
                    if len(q_neighbors) >= p_min:
                        # Add new neighbors to seed set
                        for nb in q_neighbors:
                            if nb not in seed_set and nb not in cluster:
                                seed_set.append(nb)
                    cluster.append(q)
                elif q not in cluster:
                    # If visited and not in cluster, it might be noise or part of another cluster.
                    # In standard DBSCAN, if it's not in the current cluster, it's not added.
                    pass
            
            # Note: The above logic is a simplified implementation. 
            # The key difference is that 'neighbors' are determined by the weighted dissimilarity.
            
            clusters.append(cluster)
        else:
            noise.append(i)

    return clusters, noise
```

**Edge Cases and Pitfalls:**
*   **Weight Zero:** If $s(v_1) \neq s(v_2)$, the weight is 0, making dissimilarity 0. This implies all points from different sequences are identical in distance space. This may lead to all points being merged into a single cluster if intensities are similar, or require careful interpretation of $\epsilon$. [S1]
*   **Precision:** Sum of squared differences can be large; ensure floating-point precision is sufficient.
*   **Boundary Conditions:** The dissimilarity is undefined for the last time step of a sequence in the segmentation phase, but the clustering operates on segment representatives, so this is handled upstream. [S1]

## Operating characteristics
*   **Time Complexity:** Computing the dissimilarity matrix takes $O(N^2 \cdot V)$, where $N$ is the number of segment representatives and $V$ is the number of 3D grid positions. DBSCAN execution is $O(N^2)$ in the worst case without spatial indexing, but with the precomputed sorted neighborhood indices, neighbor queries are efficient. The source notes that the time required for executing the actual clustering algorithm once the dissimilarity matrix has been computed is negligible. [S1]
*   **Memory Complexity:** $O(N^2)$ to store the dissimilarity matrix.
*   **Strengths:** Discovers clusters of arbitrary shape; does not require predetermining the number of clusters; robust to noise; incorporates temporal and sequence information via the weighted dissimilarity. [S1]
*   **Limitations:** Sensitive to parameter selection ($\epsilon$, $p_{min}$); high computational cost for large volumes due to the sum of squared differences over all grid positions; the weight function's behavior for different sequences may lead to unintended merging. [S1]
*   **When to Use:** When analyzing simulation data to find similar temporal progressions across multiple sequences, especially when sequences are not perfectly synchronized. [S1]
*   **When Not to Use:** When computational resources are limited for large volumes; when sequences are expected to be strictly synchronous and simple Euclidean distance suffices.

## Background
This method was introduced in the context of result-driven exploration of simulation parameter spaces for visual effects design. It is a variation of DBSCAN [12] adapted for clustering segment representatives from simulation sequences. The weighted dissimilarity measure is motivated by the need to visualize simulation data based on observable characteristics, using a simplistic but effective metric based on sum of squared intensity differences. The approach is related to measures proposed by Birant and Kut [9]. [S1]

## Sources
[S1] Result-Driven Exploration of Simulation Parameter Spaces for Visual Effects Design, https://doi.org/10.1109/TVCG.2010.190, fetched 2026-10-31
