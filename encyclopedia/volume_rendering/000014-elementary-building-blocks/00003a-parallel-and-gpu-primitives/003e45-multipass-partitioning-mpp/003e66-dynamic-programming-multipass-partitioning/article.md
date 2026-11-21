# Dynamic Programming Multipass Partitioning


## Overview
Dynamic Programming Multipass Partitioning (DPMPP) is a method for solving the resource-constrained GPU shader execution problem by partitioning a shader program's dependency graph (DAG) into multiple passes [S1, unofficial]. It determines a globally optimal schedule of passes that minimizes total runtime while respecting hardware resource limits and precedence constraints [S1, unofficial]. Unlike greedy approximations such as the MIO algorithm, which yields locally optimal but potentially suboptimal global solutions [S1, unofficial], DPMPP guarantees optimality by observing the principle of optimality [S1, unofficial]. It builds on the general Multipass Partitioning (MPP) framework [9704-multipass-partitioning-mpp.md].

## How it works
**Inputs:** A DAG representing the shader program's dependency graph, where nodes are operations and edges are data dependencies. A set of available hardware configurations (defining resource constraints such as memory or register limits).

**Outputs:** A sequence of passes (partitions) assigning each operation to a specific pass index, such that no pass exceeds resource constraints and all dependencies are respected, minimizing the total cost (runtime).

**Data Structures:**
The algorithm operates on a set of transitions $T$. Each transition $t$ is a tuple with four fields [S1, unofficial]:
1.  $t.\text{precondition}$: The machine state before the operation.
2.  $t.\text{postcondition}$: The machine state after the operation. Defined as the state resulting from applying $t.\text{operation}$ to $t.\text{precondition}$ [S1, unofficial].
3.  $t.\text{operation}$: The specific shader operation or node from the DAG.
4.  $t.\text{cost}$: The computational cost (e.g., time) associated with the operation.

**Algorithm Steps:**
The algorithm proceeds backwards from the last stage (leaf nodes or final output) to the first stage (root nodes or input) [S1, unofficial].

1.  **Initialization:** For the final stage, initialize the set of transitions $T_{final}$ with all possible terminal states of the computation.
2.  **Backward Iteration:** For each stage $i$ from $N-1$ down to $0$:
    a.  Initialize the set of candidate transitions $T_i$ for the current stage.
    b.  For each transition $t \in T_i$, evaluate its validity against resource constraints.
    c.  Apply the **principle of optimal substructure**: Determine which transitions in $T_i$ can lead to an optimal solution for the subsequent (earlier) stage $i-1$. A transition is kept only if it is part of a globally optimal path [S1, unofficial].
    d.  The determined optimal transitions from stage $i$ become the input (preconditions) for the transitions in stage $i-1$.
3.  **Termination:** When the first stage is reached, the remaining transition(s) in $T_0$ constitute the globally optimal schedule [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DPMPP(DAG, resource_constraints):
    # DAG is a list of stages, where each stage is a list of operations
    # resource_constraints is a function check(state) -> bool
    
    # Initialize the set of transitions for the last stage
    # T[k] stores the set of optimal transitions for stage k
    num_stages = len(DAG)
    T = [set() for _ in range(num_stages)]
    
    # Base case: Last stage
    # Assuming the last stage has operations that produce the final output
    # We initialize T with transitions that represent valid final states
    last_stage_ops = DAG[-1]
    for op in last_stage_ops:
        t = Transition(
            precondition=initial_state, # [G] Initial machine state
            postcondition=apply(op, initial_state),
            operation=op,
            cost=op.cost
        )
        if check_resource_constraints(t.postcondition, resource_constraints):
            T[-1].add(t)
            
    # Backward iteration from second-to-last stage to first
    for k in range(num_stages - 2, -1, -1):
        current_stage_ops = DAG[k]
        next_stage_transitions = T[k+1]
        
        for op in current_stage_ops:
            # Generate potential transitions for this operation
            # In a full DAG, this involves matching preconditions from previous stages
            # Here we assume a simplified linear stage structure for demonstration
            # [G] In a general DAG, predecessors must be matched to successors
            for prev_t in T[k+1]: # This logic assumes a specific DAG structure
                # [G] Correct logic: For each op, find all valid predecessors
                # and combine with optimal successors
                pass 
            
        # The provided source describes the principle:
        # "For each stage, the principle of optimal substructure is used to determine 
        # the transitions which may be optimal. These are used as input to the next 
        # subsequent (earlier) stage."
        
        # Simplified implementation of the backward pass principle:
        # We filter T[k] based on whether they can lead to an optimal T[k+1]
        # Since the source does not provide the exact DAG traversal mechanics,
        # we implement the core DP recurrence:
        
        optimal_transitions_for_k = set()
        
        # [G] Standard DP on DAG: For each node, compute min cost to end
        # The source emphasizes the "backward" nature and "optimal substructure"
        # We assume T[k] contains transitions ending at stage k
        
        # Re-interpreting the source's "backward" description:
        # The source says: "proceeds backwards from the last stage to the first"
        # and "determined optimal transitions are used as input to the next subsequent (earlier) stage"
        
        # Let's implement the filtering of optimal paths
        for t in T[k+1]:
            # Find operations in stage k that can precede t
            # [G] This requires the DAG structure to be known
            # We assume a helper function get_predecessors(t) exists
            predecessors = get_predecessors(t, DAG, k)
            for pred_op in predecessors:
                pred_t = Transition(
                    precondition=..., # Derived from earlier stages
                    postcondition=apply(pred_op, ...),
                    operation=pred_op,
                    cost=pred_op.cost + t.cost # Accumulate cost
                )
                if check_resource_constraints(pred_t.postcondition, resource_constraints):
                    optimal_transitions_for_k.add(pred_t)
        
        T[k] = optimal_transitions_for_k

    # The result is in T[0]
    return T[0]
```
*Note: The pseudocode above is a structural interpretation. The source [S1, unofficial] explicitly states the algorithm proceeds backwards and uses optimal substructure to filter transitions. The exact DAG traversal (handling branching/merging) is not detailed in the provided notes, so the core DP logic of backward filtering is preserved.*

**Parameters:**
-   **Resource Constraints:** Memory, register file size, or shared memory limits per pass [S1, unofficial].
-   **Cost Function:** Typically total runtime or compute time [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Intractability:** The theoretical time complexity is intractable for large graphs [S1, unofficial].
-   **Precision:** Not given in the sources.
-   **Boundaries:** The algorithm assumes a well-defined DAG with clear stages [S1, unofficial].

## Subtopics

### Brick grouping DP
A dynamic programming solution to the Load-Balanced Network Distribution (LBND) problem [S1, unofficial]. It starts with a directed acyclic graph (DAG) of segmented and cropped bricks, based on the MPP solution by Heirich [S1, unofficial]. It outputs a render-node assignment that minimizes total runtime cost, respects physical resource constraints, and observes precedence order for image composition [S1, unofficial]. The depth of the search tree is limited to prevent intractability, which limits the size of data bricks relative to the whole dataset [S1, unofficial]. The partition is optimized with respect to a single viewpoint [S1, unofficial]. Inputs include the scene with volumetric data and distributed system configuration (number of render-nodes, rendering/local composition costs, network transfer costs, memory capacity) [S1, unofficial].

### Moving walls DP
A dynamic programming solution to a relaxed version of the LBND problem [S1, unofficial]. It applies dynamic programming to slab-projection slices [S1, unofficial]. The output is a flex-block partition, which is a view-independent partition of cells containing a combination of empty space and cropped subvolumes [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Theoretically intractable, but experimentally achieves an average time complexity of $O(n^{1.14966})$ [S1, unofficial].
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Achieves globally optimal solutions, outperforming greedy algorithms like MIO which have $O(n \log n)$ complexity but may be suboptimal globally [S1, unofficial].
-   **Strengths:** Guarantees global optimality; respects resource constraints and precedence.
-   **Limitations:** Theoretical intractability; limited search tree depth in variants like Brick grouping DP restricts brick size [S1, unofficial].
-   **When to use:** When global optimality is critical and the problem size is manageable or can be segmented (e.g., distributed volume visualization) [S1, unofficial].
-   **When not:** For very large, unsegmented DAGs where theoretical intractability becomes a practical barrier [S1, unofficial].

## Background
DPMPP was introduced as a dynamic programming solution to the Multipass Partitioning (MPP) problem [S1, unofficial]. MPP arises when a computation targeted for the GPU exceeds resource constraints, requiring division into multiple passes to minimize total compute time [S1, unofficial]. The MPP problem was introduced by Heirich [S1, unofficial]. DPMPP is applied in the context of distributed volume visualization and ray tracing, specifically for load-balanced network distribution (LBND) of volumetric data [S1, unofficial]. It is used to optimize rendering schedules on clusters, considering network transfer costs and node memory capacities [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
