# Dynamic load balancing using kd-trees

Also called: kd-tree based load balancing, hierarchical load balancing

## Overview
This method uses a kd-tree to dynamically redistribute rendering work across distributed processors by encoding the spatial decomposition of the volume data. It solves the problem of load imbalance in distributed volume rendering by allowing the system to recalculate and reassign partitions based on estimated costs or occlusion statistics for subsequent frames [S1, unofficial]. It sets itself apart from static decomposition methods by enabling runtime adaptation, and from central queue approaches by maintaining a hierarchical spatial structure rather than a flat list of tasks [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset (partitioned into bricks or chunks).
*   Current load statistics (e.g., rendering cost estimates, occlusion information) [S1, unofficial].
*   Number of processors $P$.

**Outputs:**
*   Assignment of work partitions to processors.
*   Updated kd-tree structure reflecting the new distribution.

**Data Structures:**
*   **kd-tree:** A binary tree where each node represents a region of the data space.
    *   **Inner Nodes:** Store load statistics (e.g., total estimated cost of children) [S1, unofficial].
    *   **Leaf Nodes:** Store the effective partitions of work (bricks/chunks) assigned to specific processors or groups [S1, unofficial].
    *   **Splitting:** At each level, the data space is split alternating along orthogonal axes (x, y, z) [S1, unofficial].

**Steps:**
1.  **Estimate Costs:** Calculate rendering costs or occlusion information for current partitions [S1, unofficial].
2.  **Build/Update Tree:** Construct or update the kd-tree. Inner nodes aggregate load statistics from their children. Leaf nodes hold the actual work units [S1, unofficial].
3.  **Traverse for Balance:** Traverse the kd-tree in a specific order to balance partitions. The traversal strategy determines how work is redistributed among processors to minimize idle time [S1, unofficial].
4.  **Assign Work:** Map the balanced leaf nodes to available processors.
5.  **Render:** Processors render their assigned partitions.
6.  **Compositing (Optional Use):** The kd-tree can also be used to determine the compositing order of partial rays. **Pitfall:** When used for compositing order determination, replication of boundary data is necessary to ensure correct blending across partition edges [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_load_balance(volume_data, processors, current_load_stats):
    # 1. Build kd-tree from volume data partitions
    # Split alternating along axes
    kd_tree = build_kd_tree(volume_data.partitions, axis_order=['x', 'y', 'z'])
    
    # 2. Populate inner nodes with load statistics
    # Leaf nodes contain effective partitions
    for node in kd_tree.all_nodes():
        if node.is_leaf():
            node.load = current_load_stats.get(node.partition_id, 0)
        else:
            # Aggregate statistics from children
            node.load = sum(child.load for child in node.children)
            
    # 3. Traverse tree to balance partitions
    # Specific traversal order depends on balancing strategy
    balanced_assignments = traverse_and_balance(kd_tree, processors)
    
    # 4. Apply assignments
    for processor, partitions in balanced_assignments.items():
        processor.assign(partitions)
        
    return balanced_assignments

def determine_compositing_order(kd_tree, ray):
    # Use kd-tree to find intersection order
    # Requires boundary data replication
    intersections = []
    for leaf in kd_tree.traverse_for_ray(ray):
        intersections.append(leaf.partition)
    return sort_front_to_back(intersections)
```

## Subtopics

### [Incremental Load Balancing with Brick Caching](9862-incremental-load-balancing-with-brick-caching.md)
A dynamic load balancing technique for distributed volume rendering that incrementally moves bricks between cluster nodes to correct imbalance, using a caching mechanism to reduce transfer costs when reversing moves. It sets itself apart from full-tree balancing methods by processing only one node per frame to respect limited computation time, and from naive incremental methods by employing a brick cache to mitigate oscillation and stagnation caused by high network transfer costs.

## Operating characteristics
*   **Complexity:** Building the kd-tree is $O(N \log N)$ for $N$ partitions. Traversal and rebalancing depend on the specific strategy but are generally faster than global optimization.
*   **Strengths:** Efficiently handles spatial locality; supports empty-space skipping [S1, unofficial]; allows for dynamic adaptation to changing scene complexity or occlusion.
*   **Limitations:** Requires communication of load statistics; boundary data replication adds memory overhead when used for compositing order [S1, unofficial]; rebalancing overhead must be justified by significant load imbalance.
*   **When to use:** Scenes with varying density or occlusion where static decomposition leads to poor load balance.

## Background
This approach leverages hierarchical data structures like kd-trees to manage load statistics in distributed volume rendering, as described in various research contexts [S1, unofficial]. It addresses the limitations of static partitioning by enabling runtime redistribution based on estimated costs or occlusion information [S1, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
