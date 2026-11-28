# Multilevel Hypergraph Partitioning

Also called: Multilevel bisection, Multilevel methods

## Overview
Multilevel Hypergraph Partitioning is a paradigm for solving the NP-hard hypergraph partitioning problem by reducing computational complexity through hierarchical abstraction [S1, unofficial]. It addresses the remapping problem in image-space parallel volume rendering, where changing visualization parameters disrupts load balance and increases communication costs [S1, unofficial]. Unlike direct partitioning methods, it enhances iterative refinement by operating on a simplified graph structure, achieving high-quality partitions with significantly reduced runtime [S1, unofficial].

## How it works
The method consists of three primary phases: Coarsening, Initial Partitioning, and Uncoarsening, often followed by Refinement [S1, unofficial].

**Inputs:** A hypergraph $H_0 = (V_0, N_0)$, where $V_0$ is the set of vertices and $N_0$ is the set of hyperedges. The goal is to partition $V_0$ into $K$ parts.
**Outputs:** A partition of $V_0$ into $K$ disjoint sets $P_1, \dots, P_K$ such that $\bigcup P_i = V_0$ and $|P_i| \approx |V_0|/K$, minimizing cut size (hyperedges spanning multiple parts) while respecting load balance constraints.

### Data Structures
*   **Hypergraph Level $H_i$:** Defined by vertex set $V_i$ and hyperedge set $N_i$.
*   **Multinode:** A group of original vertices merged into a single vertex in a coarser level.
*   **Gain:** A metric used in initial partitioning to determine the benefit of moving a vertex to a cluster.

### Algorithm Steps

1.  **Coarsening:**
    Construct a sequence of hypergraphs $H_0, H_1, \dots, H_m$ where $H_m$ is the coarsest.
    *   For each level $i$ from $0$ to $m-1$:
        *   Cluster vertices from $H_i$ into multinodes to form $H_{i+1}$.
        *   Clustering reduces the vertex count $|V_{i+1}| < |V_i|$ and increases vertex degrees [S1, unofficial].
        *   Two clustering techniques are used:
            *   **Matching-based:** Vertices are visited in random or predetermined order. Each vertex is grouped with one unmatched vertex. The pair forms a multinode (single vertex in $H_{i+1}$). Vertices are marked as matched [S1, unofficial].
            *   **Agglomerative:** Allows more than two vertices to form a multinode. A single vertex can join an existing multinode [S1, unofficial].
    *   Repeat until $|V_m|$ reaches a desired small threshold.

2.  **Initial Partitioning:**
    Partition the coarsest hypergraph $H_m$ into two nearly equal-sized parts.
    *   **Greedy Hypergraph Growing (used in PaToH):**
        *   Select a random vertex as the seed.
        *   Grow a cluster by moving vertices into it based on their gain, starting with the highest gain [S1, unofficial].
        *   Stop when a fixed load balance criteria between the two resulting parts is satisfied [S1, unofficial].
        *   Assign vertices in the cluster to Part 1; remaining vertices to Part 2 [S1, unofficial].
    *   *Alternative:* Direct $K$-way partitioning can be used if $H_m$ is coarsened until $K$ vertices remain, but recursive bisection is more common [S1, unofficial].

3.  **Uncoarsening:**
    Project the partition from $H_i$ back to $H_{i-1}$ for $i = m$ down to $1$.
    *   For each vertex in $H_{i-1}$ that was merged into a multinode in $H_i$, assign it to the same part as its parent multinode in $H_i$.
    *   This provides an initial partition for the finer hypergraph [S1, unofficial].

4.  **Refinement:**
    After uncoarsening to a finer level, apply a refinement heuristic to improve partition quality.
    *   Use algorithms similar to Fiduccia-Mattheyses (FM) or Kernighan-Lin (KL) [S1, unofficial].
    *   Move vertices between parts to minimize the objective function (cut size) while ensuring load constraints are not violated [S1, unofficial].

5.  **Multi-way Partitioning (Recursive Bisection):**
    *   To partition into $K$ parts, apply the two-way bisection recursively.
    *   A two-way bisection is further partitioned in a recursive manner [S1, unofficial].
    *   The number of bisection levels required is $\lceil \log_2 K \rceil$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multilevel_hypergraph_partition(H0, K):
    """
    H0: Initial hypergraph (V0, N0)
    K: Number of desired parts
    Returns: Partition list of sets
    """
    # 1. Coarsening
    levels = [H0]
    current_H = H0
    while len(current_H.vertices) > TARGET_COARSE_SIZE:
        # Choose clustering method (e.g., matching-based)
        next_H = coarsen(current_H, method='matching')
        levels.append(next_H)
        current_H = next_H
    
    # 2. Initial Partitioning on coarsest level
    # Partition into 2 parts first
    coarsest_H = levels[-1]
    partition_2 = greedy_hypergraph_growing(coarsest_H)
    
    # 3. Uncoarsening and Refinement
    # Project partition_2 back up the levels
    current_partition = partition_2
    for i in range(len(levels) - 2, -1, -1):
        finer_H = levels[i]
        # Project: assign original vertices to the part of their merged multinode
        projected_partition = project(current_partition, finer_H)
        # Refine using FM/KL-like heuristic
        refined_partition = refine(projected_partition, finer_H)
        current_partition = refined_partition
        
    # 4. Recursive Bisection for K parts
    # If K > 2, recursively bisect the resulting parts
    if K > 2:
        final_parts = recursive_bisect(current_partition, K)
    else:
        final_parts = current_partition
        
    return final_parts

def coarsen(H, method):
    """
    Clusters vertices of H into multinodes to create a coarser hypergraph.
    """
    new_vertices = []
    new_edges = []
    matched = set()
    
    if method == 'matching':
        # Visit vertices in random order
        for v in random_order(H.vertices):
            if v not in matched:
                # Find an unmatched neighbor or arbitrary unmatched vertex
                u = find_unmatched_neighbor(v, H) or find_any_unmatched(H)
                if u:
                    # Create multinode {v, u}
                    multinode = {v, u}
                    new_vertices.append(multinode)
                    matched.add(v)
                    matched.add(u)
                else:
                    new_vertices.append({v})
                    matched.add(v)
    elif method == 'agglomerative':
        # Allow multiple vertices per multinode
        # Implementation details depend on specific metric
        pass
        
    # Construct new hypergraph from multinodes
    # Edges are merged if they connect vertices in the same multinodes
    return Hypergraph(new_vertices, new_edges)

def greedy_hypergraph_growing(H):
    """
    Partitions H into 2 parts using greedy growth.
    """
    seed = random_vertex(H.vertices)
    cluster = {seed}
    remaining = H.vertices - cluster
    
    # Calculate gains for vertices in remaining
    gains = calculate_gains(remaining, cluster, H)
    
    while remaining and not load_balance_satisfied(cluster, remaining):
        # Pick vertex with highest gain
        best_v = argmax(gains)
        cluster.add(best_v)
        remaining.remove(best_v)
        # Update gains
        gains = calculate_gains(remaining, cluster, H)
        
    return [cluster, remaining]

def refine(partition, H):
    """
    Improves partition quality using FM/KL-like moves.
    """
    # Iterate over vertices, try moving to neighbor parts
    # Accept move if cut size decreases and load constraint holds
    # Repeat until no improvement
    pass

def recursive_bisect(partition, K):
    """
    Recursively bisects partition into K parts.
    """
    if K == 1:
        return [partition]
    
    # Bisect current partition into two
    parts = bisect(partition)
    
    # Distribute K parts between the two halves
    k1 = K // 2
    k2 = K - k1
    
    result1 = recursive_bisect(parts[0], k1)
    result2 = recursive_bisect(parts[1], k2)
    
    return result1 + result2
```

## Subtopics

### Greedy Hypergraph Growing
An initial partitioning algorithm used by PaToH that grows a cluster around a randomly selected vertex to bisect the coarsest hypergraph [S1, unofficial]. Vertices are moved into the cluster according to their gains, starting from the vertex with the highest gain [S1, unofficial]. Cluster growth stops when a fixed load balance criteria between the two parts is satisfied [S1, unofficial]. Vertices in the cluster are assigned to the first part, and remaining vertices to the second part [S1, unofficial].

### Hypergraph Coarsening
The process of clustering vertices together in the same multinode to create a coarser hypergraph [S1, unofficial]. It employs two main techniques: Matching-based clustering, where vertices are visited in random or predetermined order and grouped with an unmatched vertex, marking them as matched; and Agglomerative clustering, which allows more than two vertices to involve in the same multinode and allows a single vertex to join a multinode [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly faster than direct partitioning methods due to the reduced size of the hypergraph during initial partitioning. The coarsening and uncoarsening phases are linear or near-linear with respect to the number of vertices and hyperedges [S1, unofficial].
*   **Memory Complexity:** Requires storage for multiple levels of hypergraphs during coarsening. However, the coarsest levels require minimal memory.
*   **Performance:** Achieves high-quality partitions with low cut sizes. The recursive bisection approach requires $\lceil \log_2 K \rceil$ bisection levels to partition into $K$ parts [S1, unofficial].
*   **Strengths:** Scalable to large hypergraphs; effective load balancing; robust to changes in visualization parameters by enabling efficient remapping [S1, unofficial].
*   **Limitations:** Quality depends on the coarsening and refinement heuristics; recursive bisection may not be optimal for all $K$ values compared to direct $K$-way methods [S1, unofficial].

## Background
The multilevel hypergraph partitioning paradigm was developed to address the computational complexity of hypergraph partitioning, which is NP-hard [S1, unofficial]. It is widely used in parallel computing, particularly in image-space parallel volume rendering, to manage dynamic load balancing and communication minimization when visualization parameters change [S1, unofficial]. Tools like PaToH and hMETIS utilize this paradigm [S1, unofficial]. The approach builds on the Hypergraph Partitioning Remapping Model, which establishes interaction between object-space and image-space to optimize remapping [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
