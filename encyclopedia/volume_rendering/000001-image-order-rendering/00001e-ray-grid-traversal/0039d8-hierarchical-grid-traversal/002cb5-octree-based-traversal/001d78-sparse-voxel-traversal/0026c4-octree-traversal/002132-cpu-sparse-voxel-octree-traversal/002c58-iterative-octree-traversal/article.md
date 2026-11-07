# Iterative Octree Traversal

Also called: BaseOctreeRenderer, Non-recursive Octree Ray Tracing, Ray-Tree Traversal for Color Rendering, Ray-Tree Traversal for Depth Visualization, Rekursionsfreie Traversierung eines Octree, Stack-based Octree Traversal, WorkOctreeRenderer

## Overview
Iterative Octree Traversal is a method for raycasting against a 3D Sparse Voxel Octree (SVO) that replaces recursive function calls with an explicit stack to comply with hardware constraints (such as HLSL recursion limits) or to optimize CPU cache behavior [S2, S4, unofficial]. It solves the problem of unbounded call stack depth by simulating the traversal state using fixed-size arrays or dynamic stacks, employing a greedy approach that prioritizes the closest node while handling false positives via backtracking [S2, unofficial]. It sets itself apart from [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by explicitly managing the call stack and from [Parametric Octree Traversal](2361-parametric-octree-traversal.md) by often using discrete slab boundaries or DDA steps rather than parametric intervals for child selection [S2, S4, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   An octree data structure with a root node pointer/address [S2, S4, unofficial].
*   A ray defined by origin $(o_x, o_y, o_z)$ and direction $(d_x, d_y, d_z)$ [S2, S4, unofficial].
*   Maximum iteration count `maxIterations` to limit search depth in greedy implementations [S2, unofficial].

**Outputs:**
*   Intersection distance $t$ to the closest leaf voxel, or $-1$ if no hit is found [S4, unofficial].
*   (In rendering contexts) Color and alpha values written to a pixel buffer [S2, unofficial].

### Data Structures
The method requires explicit state management to replace recursion. Implementations vary between dynamic stack objects and fixed-size parallel arrays [S2, S5, unofficial].

1.  **Explicit Stack (GPU/General):**
    *   `stack_item_t`: Contains slab boundaries (`tx0, ty0, tz0, tx1, ty1, tz1`), node address (`adr`), and current child index (`idx`) [S5, unofficial].
    *   `STACK_SIZE`: Maximum depth, typically set to 100 [S5, unofficial].
    *   State struct `value`: Contains `m_node_address` (uint32_t), `m_child_num` (uint32_t), and `m_morton_code` (uint32_t) [S6, unofficial].

2.  **Fixed-Size Arrays (CPU/Greedy Approach):**
    *   `skipListPerDepth[4]`: Bitmask tracking visited children at each depth level [S2, unofficial].
    *   `parentListPerDepth[4]`: Stores the index of the parent node at each depth [S2, unofficial].
    *   `positionListPerDepth[4]`: Stores the position (`float3`) of the parent node at each depth [S2, unofficial].
    *   *Note:* The size 4 is specific to implementations where the depth is limited or compressed (e.g., $512^3 / 32^3$ context) [S2, unofficial].

3.  **Cell Structure:**
    *   A cell is an array of 8 elements, each containing a pointer to a descendant or null if empty [S3, unofficial].

### Steps

1.  **Ray Preprocessing and Slab Initialization:**
    *   If any ray direction component is zero, add a small epsilon (e.g., 0.001) to avoid division by zero [S5, unofficial].
    *   Compute slab entry/exit distances for the octree bounds along each axis:
        $$tx_0 = \frac{min_x - o_x}{d_x}, \quad tx_1 = \frac{max_x - o_x}{d_x}$$
        $$ty_0 = \frac{min_y - o_y}{d_y}, \quad ty_1 = \frac{max_y - o_y}{d_y}$$
        $$tz_0 = \frac{min_z - o_z}{d_z}, \quad tz_1 = \frac{max_z - o_z}{d_z}$$
        [S2, S4, S5, unofficial]
    *   If a direction component is negative, swap the corresponding slab entry and exit values to maintain $t_{entry} \le t_{exit}$ [S4, unofficial].
    *   (Optional) Scale all slab boundaries by a fixed-point factor (e.g., `tnum_scale = 10000`) to convert them to integer coordinates for traversal [S5, unofficial].

2.  **Initialization:**
    *   Initialize the stack with the octree root node and its computed slab boundaries [S4, unofficial].
    *   Set initial traversal distance $t = +\infty$ [S4, unofficial].
    *   Initialize depth counters and parent pointers if using the greedy array-based approach [S2, unofficial].

3.  **Traversal Loop:**
    *   While the stack is not empty:
        1.  Pop the current node $n$ from the stack [S4, unofficial].
        2.  Compute ray-AABB intersection for the node's slab boundaries, yielding $t_{enter}$ and $t_{exit}$ [S4, unofficial].
        3.  If $t_{exit} < 0$ or $t_{enter} > t_{exit}$, the ray misses the node; continue to the next iteration [S4, unofficial].
        4.  Determine hit distance $t_{hit} = t_{enter}$ if $t_{enter} > 0$, else $t_{exit}$ (origin inside box) [S4, unofficial].
        5.  If $t_{hit} < t$ (valid intersection closer than current best):
            *   If $n$ is a leaf (voxel):
                *   Update $t = t_{hit}$ [S4, unofficial].
                *   Extract color data and write to buffer if required [S2, unofficial].
                *   Do not break immediately; continue traversal to ensure no closer voxel exists in other branches [S2, unofficial].
            *   If $n$ is an internal node:
                *   Compute child slab midpoints: $txm = (tx_0 + tx_1) \times 0.5$, $tym = (ty_0 + ty_1) \times 0.5$, $tzm = (tz_0 + tz_1) \times 0.5$ [S4, unofficial].
                *   Determine child order based on direction signs ($s_x, s_y, s_z$) to push closer children last so they pop first [S4, unofficial].
                *   Calculate child slab boundaries using the midpoints and octant bits [S4, unofficial].
                *   Apply Non-Intersectable Node Omission: skip pushing a child if its minimum slab boundary $t_{i1} < t_{hit}$ or maximum $t_{i0} > t_{exit}$ [S4, unofficial].
                *   Push valid children onto the stack [S4, unofficial].

4.  **Greedy Backtracking (Alternative Implementation):**
    *   Instead of a standard stack, use a greedy approach with fixed-size arrays [S2, unofficial].
    *   Iterate up to `maxIterations`.
    *   Identify the child with the shortest distance to the viewer [S2, unofficial].
    *   If a false positive occurs (parent hit, but no child hit or a sibling's child is closer):
        *   Backtrack to the parent using `parentListPerDepth` and `positionListPerDepth` [S2, unofficial].
        *   Mark the visited child in `skipListPerDepth` [S2, unofficial].
        *   Proceed to the next best child [S2, unofficial].

5.  **Termination:**
    *   If the stack is empty and no hit found, return $-1$ [S4, unofficial].
    *   If a hit is found, return $t$ [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_octree_traversal(ray_origin, ray_direction, octree_root, max_iterations=100):
    # 1. Preprocess ray direction
    d = list(ray_direction)
    o = list(ray_origin)
    for axis in range(3):
        if d[axis] == 0:
            d[axis] = 0.001  # Avoid division by zero [S5, unofficial]
            
    # 2. Compute initial slab boundaries for root
    tx0 = (octree_root.min[0] - o[0]) / d[0]
    tx1 = (octree_root.max[0] - o[0]) / d[0]
    ty0 = (octree_root.min[1] - o[1]) / d[1]
    ty1 = (octree_root.max[1] - o[1]) / d[1]
    tz0 = (octree_root.min[2] - o[2]) / d[2]
    tz1 = (octree_root.max[2] - o[2]) / d[2]
    
    # Swap if direction is negative to maintain t_enter <= t_exit [S4, unofficial]
    if ray_direction[0] < 0: tx0, tx1 = tx1, tx0
    if ray_direction[1] < 0: ty0, ty1 = ty1, ty0
    if ray_direction[2] < 0: tz0, tz1 = tz1, tz0
    
    stack = [{'node': octree_root, 'tx0': tx0, 'tx1': tx1, 'ty0': ty0, 'ty1': ty1, 'tz0': tz0, 'tz1': tz1}]
    best_t = float('inf')
    
    # 3. Traversal loop
    while stack:
        current = stack.pop()
        node = current['node']
        
        # Ray-AABB intersection
        t_enter = max(current['tx0'], current['ty0'], current['tz0'])
        t_exit = min(current['tx1'], current['ty1'], current['tz1'])
        
        if t_exit < 0 or t_enter > t_exit:
            continue
            
        t_hit = t_enter if t_enter > 0 else t_exit
        
        if t_hit < best_t:
            if node.is_leaf():
                best_t = t_hit
                # Process leaf data (e.g., color extraction) [S2, unofficial]
                # Continue traversal to check other branches for closer hits [S2, unofficial]
            else:
                # Internal node: compute child slab boundaries
                tm_x = (current['tx0'] + current['tx1']) / 2.0
                tm_y = (current['ty0'] + current['ty1']) / 2.0
                tm_z = (current['tz0'] + current['tz1']) / 2.0
                
                children_to_push = []
                for i in range(8):
                    # Determine child octant boundaries based on bits of i
                    x_bit = (i >> 2) & 1
                    y_bit = (i >> 1) & 1
                    z_bit = i & 1
                    
                    c_tx0 = current['tx0'] if x_bit == 0 else tm_x
                    c_tx1 = tm_x if x_bit == 0 else current['tx1']
                    c_ty0 = current['ty0'] if y_bit == 0 else tm_y
                    c_ty1 = tm_y if y_bit == 0 else current['ty1']
                    c_tz0 = current['tz0'] if z_bit == 0 else tm_z
                    c_tz1 = tm_z if z_bit == 0 else current['tz1']
                    
                    c_t_enter = max(c_tx0, c_ty0, c_tz0)
                    c_t_exit = min(c_tx1, c_ty1, c_tz1)
                    
                    # Non-Intersectable Node Omission [S4, unofficial]
                    if c_t_exit > t_hit and c_t_enter < best_t:
                        child_node = node.children[i]
                        if child_node is not None:
                            children_to_push.append((c_t_enter, child_node))
                
                # Ray-Direction Child Ordering: push closer children last so they pop first [S4, unofficial]
                children_to_push.sort(key=lambda x: x[0])
                for _, child_node in reversed(children_to_push):
                    stack.append({
                        'node': child_node,
                        'tx0': c_tx0, 'tx1': c_tx1,
                        'ty0': c_ty0, 'ty1': c_ty1,
                        'tz0': c_tz0, 'tz1': c_tz1
                    })
                    
    return best_t if best_t != float('inf') else -1
```

### Parameters
*   `maxIterations`: Maximum number of iterations to find the ideal leaf node hit. Allows assigning different compute time to different operations (e.g., shadow quality) [S2, unofficial].
*   `tnum_scale`: Fixed-point scaling factor for slab coordinates, typically 10000 [S5, unofficial].
*   `STACK_SIZE`: Maximum depth of the traversal stack, typically 100 [S5, unofficial].

### Edge Cases and Pitfalls
*   **Zero Ray Direction:** Division by zero in slab intersection. Handled by adding 0.001 to the component, which may slightly alter the ray path for exact grid alignments [S5, unofficial].
*   **Negative Ray Direction:** Requires swapping slab entry/exit bounds to maintain consistent traversal order [S4, unofficial].
*   **False Positives:** A parent node may have the shortest distance, but its children might not be hit, or a sibling's child might be closer. The algorithm handles this by continuing traversal or backtracking to the parent and checking the next best child [S2, unofficial].
*   **Recursion Limits:** HLSL and some GPU architectures do not support general method recursion, necessitating this iterative approach [S2, unofficial].

## Subtopics

### [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md)
This method optimizes hierarchical ray marching by fixing the leaf node size to a constant volume (32×32×32 voxels) instead of using the smallest available octree nodes. It switches between octree levels and voxel levels based on hit/miss status to reduce traversal overhead for sparse data, at the cost of potential false positives along the ray direction. It sets itself apart from standard iterative octree traversal by decoupling the inner marching loop from the octree depth, simplifying the inner loop and reducing memory overhead for sparse data.

### [Octree Traversal with LOD & Constant Nodes](4181-octree-traversal-with-lod-constant-nodes.md)
This method renders a Sparse Voxel Octree (SVO) on the CPU by traversing the hierarchy to project voxels into a color buffer, using depth-sorted traversal and hierarchical culling based on projected size. It distinguishes itself from sibling methods by supporting Level of Detail (LOD) switching between constant-value nodes and textured bricks. It builds on [Iterative Octree Traversal](2618-iterative-octree-traversal.md) by adding specific logic for constant node handling and brick texture sampling.

### [Ordered List Generation via N3-Tree Traversal](5023-ordered-list-generation-via-n3-tree-traversal.md)
This method generates a list of spatial bricks ordered from closest to furthest relative to the viewpoint to avoid testing blocks behind untested unloaded blocks. It solves the problem of unnecessary data loading by ensuring that visibility tests are performed in an order where any previously reached node cannot be occluded by a subsequently reached node. It sets itself apart from sibling methods by integrating view-dependent axis prioritization directly into a depth-first traversal of the N3-tree structure, followed by a lightweight distance sort.

### [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md)
Hierarchical Ray Marching is a sparse voxel traversal technique that dynamically switches between octree levels and voxel levels to optimize ray-marching performance. It differs from sibling methods by adjusting the traversal granularity up or down based on hit/miss status, allowing rapid skipping of empty regions at coarse levels and precise sampling at fine levels.

## Operating characteristics
*   **Time Complexity:** $O(\log s)$ in the ideal case, where $s$ is the initial size of the Octree. The leaf raycasting is $O(1)$ as it always processes a fixed 32x32x32 volume [S2, unofficial]. In the worst case (e.g., diagonal traversal through dense data), it can reach $O(s \cdot \log s)$ [S2, unofficial].
*   **Memory Complexity:** Depends on the stack implementation. Fixed-size arrays use constant memory per pixel (e.g., 4 levels tracked) [S2, unofficial]. Explicit stacks use memory proportional to tree depth [S5, unofficial].
*   **Measured Performance:** On Sponza 2k, ESVO (a variant) achieves 95.0 Mrays/s for primary rays and 66.2 Mrays/s for primary + 1 diffuse bounce [S7, unofficial].
*   **Strengths:** Complies with GPU recursion limits; reduces intersection tests via direction-aware ordering and non-intersectable node omission [S4, unofficial]; efficient for sparse data [S1, unofficial].
*   **Limitations:** Potential false positives require backtracking; performance can degrade with dense diagonal rays [S2, unofficial]; fixed-point arithmetic may introduce precision issues if scaling is not handled carefully [S5, unofficial].

## Background
Iterative Octree Traversal originated as a solution to the limitations of recursive ray tracing in shader languages like HLSL, which do not support general recursion [S2, unofficial]. It was developed to enable real-time raycasting of sparse voxel octrees on GPUs and CPUs by explicitly managing the traversal state [S2, S4, unofficial]. The method builds upon earlier discrete ray-tracing techniques [S3, unofficial] and has been adapted for various contexts, including ESVO (Efficient Sparse Voxel Octrees) [S7, unofficial] and CPU-based renderers [S1, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S3] Discrete Ray-Tracing of Huge Voxel Spaces - doczz.net, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25 (unofficial)
[S4] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S5] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S6] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S7] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
