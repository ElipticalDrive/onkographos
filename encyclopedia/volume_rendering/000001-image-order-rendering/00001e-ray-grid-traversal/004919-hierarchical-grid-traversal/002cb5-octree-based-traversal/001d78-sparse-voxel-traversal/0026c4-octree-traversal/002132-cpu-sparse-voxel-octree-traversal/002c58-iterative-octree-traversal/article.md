# Iterative Octree Traversal

Also called: BaseOctreeRenderer, Non-recursive Octree Ray Tracing, Ray-Tree Traversal for Color Rendering, Ray-Tree Traversal for Depth Visualization, Rekursionsfreie Traversierung eines Octree, Stack-based Octree Traversal, WorkOctreeRenderer

## Overview
Iterative Octree Traversal is a method for raycasting against a Sparse Voxel Octree (SVO) that replaces recursive function calls with an explicit stack to manage traversal state [S2, S4, unofficial]. It solves the problem of recursion limits in environments like HLSL or the need for explicit stack management on the CPU, finding the closest voxel intersection by greedily selecting the nearest child node and backtracking on false positives [S2, unofficial]. It sets itself apart from [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by using fixed-size arrays to simulate the call stack and explicitly managing depth and parent pointers for backtracking [S2, unofficial], and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by relying on slab-based intersection tests rather than parametric ray equations for child selection.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $r$ defined by origin $o$ and direction $d$, and an octree structure with a root node.
*   **Output:** The distance $t$ to the closest intersected voxel, or $-1$ if no intersection is found.

**Data Structures:**
*   **Stack:** A fixed-size array-based stack is used to store traversal state. In GPU implementations (HLSL), this is often simulated using parallel arrays indexed by depth [S2, unofficial].
    *   `skipListPerDepth[4]`: Integer array tracking visited children per depth level.
    *   `parentListPerDepth[4]`: Integer array storing the index of the parent node at each depth.
    *   `positionListPerDepth[4]`: Float3 array storing the position of the parent node at each depth.
    *   *Note:* The size 4 is specific to implementations with a depth limit or specific grid sizes (e.g., $512^3$ with $32^3$ leaves) [S2, unofficial].
*   **Stack Item (CPU/General):** Contains slab boundaries (`tx0, ty0, tz0, tx1, ty1, tz1`), node address (`adr`), and current child index (`idx`) [S5, unofficial].
*   **State Struct (GPU/Specific):** Contains `m_node_address`, `m_child_num`, and `m_morton_code` [S6, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize the stack with the root node of the octree [S4, unofficial].
    *   Initialize traversal state: set stack index and level to 0, mark as first node, get root node entry, and decode child validity/leaf status into arrays [S6, unofficial].
    *   Handle ray direction components: If any component is zero, add a small epsilon (e.g., 0.001) to avoid division by zero [S5, unofficial]. If a component is negative, negate the ray origin and direction components for that axis, and set the corresponding bit in `dir_mask` (x=4, y=2, z=1) [S5, unofficial].
    *   Compute initial slab entry and exit distances for the octree bounds along each axis using the ray origin and direction [S5, unofficial]:
        $$tx_0 = (min_x - o_x)/d_x, \quad tx_1 = (max_x - o_x)/d_x$$
        $$ty_0 = (min_y - o_y)/d_y, \quad ty_1 = (max_y - o_y)/d_y$$
        $$tz_0 = (min_z - o_z)/d_z, \quad tz_1 = (max_z - o_z)/d_z$$
    *   Scale slab boundaries by a fixed-point factor (e.g., `tnum_scale = 10000`) to convert them to integer coordinates for traversal [S5, unofficial].

2.  **Main Traversal Loop:**
    *   While the stack is not empty:
        *   Pop node $n$ from the stack [S4, unofficial].
        *   Perform a Ray-AABB intersection test between ray $r$ and the bounding box $b$ of node $n$, resulting in $t'$ [S4, unofficial].
        *   If $t' \in (0, t)$ (intersection is valid and closer than current best $t$):
            *   If $n$ is a voxel (leaf):
                *   Update $t$ to $t'$ [S4, unofficial].
                *   [S2, unofficial] If leaf size is $\le 32$ (e.g., $32 \times 32 \times 32$), perform detailed raycasting on the leaf volume. If a voxel is found with distance $t_{minVoxel} < t$, update $t$ [S2, unofficial].
            *   If $n$ is an internal node:
                *   Compute child slab boundaries by selecting either the current slab boundary or the midpoint based on the child index bits, then flip indices using `dir_mask` [S5, unofficial].
                *   Iterate through the 8 children. For each child $i$:
                    *   Check if the child has been visited using `skipListPerDepth` [S2, unofficial].
                    *   Check if the child has data [S2, unofficial].
                    *   **Non-Intersectable Node Omission:** If a child's minimum slab boundary $t_{i1}$ is less than the current best intersection $t_{min}$, or if the maximum slab boundary $t_{i0}$ is greater than $t_{max}$, skip the child [S4, unofficial].
                    *   If the child is valid and not culled, push its slab boundaries and address onto the stack [S5, unofficial].
                    *   [S2, unofficial] Push children in reverse order (8 down to 1) so that child 1 is popped first [S4, unofficial].

3.  **Backtracking and State Management:**
    *   [S2, unofficial] The algorithm uses a greedy approach: it always selects the child with the shortest distance to the viewer. If a false positive occurs (a parent has the shortest distance but its children do not intersect, or a child of a different parent is closer), the algorithm backtracks [S2, unofficial].
    *   If no valid next parent is found (backtrack):
        *   Reset `skipListPerDepth` for the current depth.
        *   Decrement `currentDepth`.
        *   If `currentDepth < 0`, break the loop.
        *   Retrieve the previous parent index and load the parent node [S2, unofficial].
    *   Else (forward traversal):
        *   Mark the current child as visited in `skipListPerDepth` [S2, unofficial].
        *   Increment `currentDepth` [S2, unofficial].
        *   Update parent and current index for the next iteration [S2, unofficial].

4.  **Termination:**
    *   If the stack is empty and no intersection was found ($t = +\infty$), return $-1$ [S4, unofficial].
    *   Otherwise, return $t$ [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_octree_traversal(ray, octree_root, max_iterations=100):
    # Initialize stack with root node
    # Stack stores: (node_address, slab_tx0, slab_ty0, slab_tz0, slab_tx1, slab_ty1, slab_tz1, child_index)
    stack = [(octree_root, 0, 0, 0, 0, 0, 0, 0)]
    
    # Best intersection distance found so far
    t_best = float('inf')
    
    # Arrays to simulate stack state for backtracking (GPU-style)
    # In CPU implementation, these can be part of the stack item or separate structures
    skip_list = [0] * 4
    parent_list = [0] * 4
    position_list = [0.0, 0.0, 0.0] * 4
    current_depth = 0
    
    # Pre-process ray direction
    dir_mask = 0
    if ray.d.x == 0: ray.d.x = 0.001
    if ray.d.y == 0: ray.d.y = 0.001
    if ray.d.z == 0: ray.d.z = 0.001
    
    if ray.d.x < 0:
        ray.o.x = -ray.o.x
        ray.d.x = -ray.d.x
        dir_mask |= 4
    if ray.d.y < 0:
        ray.o.y = -ray.o.y
        ray.d.y = -ray.d.y
        dir_mask |= 2
    if ray.d.z < 0:
        ray.o.z = -ray.o.z
        ray.d.z = -ray.d.z
        dir_mask |= 1
        
    # Initial slab calculation for root
    # Assuming octree bounds are known
    tx0 = (octree_root.min.x - ray.o.x) / ray.d.x
    tx1 = (octree_root.max.x - ray.o.x) / ray.d.x
    ty0 = (octree_root.min.y - ray.o.y) / ray.d.y
    ty1 = (octree_root.max.y - ray.o.y) / ray.d.y
    tz0 = (octree_root.min.z - ray.o.z) / ray.d.z
    tz1 = (octree_root.max.z - ray.o.z) / ray.d.z
    
    # Scale to integers if using fixed-point arithmetic
    tnum_scale = 10000
    tx0 *= tnum_scale; tx1 *= tnum_scale
    ty0 *= tnum_scale; ty1 *= tnum_scale
    tz0 *= tnum_scale; tz1 *= tnum_scale
    
    # Push initial root state
    stack[0] = (octree_root, tx0, ty0, tz0, tx1, ty1, tz1, 0)
    
    for iteration in range(max_iterations):
        if not stack:
            break
            
        # Pop node
        node, s_tx0, s_ty0, s_tz0, s_tx1, s_ty1, s_tz1, child_idx = stack.pop()
        
        # Ray-AABB intersection test
        t_prime = ray_aabb_intersect(ray, node.bbox)
        
        if 0 < t_prime < t_best:
            if node.is_leaf():
                # Leaf node: detailed raycasting
                t_leaf = ray_voxel_intersect(ray, node.voxel_data)
                if t_leaf < t_best:
                    t_best = t_leaf
            else:
                # Internal node: push children
                # Compute midpoints
                tm_x = (s_tx0 + s_tx1) * 0.5
                tm_y = (s_ty0 + s_ty1) * 0.5
                tm_z = (s_tz0 + s_tz1) * 0.5
                
                # Iterate children 7 down to 0 (reverse order for stack)
                for i in range(7, -1, -1):
                    # Check skip list
                    if (skip_list[current_depth] >> i) & 1:
                        continue
                    
                    # Get child properties
                    child = node.get_child(i)
                    if not child or not child.has_data:
                        continue
                        
                    # Compute child slab boundaries
                    # This depends on the octant index i and dir_mask
                    # Simplified: select min/max based on bit i and midpoint
                    c_tx0, c_tx1 = compute_child_slab(i, 0, s_tx0, s_tx1, tm_x, dir_mask)
                    c_ty0, c_ty1 = compute_child_slab(i, 1, s_ty0, s_ty1, tm_y, dir_mask)
                    c_tz0, c_tz1 = compute_child_slab(i, 2, s_tz0, s_tz1, tm_z, dir_mask)
                    
                    # Non-Intersectable Node Omission
                    # If child's min slab > t_best, skip
                    if min(c_tx0, c_ty0, c_tz0) > t_best * tnum_scale:
                        continue
                        
                    # Push child
                    stack.append((child, c_tx0, c_ty0, c_tz0, c_tx1, c_ty1, c_tz1, 0))
                    
                    # Update skip list for backtracking
                    skip_list[current_depth] |= (1 << i)
                    
        # Backtracking logic (simplified for iterative stack)
        # In the greedy approach, if a node is popped and processed, 
        # we need to ensure we visit the next sibling if the current one failed to yield a better hit.
        # The skip_list mechanism handles this by marking visited children.
        # If stack becomes empty, we backtrack by popping parents and trying their next unvisited child.
        
    if t_best == float('inf'):
        return -1
    return t_best

def compute_child_slab(octant_idx, axis, t0, t1, tm, dir_mask):
    # Determine if the child is in the "min" or "max" half of the parent along the axis
    # Bit 0 for X, 1 for Y, 2 for Z
    bit = 1 << axis
    is_max = (octant_idx & bit) != 0
    
    # If direction is negative, the "min" slab in local space might correspond to the "max" slab in world space
    # This is handled by dir_mask flipping
    if (dir_mask >> axis) & 1:
        is_max = not is_max
        
    if is_max:
        return tm, t1
    else:
        return t0, tm
```

**Parameters:**
*   `maxIterations`: Maximum number of iterations allowed to find the ideal leaf node hit. Allows assigning different compute time to different operations (e.g., shadow quality) [S2, unofficial].
*   `tnum_scale`: Fixed-point scaling factor for slab coordinates, set to 10000 in this implementation [S5, unofficial].
*   `STACK_SIZE`: Maximum depth of the traversal stack, set to 100 [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Ray Direction:** Division by zero in slab intersection if ray direction components are zero. Handled by adding 0.001 to the component [S5, unofficial].
*   **Negative Ray Direction:** Requires flipping the ray origin and direction to maintain consistent traversal order, tracked via `dir_mask` bits [S5, unofficial].
*   **False Positives:** A parent node may have the shortest distance to the viewer, but its children might not be hit by the ray, or a child of a different parent might be closer. The algorithm handles this by backtracking to the parent and checking the next best child [S2, unofficial].
*   **Recursion Limits:** HLSL does not support general method recursion, requiring an iterative approach with explicit stack management [S2, unofficial].

## Subtopics

### [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md)
This method optimizes hierarchical ray marching by fixing the leaf node size to a constant volume (32×32×32 voxels) instead of using the smallest available octree nodes. It switches between octree levels and voxel levels based on hit/miss status to reduce traversal overhead for sparse data, at the cost of potential false positives along the ray direction. It sets itself apart from standard iterative octree traversal by decoupling the inner marching loop from the octree depth, simplifying the inner loop and reducing memory overhead for sparse data.

### [Octree Traversal with LOD & Constant Nodes](4181-octree-traversal-with-lod-constant-nodes.md)
This method renders a Sparse Voxel Octree (SVO) on the CPU by traversing the hierarchy to project voxels into a color buffer, using depth-sorted traversal and hierarchical culling based on projected size. It distinguishes itself from sibling methods by supporting Level of Detail (LOD) switching between constant-value nodes and textured bricks. It builds on [Iterative Octree Traversal](2618-iterative-octree-traversal.md) by adding specific logic for constant node handling and brick texture sampling.

### [Ordered List Generation via N3-Tree Traversal](5023-ordered-list-generation-via-n3-tree-traversal.md)
This method generates a list of spatial bricks ordered from closest to furthest relative to the viewpoint to avoid testing blocks behind untested unloaded blocks. It solves the problem of unnecessary data loading by ensuring that visibility tests are performed in an order where any previously reached node cannot be occluded by a subsequently reached node. It sets itself apart from sibling methods by integrating view-dependent axis prioritization directly into a depth-first traversal of the N3-tree structure, followed by a lightweight distance sort.

### [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md)
Hierarchical Ray Marching is a sparse voxel traversal technique that dynamically switches between octree levels and voxel levels to optimize ray-marching performance. It differs from sibling methods by adjusting the traversal granularity up or down based on hit/miss status, allowing rapid skipping of empty regions at coarse levels and precise sampling at fine levels.

### [Ray-Direction Child Ordering](5462-ray-direction-child-ordering.md)
Ray-Direction Child Ordering is an iterative traversal optimization for sparse voxel octrees that processes child nodes in a specific sequence determined by the ray's direction, ensuring that nodes closer to the ray origin are visited first. It solves the inefficiency of standard recursive traversal, which examines all eight children and performs redundant intersection tests for non-intersected subtrees, by using a deterministic neighbor-linking strategy to skip empty or non-intersected branches. Unlike its siblings, it avoids recursion entirely within a node's children by maintaining a `currNode` state and computing the next neighbor based on exit plane t-values, thereby reducing the number of intersection tests and improving performance on sparse structures.

### [Octree Iteration with Culling](11319-octree-iteration-with-culling.md)
Octree Iteration with Culling is a traversal method that visits only the leaf nodes of a sparse voxel octree that intersect a specified spatial region, typically an axis-aligned bounding box (AABB). It solves the problem of efficiently identifying relevant geometry for rendering or raycasting by skipping empty or non-intersecting subtrees, thereby reducing the number of nodes processed compared to a full tree traversal. It differs from sibling methods like [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md) by supporting arbitrary leaf sizes and dynamic culling regions, and from [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md) by focusing on node enumeration rather than direct volume integration.

### ESVO (Efficient Sparse Voxel Octrees)
A port of the 'Efficient Sparse Voxel Octrees' algorithm, specifically excluding contour generation and beam optimization features [S7, unofficial]. The traversal is a near 1:1 port of the original ESVO implementation [S7, unofficial]. Node encoding was changed: relative pointers are 'backwards' to allow trees to be built without intermediate buffers (parent nodes appear after children), and far pointers are signaled by a reserved upper range rather than a dedicated bit [S7, unofficial]. Memory cost is approximately 4 bytes per node [S7, unofficial]. Nodes are very small (4 bytes each) and localized, with a z-curve layout resulting from building the tree in depth-first order [S7, unofficial]. Performance is underwhelming because it takes many extra iterations to ascend and descend the tree and advance positions by DDA [S7, unofficial]. Editing cost is approximately O(log m), leveraging path copying for persistent data structures [S7, unofficial]. On Sponza 2k, ESVO achieves 95.0 Mrays/s for primary rays and 66.2 Mrays/s for primary + 1 diffuse bounce [S7, unofficial].

### Non-Intersectable Node Omission
Optimization to avoid adding children to the stack if they cannot possibly intersect the ray closer than the current best intersection t. It uses precomputed slab boundaries of children [S4, unofficial]. If a child's minimum slab boundary $t_{i1}$ (for $i \in \{x,y,z\}$) is less than the current best intersection $t_{min}$, the child is guaranteed not to intersect closer than $t_{min}$ and is skipped [S4, unofficial]. Similarly, if a child's maximum slab boundary $t_{i0}$ is greater than $t_{max}$, the child is skipped [S4, unofficial]. Factors separating children of a node are defined as $((tx_0, ty_0, tz_0) + (tx_1, ty_1, tz_1)) \times 0.5$ [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** O(log s) in the ideal case, where s is the initial size of the Octree [S2, unofficial]. The leaf raycasting is O(1) as it always processes a fixed 32x32x32 volume [S2, unofficial]. In the worst case, it is O(s · log s), occurring when every 32x32x32 leaf node along the diagonal must be examined [S2, unofficial].
*   **Memory Complexity:** O(1) auxiliary space for the stack, with stack size bounded by the maximum depth of the octree [S5, unofficial].
*   **Measured Performance:**
    *   On Sponza 2k (499k * 8³ voxels), ESVO variant achieves 95.0 Mrays/s for primary rays and 66.2 Mrays/s for primary + 1 diffuse bounce [S7, unofficial].
    *   On Eco House 2k (448k * 8³ voxels), ESVO achieves 67.1 Mrays/s for primary rays [S7, unofficial].
    *   On Forest Lake 2k (807k * 8³ voxels), ESVO achieves comparable throughput to other methods, possibly due to low memory bandwidth, given that nodes are very small (4 bytes each) and localized [S7, unofficial].
*   **Strengths:** Avoids recursion limits in GPU shaders (HLSL) [S2, unofficial]; reduces traversal overhead for sparse data by fixing leaf sizes [S1, unofficial]; efficient culling of non-intersectable nodes [S4, unofficial].
*   **Limitations:** Potential false positives requiring backtracking [S2, unofficial]; performance can be underwhelming due to extra iterations for tree ascension/descension in ESVO variant [S7, unofficial]; not designed for smooth surfaces, where other approaches may offer better performance [S7, unofficial].
*   **When to Use:** When rendering sparse voxel data on hardware with recursion limits (e.g., older GPUs) or when explicit stack management is preferred for performance tuning.
*   **When Not to Use:** When rendering dense voxel data or smooth surfaces where other traversal methods may be more efficient [S7, unofficial].

## Background
Iterative Octree Traversal was developed to address the limitations of recursive ray-traversal algorithms in environments that do not support recursion, such as HLSL [S2, unofficial]. It is based on the principles of recursive octree traversal but replaces the call stack with an explicit data structure [S2, unofficial]. The method has been implemented in various contexts, including CPU-based voxel raycasting [S1, unofficial] and GPU-based volume rendering [S2, S4, unofficial]. The ESVO variant is a port of the 'Efficient Sparse Voxel Octrees' algorithm [S7, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S3] Discrete Ray-Tracing of Huge Voxel Spaces - doczz.net, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25 (unofficial)
[S4] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S5] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S6] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S7] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
