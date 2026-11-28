# Hierarchical Partition Optimization

Also called: Bottom-up Lagrangian optimization, Dynamic programming for hierarchical partitions, Offline optimization setup, Preprocessing Algorithm

## Overview
Hierarchical Partition Optimization (HPO) is an offline preprocessing method that computes an optimal adaptive partitioning of a volume dataset and assigns extraction methods to each partition block to minimize a combined cost of memory usage and extraction time [S1, unofficial]. It solves the problem of balancing memory constraints against rendering performance by constructing a hierarchical tree of partitions where each node is either a single block or a subdivision into children, optimized via a Lagrangian relaxation [S1, unofficial]. It sets itself apart from sibling methods like [Min/Max Tree Construction](1623-min-max-tree-construction.md) by explicitly optimizing for a trade-off between two competing metrics (space and time) rather than just geometric homogeneity or structural compactness [S1, unofficial].

## How it works
**Inputs:**
- A 3D scalar volume dataset [S1, unofficial].
- A set of $m$ possible extraction methods, each with known space and time characteristics [S1, unofficial].
- A sequence of Lagrange multipliers $(\lambda_i)$ with $0 < \lambda_1 < \lambda_2 < \dots < \lambda_l$ [S1, unofficial].
- A resolution parameter $\delta > 0$ [S1, unofficial].

**Outputs:**
- An optimal hierarchical partition $P^*$ and an assignment vector $A^*$ mapping each leaf block to one of the $m$ methods [S1, unofficial].
- A lookup table of optimal costs for various $\lambda$ values to allow runtime selection under different memory constraints [S1, unofficial].

**Data Structures:**
- **Maximal Partition Tree ($P_{max}$):** A full octree (or quadtree in 2D) where leaves are minimal blocks (e.g., $2 \times 2 \times 2$ voxels) [S1, unofficial]. Each node $\nu$ stores:
  - $B_\nu$: The set of voxels in the block [S1, unofficial].
  - $J_\lambda(P_\nu, A_\nu)$: The minimal local Lagrangian cost for the subtree rooted at $\nu$ [S1, unofficial].
  - $M(P_\nu, A_\nu)$: The memory required for the optimal partition at $\nu$ [S1, unofficial].
  - $E[T(P_\nu, A_\nu, t)]$: The expected extraction time for the optimal partition at $\nu$ [S1, unofficial].

**Steps:**

1.  **Read Volume Data:** Load the scalar field into memory [S1, unofficial].
2.  **Build Maximal Tree:** Construct $P_{max}$ using a minimal block size (e.g., $2 \times 2 \times 2$) as the leaf termination criterion [S1, unofficial].
3.  **Estimate Block Costs:** For every node $\nu$ in $P_{max}$ and for each method $a \in \{1, \dots, m\}$, estimate $M(B_\nu, a)$ and $E[T(B_\nu, a, t)]$ [S1, unofficial].
4.  **Select Parameters:** Choose a sequence of Lagrange multipliers $\lambda_1 < \lambda_2 < \dots < \lambda_l$ and a resolution parameter $\delta > 0$ [S1, unofficial].
5.  **Bottom-Up Optimization:** For each $\lambda_i$:
    -   Initialize leaf nodes. For a leaf node $\nu$ (trivial partition $P_{\nu 0}$ consisting of only $B_\nu$), compute the cost for each method $a$:
        $$ J_\lambda(P_{\nu 0}, (a)) = E[T(B_\nu, a, t)] + \lambda \cdot M(B_\nu, a) $$
    -   Select the best method $a^*$ for the leaf:
        $$ a^* = \arg \min_{a \in \{1, \dots, m\}} J_\lambda(P_{\nu 0}, (a)) $$
    -   Set $J_\lambda(P_\nu, A_\nu) = \min_a J_\lambda(P_{\nu 0}, (a))$ and store the corresponding $M$ and $T$ values [S1, unofficial].
    -   Process inner nodes bottom-up. For an inner node $\nu$, compare two options:
        1.  **Trivial Partition:** Do not split $\nu$. Use the best single method $a$ for the whole block $B_\nu$.
            $$ Cost_{triv} = \min_{a \in \{1, \dots, m\}} \left( E[T(B_\nu, a, t)] + \lambda \cdot M(B_\nu, a) \right) $$
        2.  **Partitioned:** Split $\nu$ into its children $\mu$. The cost is the sum of the optimal costs of the children plus overhead for indicating the split:
            $$ Cost_{part} = \Delta E[T](\nu) + \lambda \Delta M + \sum_{\mu \in children(\nu)} J_\lambda(P_\mu, A_\mu) $$
            Where $\Delta E[T](\nu)$ is the time overhead to check node $\nu$ and $\Delta M$ is the space overhead to store the partition structure [S1, unofficial].
    -   Set $J_\lambda(P_\nu, A_\nu) = \min(Cost_{triv}, Cost_{part})$. Store the choice (split or not) and the associated $M(P_\nu, A_\nu)$ and $E[T(P_\nu, A_\nu, t)]$ [S1, unofficial].
6.  **Refine $\lambda$:** Insert more $\lambda$ values if the memory requirements of consecutive solutions do not satisfy:
    $$ \frac{M(P^*_{i}, A^*_{i})}{M(P^*_{i+1}, A^*_{i+1})} \le 1 + \delta $$
    Repeat Step 5 for new $\lambda$ values [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_partition_optimization(volume_data, methods, lambda_sequence, delta):
    # 1. Read volume data
    data = volume_data
    
    # 2. Build full partitioning tree P_max
    # Leaves are minimal blocks, e.g., 2x2x2
    def build_octree(block_voxels):
        if block_voxels.shape <= (2, 2, 2):
            return Node(block_voxels, is_leaf=True)
        children = []
        for sub_block in split_into_8(block_voxels):
            children.append(build_octree(sub_block))
        return Node(block_voxels, is_leaf=False, children=children)
    
    root = build_octree(data)
    
    # 3. Estimate costs for all nodes and methods
    # costs[node][method] = (time, space)
    costs = {}
    for node in traverse_all(root):
        block = node.voxels
        costs[node] = {}
        for method in methods:
            # Estimate space and time for method on block
            # [G] Standard complexity estimation based on block dimensions and method algorithm
            t, m = method.estimate(block)
            costs[node][method] = (t, m)
            
    # 4. Choose lambda sequence and delta (provided as input)
    
    # 5. Run bottom-up optimization for each lambda
    results = {}
    for lam in lambda_sequence:
        # Initialize leaf costs
        for node in traverse_leaves(root):
            best_cost = float('inf')
            best_method = None
            for method in methods:
                t, m = costs[node][method]
                j = t + lam * m
                if j < best_cost:
                    best_cost = j
                    best_method = method
            node.optimal_cost = best_cost
            node.optimal_method = best_method
            node.optimal_time = costs[node][best_method][0]
            node.optimal_space = costs[node][best_method][1]
            node.split = False
            
        # Bottom-up pass
        for node in traverse_inner_bottom_up(root):
            # Option 1: Trivial partition (no split)
            best_triv_cost = float('inf')
            best_triv_method = None
            for method in methods:
                t, m = costs[node][method]
                j = t + lam * m
                if j < best_triv_cost:
                    best_triv_cost = j
                    best_triv_method = method
            
            # Option 2: Partitioned (split into children)
            # Overhead for indicating split
            # [G] These values depend on the specific data structure used for the partition tree
            delta_t = 0.0 
            delta_m = 0.0 
            
            sum_children_cost = 0
            sum_children_time = 0
            sum_children_space = 0
            
            for child in node.children:
                sum_children_cost += child.optimal_cost
                sum_children_time += child.optimal_time
                sum_children_space += child.optimal_space
                
            cost_part = (delta_t + sum_children_time) + lam * (delta_m + sum_children_space)
            
            # Choose minimum
            if best_triv_cost <= cost_part:
                node.optimal_cost = best_triv_cost
                node.optimal_method = best_triv_method
                node.optimal_time = costs[node][best_triv_method][0]
                node.optimal_space = costs[node][best_triv_method][1]
                node.split = False
            else:
                node.optimal_cost = cost_part
                node.optimal_method = None
                node.optimal_time = delta_t + sum_children_time
                node.optimal_space = delta_m + sum_children_space
                node.split = True
                
        results[lam] = {
            'partition': root,
            'total_space': root.optimal_space,
            'total_time': root.optimal_time
        }
        
    # 6. Refine lambda if needed
    # Check memory ratio between consecutive lambdas
    sorted_lambdas = sorted(results.keys())
    refined_lambdas = sorted_lambdas[:]
    i = 0
    while i < len(refined_lambdas) - 1:
        lam_curr = refined_lambdas[i]
        lam_next = refined_lambdas[i+1]
        space_curr = results[lam_curr]['total_space']
        space_next = results[lam_next]['total_space']
        
        if space_curr / space_next > 1 + delta:
            # Insert more lambdas between lam_curr and lam_next
            mid_lam = (lam_curr + lam_next) / 2.0
            refined_lambdas.insert(i + 1, mid_lam)
            # Recompute for the new lambda
            # [G] Standard binary search insertion for Lagrange multiplier refinement
            pass
        i += 1
        
    return results
```

**Edge Cases and Pitfalls:**
-   **Precision:** The Lagrangian multiplier $\lambda$ must be chosen carefully. If $\lambda$ is too small, the solution favors time over memory; if too large, it favors memory over time [S1, unofficial].
-   **Overhead Estimation:** The terms $\Delta E[T](\nu)$ and $\Delta M$ are critical [S1, unofficial].
-   **Degenerate Input:** If the volume has no variation, the trivial partition will likely be optimal for all $\lambda$ [S1, unofficial].

## Subtopics

### [Restricted Adjacency Merging](3556-restricted-adjacency-merging.md)
Restricted Adjacency Merging is a load-balancing strategy for parallel isosurface extraction that allows completed sub-blocks to merge with available neighbors beyond immediate siblings, provided they share a geometric common boundary. It solves the problem of idle processors waiting for slower sibling blocks to finish extraction, which creates bottlenecks when isosurface components are unevenly distributed across the spatial decomposition. It sets itself apart from strict hierarchical merging by introducing a "common border" criterion based on node ancestry types, enabling asynchronous merging of a block with its brother or a brother's child (nephew).

### Rotated Partition Sequence Generation
Rotated Partition Sequence Generation is a technique for constructing a sequence of partitions to improve DAG compression in tetrahedral mesh rendering [S2]. It starts with a partition $\Pi_0$ of the original mesh $M_0$ [S2]. For odd $i$, the mesh $M_i$ is rotated by 45 degrees around an axis $a$ (chosen to rotate around the two shortest axes of the bounding box) centered at the centroid of all vertices [S2]. For even $i$, the non-rotated mesh $M_i$ is used to construct the next partition [S2]. DAG cuts between $\Pi_i$ and $\Pi_{i+1}$ are computed by analyzing how tetrahedra of floor segments spread over ceiling segments [S2]. Segments of $\Pi_{i+1}$ are simplified, leaving their borders unchanged, to create an approximated mesh $M_{i+1}$ [S2].

### Dynamic programming for volume distribution
Dynamic programming for volume distribution is an approach used to create flex-blocks within a kd-tree like partition of the scene, giving the user control over the tradeoff between distribution quality and algorithm run time [S3, unofficial]. It is used in conjunction with a "moving walls" algorithm that partitions the scene into cells containing closely cropped blocks of volume data [S3, unofficial]. The total size of each group of data blocks is closely matched to a target memory size [S3, unofficial].

### Entropy-guided INR Spatial Partitioning
Entropy-guided INR Spatial Partitioning is a method applying divide-and-conquer spatial partitioning with entropy-guided INR sizing to compress datasets faster than centralized INR methods [S4, unofficial]. It applies divide-and-conquer spatial partitioning with entropy-guided INR sizing to compress datasets [S4, unofficial]. Compressed datasets up to 23.44 GB more than 10× faster than centralized INR methods, with PSNR that matches or exceeds theirs [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** The preprocessing step runs in $O(N \cdot m)$ for leaves and $O(N \cdot m)$ for inner nodes, where $N$ is the number of nodes in $P_{max}$ and $m$ is the number of methods [S1, unofficial]. The refinement step adds iterations proportional to the density of $\lambda$ values [S1, unofficial].
-   **Memory Complexity:** $O(N \cdot m)$ to store cost estimates and optimal choices for all nodes and methods [S1, unofficial].
-   **Performance:** The method is designed for offline use [S1, unofficial]. The preprocessing needs to be executed only once and can be carried out offline [S1, unofficial]. Results can be stored to a file and used multiple times for different memory constraints [S1, unofficial].
-   **Strengths:** Provides optimal trade-offs between memory and time for a given set of methods [S1, unofficial]. Flexible to different memory constraints via $\lambda$ selection [S1, unofficial].
-   **Limitations:** Requires accurate estimation of time and space for all methods on all block sizes [S1, unofficial]. Offline nature means it cannot adapt to dynamic scene changes during rendering [S1, unofficial].

## Background
Hierarchical Partition Optimization was introduced in the context of optimal hybrid memory-constrained isosurface extraction [S1, unofficial]. It builds on the principle of optimality from dynamic programming, applying Lagrangian relaxation to separate the multi-objective optimization (time and space) into a single-objective problem parameterized by $\lambda$ [S1, unofficial]. The method allows for the construction of adaptive partitions that are optimal for a given memory budget [S1, unofficial].

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
[S2] Segment-based tetrahedral meshing and rendering, https://doi.org/10.1145/1174429.1174507, fetched 2026-11-06
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S4] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
