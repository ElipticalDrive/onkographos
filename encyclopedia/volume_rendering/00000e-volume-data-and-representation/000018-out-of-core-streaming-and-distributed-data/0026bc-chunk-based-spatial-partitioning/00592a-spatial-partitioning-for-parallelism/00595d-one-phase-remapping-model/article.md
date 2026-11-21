# One-Phase Remapping Model

Also called: OS Decomposition Remapping, Remapping Graph Partitioning

## Overview
The One-Phase Remapping Model is a method for solving the remapping problem in parallel direct volume rendering (DVR) by constructing a single augmented graph (the remapping graph) and performing a single K-way partition, rather than separating partitioning and assignment into two phases [S1, unofficial]. It addresses the limitation of two-phase approaches where decisions made during the initial partitioning phase restrict the quality of the final task-to-processor assignment [S1, unofficial]. By integrating the current mapping and migration costs into the graph structure, it allows a standard graph partitioner to simultaneously optimize load balance and communication/migration overhead [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:**
    *   Computational graph $Q = (N, E, w)$ representing the visualization instance [S1, unofficial].
    *   Current cell-to-processor mapping $\mathcal{M}$ [S1, unofficial].
    *   Cell-migration cost function $\mathcal{T}$ [S1, unofficial].
    *   Number of processors $K$ [S1, unofficial].
*   **Output:**
    *   Remapped assignment $\mathcal{M}'$ where each task node is assigned to a processor index $k \in \{1, \dots, K\}$ [S1, unofficial].

**Data Structures**
The core data structure is the **Remapping Graph** $\hat{Q} = (\hat{N}, \hat{E}, \hat{w})$.

*   **Node Set $\hat{N}$:**
    *   Contains task nodes $\hat{N}_t$ and processor nodes $\hat{N}_p$.
    *   $\hat{N}_t = N$ (the original task nodes from $Q$) [S1, unofficial].
    *   $\hat{N}_p = \{p_1, p_2, \dots, p_K\}$ (one node per processor) [S1, unofficial].
    *   Total nodes: $|N| + K$.
*   **Edge Set $\hat{E}$:**
    *   Contains task-to-task edges $\hat{E}_{tt}$ and processor-to-task edges $\hat{E}_{pt}$ [S1, unofficial].
    *   $\hat{E}_{tt} = E$ (original edges between tasks) [S1, unofficial].
    *   $\hat{E}_{pt} = \{(n_i, p_k) : n_i \in \hat{N}_t, p_k \in \hat{N}_p \text{ and } \mathcal{M}(n_i) = k\}$ [S1, unofficial].
    *   Each task node $n_i$ has exactly one outgoing edge to the processor node $p_k$ corresponding to its current assignment $\mathcal{M}(n_i)$ [S1, unofficial].
*   **Weights $\hat{w}$:**
    *   **Node Weights $\bar{w}$:**
        *   $\bar{w}(n_i) = w(n_i)$ for all $n_i \in \hat{N}_t$ [S1, unofficial].
        *   $\bar{w}(p_k) = 0$ for all $p_k \in \hat{N}_p$ [S1, unofficial].
    *   **Edge Weights $\tilde{w}$:**
        *   $\tilde{w}(n_i, n_j) = w(n_i, n_j)$ for all $(n_i, n_j) \in \hat{E}_{tt}$ [S1, unofficial].
        *   $\tilde{w}(n_i, p_k) = \mathcal{T}(n_i)$ for all $(n_i, p_k) \in \hat{E}_{pt}$ [S1, unofficial].

**Steps**
1.  **Construct the Remapping Graph:**
    *   Initialize $\hat{N}_t$ with the nodes of the computational graph $Q$.
    *   Create $K$ processor nodes $p_1, \dots, p_K$ and add them to $\hat{N}_p$.
    *   Copy all edges from $Q$ to $\hat{E}_{tt}$ and assign their original weights.
    *   For each task node $n_i \in \hat{N}_t$:
        *   Identify its current processor $k = \mathcal{M}(n_i)$.
        *   Add edge $(n_i, p_k)$ to $\hat{E}_{pt}$.
        *   Set weight $\tilde{w}(n_i, p_k) = \mathcal{T}(n_i)$.
2.  **Define Constraints:**
    *   **Mapping Constraint:** A partition $\Pi = \{V_1, V_2, \dots, V_K\}$ is feasible if and only if each part $V_k$ contains exactly one processor node: $|V_k \cap \hat{N}_p| = 1$ for all $k=1, \dots, K$ [S1, unofficial].
    *   **Balance Criterion:** Maintain computational load balance. Since processor nodes have zero weight, the sum of task node weights in each part $V_k$ should be approximately equal to the total task weight divided by $K$ [S1, unofficial].
    *   **Cutsize Definition:** Minimize the sum of weights of edges crossing partition boundaries (edges $(u, v)$ where $u \in V_i, v \in V_j, i \neq j$) [S1, unofficial].
3.  **Solve K-way Partitioning:**
    *   Run a K-way graph partitioning algorithm (e.g., Kernighan-Lin based heuristic) on $\hat{Q}$ subject to the mapping constraint and balance criterion [S1, unofficial].
    *   The algorithm returns a partition $\Pi = \{V_1, \dots, V_K\}$ [S1, unofficial].
4.  **Induce Remapping:**
    *   For each part $V_k$, identify the unique processor node $p_k \in V_k$ [S1, unofficial].
    *   For each task node $n_i \in V_k$, assign $\mathcal{M}'(n_i) = k$ [S1, unofficial].

**Formulas**
*   **Node Set:** $\hat{N} = \hat{N}_t \cup \hat{N}_p$ [S1, unofficial].
*   **Edge Set:** $\hat{E} = \hat{E}_{tt} \cup \hat{E}_{pt}$ [S1, unofficial].
*   **Node Weights:**
    $$ \bar{w}(n_i) = w(n_i) \quad \forall n_i \in \hat{N}_t $$
    $$ \bar{w}(p_k) = 0 \quad \forall p_k \in \hat{N}_p $$
    [S1, unofficial]
*   **Edge Weights:**
    $$ \tilde{w}(n_i, p_k) = \mathcal{T}(n_i) \quad \forall (n_i, p_k) \in \hat{E}_{pt} $$
    $$ \tilde{w}(n_i, n_j) = w(n_i, n_j) \quad \forall (n_i, n_j) \in \hat{E}_{tt} $$
    [S1, unofficial]
*   **Mapping Constraint:**
    $$ |V_k \cap \hat{N}_p| = 1 \quad \text{for} \quad k = 1, 2, \dots, K $$
    [S1, unofficial]

**Parameters**
*   **Cell-migration cost $\mathcal{T}(n_i)$:** Constructed by scaling based on communication volume overhead. Specifically, $\mathcal{T}(n_i)$ is set equal to the ratio of the size of the data associated with cell $c_i$ to the size of an individual raysegment information [S1, unofficial].
*   **Number of partitions $K$:** Equal to the number of processors [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The partitioning heuristic must strictly enforce the mapping constraint (exactly one p-node per part). Failure to do so results in an invalid remapping [S1, unofficial].
*   **Degenerate Input:** If the computational graph $Q$ has no edges, the remapping is determined solely by the migration costs and balance constraints [G].
*   **Boundary Artifacts:** This method handles logical task remapping. Physical boundary artifacts (e.g., black stripes) are handled by the parent method's overlap strategy, not by this remapping model itself [1093-mpi-based-volume-partitioning.md].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def one_phase_remapping(computational_graph, current_mapping, num_processors):
    """
    computational_graph: dict with keys 'nodes' (list of node_ids), 'edges' (list of (u, v, weight)), 'weights' (dict node_id -> weight)
    current_mapping: dict node_id -> processor_id (0 to K-1)
    num_processors: int K
    Returns: new_mapping dict node_id -> processor_id
    """
    K = num_processors
    task_nodes = computational_graph['nodes']
    task_weights = computational_graph['weights']
    tt_edges = computational_graph['edges']
    
    # Step 1: Construct Remapping Graph
    # Nodes: task_nodes + processor_nodes (p_0 to p_{K-1})
    all_nodes = task_nodes + [f"p_{k}" for k in range(K)]
    
    # Weights: task nodes keep original weight, processor nodes have 0 weight
    node_weights = {n: task_weights[n] for n in task_nodes}
    for k in range(K):
        node_weights[f"p_{k}"] = 0
        
    # Edges: tt_edges + pt_edges
    # tt_edges keep original weight
    pt_edges = []
    for n in task_nodes:
        p_id = current_mapping[n]
        p_node = f"p_{p_id}"
        # Migration cost T(n_i)
        # Not given in sources: specific formula for T(n_i) beyond "ratio of data size to raysegment size"
        # We assume a function get_migration_cost(n) is available or precomputed
        migration_cost = get_migration_cost(n) 
        pt_edges.append((n, p_node, migration_cost))
        
    all_edges = tt_edges + pt_edges
    
    # Step 2: Solve K-way Partitioning
    # Constraint: Each part must contain exactly one processor node.
    # Balance: Sum of node_weights in each part should be equal.
    # Objective: Minimize cutsize (sum of weights of edges crossing parts).
    
    # Note: Standard graph partitioners (like METIS) do not natively support 
    # the "exactly one specific node per part" constraint. 
    # Implementation requires a custom heuristic or modification of a standard K-way partitioner.
    # The sources mention KL-based heuristics are used [S1, unofficial].
    
    partition = solve_k_way_partition_with_constraints(
        nodes=all_nodes,
        edges=all_edges,
        node_weights=node_weights,
        num_parts=K,
        constraint_type="one_processor_per_part"
    )
    
    # Step 3: Induce Remapping
    # partition is a list of K sets of nodes.
    # For each part i, find the processor node p_i.
    # Assign all task nodes in that part to processor i.
    
    new_mapping = {}
    for part_index, part_nodes in enumerate(partition):
        # Find the processor node in this part
        processor_node = None
        for node in part_nodes:
            if node.startswith("p_"):
                processor_node = node
                break
        
        if processor_node is None:
            raise ValueError("Mapping constraint violated: no processor node in part")
            
        # Assign all task nodes in this part to the corresponding processor
        for node in part_nodes:
            if node in task_nodes:
                new_mapping[node] = part_index
                
    return new_mapping

def get_migration_cost(task_node):
    """
    Placeholder for the migration cost calculation.
    Sources state: T(n_i) = ratio of size of data associated with cell c_i 
    to the size of an individual raysegment information.
    """
    # Data size for task_node
    data_size = get_data_size(task_node)
    # Raysegment size (constant)
    raysegment_size = get_raysegment_size()
    return data_size / raysegment_size
```

## Subtopics

### MPI-based Volume Partitioning
Uses message passing for distributed memory systems. Focuses on inter-node communication. [1093-mpi-based-volume-partitioning.md]

### Object-space decomposition with local clustering
Groups spatially adjacent chunks to minimize communication. [4156-object-space-decomposition-with-local-clustering.md]

### ShardedVolume
A specific data structure for sharded volume data. [12593-shardedvolume.md]

## Operating characteristics
*   **Time Complexity:** Dominated by the K-way graph partitioning step. Using KL-based heuristics, it is generally efficient, but the augmented graph size is $|N| + K$ nodes and $|E| + |N|$ edges [S1, unofficial].
*   **Memory Complexity:** Requires storing the augmented graph $\hat{Q}$, which adds $K$ nodes and $|N|$ edges to the original computational graph [S1, unofficial].
*   **Strengths:** Avoids the suboptimal solutions of two-phase approaches by jointly optimizing partitioning and assignment [S1, unofficial]. Reduces remapping overhead by minimizing migration costs and inter-processor communication simultaneously [S1, unofficial].
*   **Limitations:** Requires a graph partitioning algorithm that can handle the specific mapping constraint (one processor node per part), which is not standard in all libraries [S1, unofficial]. The quality of the remapping depends on the quality of the graph partitioning heuristic [S1, unofficial].
*   **When to use:** When remapping tasks in parallel DVR to balance load and minimize communication/migration costs in a single step [S1, unofficial].
*   **When not to use:** When the computational graph is too large for graph partitioning heuristics to be efficient, or when the mapping constraint cannot be easily enforced [G].

## Background
The One-Phase Remapping Model was proposed in the context of parallel direct volume rendering of unstructured grids based on object-space decomposition [S1, unofficial]. It addresses the limitations of two-phase approaches where partitioning and assignment are solved separately, which can restrict the quality of the remapping [S1, unofficial]. The method uses graph partitioning heuristics, such as Kernighan-Lin (KL), which are widely used for their short run-times and good quality results [S1, unofficial].

## Sources
[S1] Parallel direct volume rendering of unstructured grids based on object-space decomposition, http://hdl.handle.net/11693/17897, fetched 2026-10-26 (unofficial)
