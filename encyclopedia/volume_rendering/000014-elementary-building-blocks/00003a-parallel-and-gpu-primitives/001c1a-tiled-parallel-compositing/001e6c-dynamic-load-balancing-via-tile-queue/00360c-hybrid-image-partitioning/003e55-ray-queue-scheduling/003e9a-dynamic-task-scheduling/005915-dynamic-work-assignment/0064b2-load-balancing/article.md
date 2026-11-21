# Load Balancing

Also called: Dynamic Work Assignment, Online Linear Sum Assignment

## Overview
Load balancing is a technique applied in both point assignment and image assignment to balance workload across parallel processing units [S1, unofficial]. It serves as a refinement step within a locality-aware distribution framework, aiming to mitigate the performance degradation caused by uneven data distribution after initial partitioning. While static or hierarchical assignment provides the bulk of the performance gain, load balancing offers a modest but consistent improvement by fine-tuning assignments to reduce communication overhead [S1, unofficial].

## How it works
Inputs: A set of tasks (points or image patches) and a set of processing units (GPUs/nodes), along with a cost matrix representing the communication or computation cost of assigning each task to each unit.
Outputs: A one-to-one or many-to-one assignment mapping that minimizes total cost.

The method typically operates in two phases:
1. Offline Partitioning: The dataset is initially divided using a graph partitioning algorithm (e.g., METIS) to ensure spatial locality [S1, unofficial].
2. Online Scheduling: An online linear sum assignment algorithm is applied to the partitions to further balance the load [S1, unofficial].

**Data Structures:**
- Cost Matrix $C$: A 2D array where $C_{ij}$ represents the cost of assigning task $i$ to unit $j$.
- Assignment Vector $A$: A 1D array where $A_i = j$ indicates task $i$ is assigned to unit $j$.

**Algorithm Steps:**
1. Compute the cost matrix $C$ based on current workload and communication distances.
2. Solve the linear sum assignment problem to find the permutation $\sigma$ that minimizes $\sum_i C_{i, \sigma(i)}$.
3. Update the assignment vector $A$ based on $\sigma$.

**Parameters:**
- Solver tolerance: Not given in the sources.
- Batch size for online scheduling: Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_load_balancing(cost_matrix):
    """
    Solves the linear sum assignment problem for load balancing.
    
    Args:
        cost_matrix (List[List[float]]): 2D array of costs.
        
    Returns:
        List[int]: Assignment vector where index is task ID and value is unit ID.
    """
    # Use a standard linear sum assignment solver (e.g., scipy.optimize.linear_sum_assignment)
    # The solver returns row indices and column indices of the optimal assignment
    row_indices, col_indices = linear_sum_assignment(cost_matrix)
    
    # Construct the assignment vector
    # Initialize with -1 or a default assignment if not all tasks are matched
    assignment = [-1] * len(cost_matrix)
    for r, c in zip(row_indices, col_indices):
        assignment[r] = c
        
    return assignment
```

**Edge Cases and Pitfalls:**
- Communication Dominance: The impact of load balancing is modest because communication time often dominates the overall runtime [S1, unofficial].
- Overhead: Online scheduling must be fast; however, studies show it introduces no significant overhead [S1, unofficial].

## Subtopics

### Online Linear Sum Assignment
An online scheduling technique paired with offline METIS partitioning to assign points/images, reducing communication overhead [S1, unofficial]. It is chosen when fine-grained balancing is needed after coarse spatial partitioning. It introduces no significant overhead, as indicated by end-to-end speedups across scenes [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- The complexity depends on the solver used for the linear sum assignment. For $N$ tasks and $M$ units, standard algorithms are $O(N^2 M)$ or $O(N^3)$ for dense matrices [G].
- METIS partitioning time varies by scene complexity, ranging from 1.9s to 31.0s for large scenes [S1, unofficial].

**Measured Performance:**
- Removing load balancing causes performance reductions ranging from 1% to 14% [S1, unofficial].
- In contrast, removing hierarchical assignment ("w/o Hier.") results in 16% to 35% slower performance, and removing patching ("w/o Patch") results in 10% to 52% slower performance [S1, unofficial].
- The modest effect of load balancing stems from the fact that communication time dominates the overall runtime in the tested setting [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Reduces communication overhead; complements hierarchical assignment; no significant scheduling overhead.
- Limitations: Modest impact compared to hierarchical partitioning and patching; less effective when communication is not the bottleneck.

**When to Use:**
- Use when the system uses a locality-aware distribution framework with offline partitioning and online scheduling [S1, unofficial].
- Particularly useful for high-resolution scenes where patching allows flexible assignment [S1, unofficial].

## Background
Origin: The technique is part of a system for scaling point-based differentiable rendering for large-scale reconstruction [S1, unofficial]. It builds on prior work in point cloud partitioning, including manual heuristics, graphics data structures (KD-Tree), and clustering techniques (k-means) [S1, unofficial]. The specific implementation combines offline METIS bipartite graph partitioning with online linear sum assignment to significantly reduce communication overhead compared to random distribution strategies [S1, unofficial].

## Sources
[S1] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
