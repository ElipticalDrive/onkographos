# Dynamic load balancing using kd-trees

Also called: kd-tree based load balancing, hierarchical load balancing

## Overview
This method uses a hierarchical kd-tree to partition volume data and track workload statistics, enabling dynamic redistribution of rendering tasks among processors to minimize the maximum execution time [S1, unofficial]. It sets itself apart from static decomposition methods by allowing load statistics stored in inner nodes to guide traversal and rebalancing strategies, and from queue-based approaches by maintaining explicit spatial hierarchy for tasks like empty-space skipping and compositing order determination [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset (often represented as uniform texture bricks), number of processors $P$, current load statistics (e.g., estimated cost or occlusion information) [S1, unofficial].
*   **Output:** Assignment of work partitions to processors, maintaining correct compositing order [S1, unofficial].

**Data Structures:**
*   **kd-tree:** A binary tree encoding the decomposition of the data space [S1, unofficial].
    *   **Inner Nodes:** Store load statistics (e.g., estimated cost, occlusion info) aggregated from children [S1, unofficial].
    *   **Leaf Nodes:** Contain effective partitions of work (subsets of bricks) [S1, unofficial].
    *   **Splitting:** At each level, data is split alternating along orthogonal axes [S1, unofficial].

**Steps:**
1.  **Tree Construction:** Build a kd-tree over the volume data space. Split data alternating along orthogonal axes at each level [S1, unofficial].
2.  **Statistics Aggregation:** Populate inner nodes with chosen load statistics (e.g., estimated rendering cost or occlusion information) [S1, unofficial].
3.  **Traversal for Balancing:** Traverse the kd-tree in specific orders to balance partitions according to the chosen strategy (e.g., greedy assignment to least-loaded nodes) [S1, unofficial].
4.  **Work Assignment:** Assign leaf node partitions to processors based on the traversal and load statistics [S1, unofficial].
5.  **Compositing Order Determination:** Use the kd-tree structure to determine the front-to-back compositing order of partial rays [S1, unofficial].
6.  **Boundary Handling:** Replicate boundary data between partitions to ensure correct compositing when using the kd-tree for order determination [S1, unofficial].

**Formulas:**
*   **Load Minimization:** The goal is to minimize the maximum workload:
    $$ T_{max} = \max_{p \in Processors} (T_{compute, p} + T_{comm, p}) $$
    Where $T_{compute}$ is rendering time and $T_{comm}$ is communication/compositing time [S1, unofficial].

**Parameters:**
*   **Splitting Axis:** Alternating orthogonal axes (x, y, z) [S1, unofficial].
*   **Load Statistic:** Estimated cost or occlusion information [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Replication:** Replication of boundary data is necessary when using kd-trees for compositing order determination to avoid artifacts [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_kd_tree(bricks, depth=0):
    """
    Recursively build a kd-tree.
    bricks: list of brick objects with spatial bounds and cost
    depth: current tree depth
    """
    if not bricks:
        return None
    
    # Determine split axis based on depth
    axis = depth % 3
    
    # Sort bricks by center coordinate along the axis
    sorted_bricks = sorted(bricks, key=lambda b: b.center[axis])
    
    # Split into two halves
    mid = len(sorted_bricks) // 2
    left_bricks = sorted_bricks[:mid]
    right_bricks = sorted_bricks[mid:]
    
    # Create node
    node = {
        'axis': axis,
        'split_value': sorted_bricks[mid-1].center[axis], # Placeholder for split plane
        'left': None,
        'right': None,
        'load_stat': 0,
        'is_leaf': False
    }
    
    if len(left_bricks) == 1 and len(right_bricks) == 1:
        # Leaf nodes contain effective partitions
        node['left'] = {'bricks': left_bricks, 'load_stat': sum(b.cost for b in left_bricks), 'is_leaf': True}
        node['right'] = {'bricks': right_bricks, 'load_stat': sum(b.cost for b in right_bricks), 'is_leaf': True}
    else:
        node['left'] = build_kd_tree(left_bricks, depth + 1)
        node['right'] = build_kd_tree(right_bricks, depth + 1)
    
    # Aggregate load statistic
    node['load_stat'] = node['left']['load_stat'] + node['right']['load_stat']
    
    return node

def balance_load(root, num_processors):
    """
    Traverse kd-tree to assign partitions to processors.
    """
    assignments = [[] for _ in range(num_processors)]
    processor_loads = [0] * num_processors
    
    def traverse(node, current_proc):
        if node['is_leaf']:
            # Assign leaf partition to processor with minimum current load
            best_proc = min(range(num_processors), key=lambda p: processor_loads[p])
            assignments[best_proc].extend(node['bricks'])
            processor_loads[best_proc] += node['load_stat']
            return
        
        # Traverse children to balance load
        # Strategy: assign subtrees to balance total load
        # Simple greedy: assign left to current, right to next? 
        # More complex strategies exist [S1, unofficial]
        traverse(node['left'], current_proc)
        traverse(node['right'], (current_proc + 1) % num_processors)

    traverse(root, 0)
    return assignments

def get_compositing_order(root, view_direction):
    """
    Determine front-to-back compositing order using kd-tree.
    """
    order = []
    
    def traverse(node):
        if node['is_leaf']:
            # Check if brick is in front of split plane relative to view
            # This requires checking spatial bounds against view ray
            # Simplified: assume leaf nodes are ordered correctly if tree is built correctly
            order.append(node['bricks'])
            return
        
        # Determine which child is closer to viewer
        # Compare split plane position with view direction
        # If view direction dot split normal > 0, right child is closer?
        # This depends on axis and view direction [S1, unofficial]
        
        # Placeholder for actual geometric test
        if is_front_child(node, view_direction):
            traverse(node['right'])
            traverse(node['left'])
        else:
            traverse(node['left'])
            traverse(node['right'])
            
    traverse(root)
    return order
```

## Subtopics

### [Incremental Load Balancing with Brick Caching](9862-incremental-load-balancing-with-brick-caching.md)
A dynamic load balancing technique for distributed volume rendering that incrementally moves bricks between cluster nodes to correct imbalance, using a caching mechanism to reduce transfer costs when reversing moves. It sets itself apart from full-tree balancing methods by processing only one node per frame to respect limited computation time, and from naive incremental methods by employing a brick cache to mitigate oscillation and stagnation caused by high network transfer costs.

### [Domain Block Decomposition](10898-domain-block-decomposition.md)
Domain Block Decomposition is a view-independent parallel volume rendering strategy that distributes a homogenizing kd-tree into roughly equal-cost subtrees, assigning each subtree to a distinct rendering node. It differs from Cost-range Decomposition by allowing the final image to be generated by any node (specifically the one assigned the "farthest" subtree) rather than always the master node, which may necessitate an additional communication step to send the final image to the master.

## Operating characteristics
**Time and Memory Complexity:**
*   **Construction:** $O(N \log N)$ for building the kd-tree from $N$ bricks, assuming sorting at each level [S1, unofficial].
*   **Traversal:** $O(P)$ for assigning $P$ processors, depending on the balancing strategy [S1, unofficial].
*   **Memory:** $O(N)$ to store the tree structure and brick data [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales with the number of processors $P$ and the depth of the kd-tree [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Enables efficient empty-space skipping, supports dynamic load redistribution based on cost/occlusion, and provides a natural structure for compositing order determination [S1, unofficial].
*   **Limitations:** Requires replication of boundary data for correct compositing, which increases memory usage [S1, unofficial]. Balancing strategies may be complex to implement [S1, unofficial].

**When to Use:**
*   Use when workload varies significantly between frames or regions (e.g., due to occlusion or varying density) [S1, unofficial].
*   Use when empty-space skipping is beneficial [S1, unofficial].

**When Not to Use:**
*   Use when communication overhead for boundary replication is prohibitive [S1, unofficial].
*   Use when static decomposition is sufficient and workload is uniform [S1, unofficial].

## Background
The use of kd-trees for load balancing in volume rendering was proposed in various works, including those cited in [S1, unofficial] (e.g., [51, 58, 56, 47, 21]). The general principle of using hierarchical data structures like kd-trees to manage load statistics and partition data is well-established in parallel volume rendering [S1, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
