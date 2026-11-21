# Multipass Partitioning (MPP)

Also called: DPMPP

## Overview
Multipass Partitioning (MPP) is a technique for solving resource-constrained GPU shader execution by partitioning a shader program's dependency graph (DAG) into multiple passes, minimizing total runtime while respecting precedence and hardware limits [S1, unofficial]. It sets itself apart from single-pass approaches by allowing complex applications, such as texture-mapped volume rendering, to exceed the resource capacity of GPU hardware through strategic decomposition [S1, unofficial]. Unlike greedy scheduling methods that may yield locally optimal but globally suboptimal results, dynamic programming variants like DPMPP solve the partitioning problem using optimal substructure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A valid shader program represented as a DAG, where nodes are operations and edges are dependencies [S1, unofficial].
*   **Input:** Costs associated with each operation and the physical resource constraints of the target GPU [S1, unofficial].
*   **Output:** A schedule of DAG operations partitioned into passes that minimizes total runtime cost [S1, unofficial].

**Data Structures:**
*   **DAG:** Nodes represent operations; edges represent precedence [S1, unofficial].
*   **Transition ($t$):** Used in DPMPP, consisting of `t.precondition`, `t.postcondition`, `t.operation`, and `t.cost` [S1, unofficial]. `t.postcondition` is the machine state in `t.precondition` following application of `t.operation` [S1, unofficial].

**Algorithm Steps (DPMPP):**
The Dynamic Programming Multipass Partitioning (DPMPP) algorithm proceeds backwards from the last stage to the first [S1, unofficial].
1.  **Initialization:** Initialize a set $T$ of transitions for the final stage [S1, unofficial].
2.  **State Definition:** For each transition $t$, define `t.postcondition` as the machine state resulting from applying `t.operation` to `t.precondition` [S1, unofficial].
3.  **Backward Induction:** For each stage, apply the principle of optimal substructure to determine which transitions may be optimal [S1, unofficial].
4.  **Cost Evaluation:** Evaluate the cost function of a proposed GPU pass by generating the code for that pass [S1, unofficial].
5.  **Selection:** Choose the partitioning that minimizes the total runtime cost, observing precedence relations and ensuring no pass exceeds the physical resource constraints of the GPU [S1, unofficial].

**Alternative Algorithms:**
*   **Recursive Dominator Split (RDS):** Uses a minimum-cut approach to minimize the total number of passes [S1, unofficial]. It computes a limited search of a subregion of the solution space and chooses the solution with the least cost [S1, unofficial]. RDS considers only a single connected region of the input DAG at any time [S1, unofficial].
*   **MIO (MIO Algorithm):** Treats MPP as a job-shop scheduling problem using a greedy list-scheduling algorithm [S1, unofficial]. It provides a locally optimal but potentially globally suboptimal solution [S1, unofficial].

**Parameters:**
*   **Hardware Constraints:** Physical resource constraints of the GPU that define the maximum capacity of a single pass [S1, unofficial].
*   **Operation Costs:** Execution time or resource usage for each node in the DAG [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Multiply-Referenced Nodes:** Algorithms like RDS and MIO allow multiply-referenced nodes to be either recomputed or saved, which significantly impacts cost [S1, unofficial].
*   **Connected Regions:** RDS considers only a single connected region of the input DAG at any time, which may limit the search space [S1, unofficial].

## Subtopics

### [Dynamic Programming Multipass Partitioning](9735-dynamic-programming-multipass-partitioning.md)
A dynamic programming solution to the MPP problem that determines a globally optimal schedule for GPU shader execution by observing the principle of optimality. It sets itself apart from greedy alternatives like the MIO algorithm by guaranteeing global optimality rather than local optimality, at the cost of potentially higher theoretical complexity.

### [Megakernel Generation](11538-megakernel-generation.md)
A code generation strategy in differentiable rendering that consolidates all program instructions required to evaluate an integrand into a single, large GPU kernel. It solves the problem of excessive memory bandwidth consumption and storage overhead caused by inter-kernel communication in partitioned execution models. Unlike wavefront-based approaches that split computation into smaller kernels with intense data exchange through device memory, megakernels keep Monte Carlo integration fully contained within one kernel, storing most sample state in registers to minimize memory usage.

## Operating characteristics
*   **Time Complexity:**
    *   RDS: $O(n^3)$, where $n$ is the number of nodes in the DAG [S1, unofficial].
    *   RDS*h*: $O(n^2)$, using a heuristic to replace the subregion solution search, trading partition quality for speed [S1, unofficial].
    *   MIO: Average time $O(n \log n)$, using a greedy list-scheduling algorithm [S1, unofficial].
    *   DPMPP: Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   MPP allows complex shaders to run on resource-constrained GPUs [S1, unofficial].
    *   RDS minimizes the total number of passes [S1, unofficial].
*   **Limitations:**
    *   RDS and RDS*h* have higher computational complexity for the partitioning phase compared to greedy methods [S1, unofficial].
    *   MIO may produce suboptimal global solutions [S1, unofficial].
    *   Handling multiply-referenced nodes adds complexity (recompute vs. save) [S1, unofficial].
*   **When to Use:** When a shader program exceeds GPU resource limits and requires partitioning into multiple passes [S1, unofficial].

## Background
Multipass Partitioning was introduced to address the limitation that most complex applications, including texture-mapped volume rendering, exceed the resource capacity of even the most up-to-date GPU hardware [S1, unofficial]. The solution is to partition the problem into multiple passes [S1, unofficial]. Several solutions to the MPP problem have been proposed, including RDS by Chan et al. [14] and MIO by Riffel et al. [100] [S1, unofficial]. The DPMPP solution is based on a dynamic programming (DP) solution to the multipass partitioning problem [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
