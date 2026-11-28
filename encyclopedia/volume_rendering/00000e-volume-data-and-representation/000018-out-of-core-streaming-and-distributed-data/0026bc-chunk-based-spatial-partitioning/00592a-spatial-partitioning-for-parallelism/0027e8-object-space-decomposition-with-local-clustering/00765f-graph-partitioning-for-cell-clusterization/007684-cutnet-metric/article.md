# Cutnet Metric
Also called: Cutnet

## Overview
The Cutnet metric is a partitioning objective function used in graph partitioning to balance computational load and minimize inter-partition communication. It defines the cost of a partition as the sum of the weights of all external nets [S1, unofficial]. It serves as a primary alternative to the Connectivity Metric, prioritizing the minimization of total cut weight over the number of cut nets [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A hypergraph $H=(V, E)$ representing the data structure, where $V$ is the set of vertices (cells/tasks) and $E$ is the set of nets (connections). Each net $n_i$ has an associated weight $c_i$. A partitioning solution $N$ assigns vertices to partitions.
*   **Output**: A scalar cost value $X(N)$ representing the quality of the partition.

**Data Structures**
*   **Net List**: A collection of nets $N_C(V)$, where each net $n_i$ contains:
    *   $c_i$: The weight of the net (typically 1 or proportional to data volume).
    *   Connectivity: The set of vertices connected by the net.

**Formulas**
The cost $X(N)$ is calculated as:
$$ X(N) = \sum_{n_i \in N_C(V)} c_i $$
Where:
*   $N_C(V)$ is the set of external nets (nets that connect vertices in more than one partition).
*   $c_i$ is the weight of net $n_i$ [S1, unofficial].

**Steps**
1.  Identify all nets $n_i$ in the hypergraph.
2.  For each net, determine if it is an "external net" (i.e., its connected vertices are distributed across multiple partitions).
3.  If a net is external, add its weight $c_i$ to the total cost.
4.  Sum these weights to produce the final cost $X(N)$ [S1, unofficial].

**Parameters**
*   $c_i$: Net weights. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Empty Nets**: Nets with no vertices or zero weight contribute 0 to the cost.
*   **Internal Nets**: Nets entirely contained within a single partition have a cost contribution of 0.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_cutnet_cost(partition_assignment, nets):
    """
    Calculate the Cutnet metric cost for a given partitioning.
    
    Args:
        partition_assignment: A dictionary mapping vertex_id -> partition_id.
        nets: A list of net objects. Each net object has:
              - weight: float (c_i)
              - vertices: list of vertex_ids connected by this net
    
    Returns:
        float: The total cutnet cost.
    """
    total_cost = 0.0
    
    for net in nets:
        # Determine if this net is external
        # An external net connects vertices in more than one partition
        partitions_involved = set()
        for vertex_id in net.vertices:
            if vertex_id in partition_assignment:
                partitions_involved.add(partition_assignment[vertex_id])
        
        # If the net spans multiple partitions, it is an external net
        if len(partitions_involved) > 1:
            total_cost += net.weight
            
    return total_cost
```

## Subtopics

### Connectivity Metric
A partitioning objective function where each net contributes $c_i \cdot (A_i - 1)$ to the cost, with $A_i$ being the connectivity of the net (the number of partitions the net spans) [S1, unofficial]. It penalizes nets that span many partitions more heavily than the Cutnet metric, which treats all external nets equally regardless of how many partitions they span.

## Operating characteristics
*   **Complexity**: Linear with respect to the number of external nets.
*   **Strengths**: Simple to compute; directly minimizes the total weight of data transferred across partition boundaries [S1, unofficial].
*   **Limitations**: Does not account for the number of partitions a net spans, potentially leading to suboptimal communication patterns if nets are spread across many partitions [S1, unofficial].
*   **When to use**: When the primary goal is to minimize the total volume of data crossing partition boundaries.

## Background
The Cutnet metric is a standard objective function in hypergraph partitioning literature, widely used in VLSI design and parallel computing [S1, unofficial]. It is classified under move-based approaches in partitioning methods, which are noted for their success in speed and solution quality [S1, unofficial].

## Sources
[S1] A hypergraph-partitioning based remapping model for image-space parallel volume rendering, http://hdl.handle.net/11693/18191, fetched 2026-10-26 (unofficial)
