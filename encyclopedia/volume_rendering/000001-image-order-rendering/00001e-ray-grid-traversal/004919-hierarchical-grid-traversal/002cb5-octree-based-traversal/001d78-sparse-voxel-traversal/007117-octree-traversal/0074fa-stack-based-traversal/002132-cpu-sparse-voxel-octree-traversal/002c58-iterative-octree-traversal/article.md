# Iterative Octree Traversal

Also called: BaseOctreeRenderer, Non-recursive Octree Ray Tracing, Ray-Tree Traversal for Color Rendering, Ray-Tree Traversal for Depth Visualization, Rekursionsfreie Traversierung eines Octree, Stack-based Octree Traversal, WorkOctreeRenderer

## Overview
Iterative Octree Traversal is a method for raycasting against a Sparse Voxel Octree (SVO) that replaces recursive function calls with an explicit stack-based data structure, primarily to satisfy hardware constraints such as HLSL recursion limits or CPU stack size restrictions [S2, S4, unofficial]. It solves the problem of traversing hierarchical spatial data without risking stack overflow or violating shader model restrictions, using a greedy approach that prioritizes the closest node and backtracks on false positives [S2, unofficial]. Unlike [Recursive Octree Traversal](2632-recursive-octree-traversal.md), which relies on the call stack, this method explicitly manages traversal state and parent pointers to navigate the tree hierarchy [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree structure (root node address, hierarchy data), a ray defined by origin $O$ and direction $D$, and a maximum iteration count `maxIterations`.
*   **Output:** The color/depth of the closest voxel intersected by the ray, or a default value (e.g., background) if no intersection is found.

**Data Structures:**
1.  **Stack (`stack_item_t`):** A fixed-size array simulating the call stack. Each item contains:
    *   `tx0, ty0, tz0`: Entry slab boundaries (scaled to fixed-point integers).
    *   `tx1, ty1, tz1`: Exit slab boundaries (scaled to fixed-point integers).
    *   `adr`: Node address/index in the octree data structure.
    *   `idx`: Current child index being processed (0-7).
    *   *Note:* In some implementations, auxiliary arrays `skipListPerDepth`, `parentListPerDepth`, and `positionListPerDepth` are used to track visited children and parent context for backtracking [S2, unofficial].
2.  **State Variables:**
    *   `minDistToNode`: Tracks the closest intersection distance found so far.
    *   `currentDepth`: Tracks the current depth in the octree.
    *   `dir_mask`: Bitmask indicating ray direction signs (x=4, y=2, z=1) to handle coordinate flipping [S5, unofficial].

**Steps:**

1.  **Ray Preprocessing:**
    *   If any ray direction component is zero, add a small epsilon (e.g., 0.001) to avoid division by zero [S5, unofficial].
    *   If a direction component is negative, negate the corresponding ray origin and direction components, and set the corresponding bit in `dir_mask` [S5, unofficial].
    *   Compute initial slab boundaries for the octree root bounds $[min_x, max_x] \times [min_y, max_y] \times [min_z, max_z]$:
        $$ tx_0 = \frac{min_x - o_x}{d_x}, \quad tx_1 = \frac{max_x - o_x}{d_x} $$
        $$ ty_0 = \frac{min_y - o_y}{d_y}, \quad ty_1 = \frac{max_y - o_y}{d_y} $$
        $$ tz_0 = \frac{min_z - o_z}{d_z}, \quad tz_1 = \frac{max_z - o_z}{d_z} $$
    *   Scale these boundaries by a fixed-point factor (e.g., `tnum_scale = 10000`) to convert them to integers for traversal [S5, unofficial].

2.  **Initialization:**
    *   Initialize the stack with the root node. Set `minDistToNode` to infinity.
    *   Initialize backtracking structures (e.g., `skipListPerDepth`, `parentListPerDepth`) if using the greedy backtracking variant [S2, unofficial].

3.  **Traversal Loop:**
    *   While the stack is not empty and `minDistToNode` is not yet determined as the final hit (or within `maxIterations`):
        a.  **Pop:** Retrieve the top node `n` from the stack.
        b.  **Intersection Test:** Perform a Ray-AABB intersection test between the ray and the bounding box of node `n`. Let the result be $t'$.
        c.  **Validation:** If $t' \in (0, t)$ (where $t$ is the current best intersection distance, initially infinity):
            i.  **Leaf Check:** If `n` is a leaf node (voxel):
                *   Update $t = t'$.
                *   Extract color/depth data from the leaf.
                *   *Note:* In some variants, the loop continues to check if other branches contain closer voxels [S2, unofficial].
            ii. **Internal Node:** If `n` is an internal node:
                *   Compute slab boundaries for its 8 children. The midpoints $t_m$ are calculated as $((t_0 + t_1) \times 0.5)$ [S4, unofficial].
                *   For each child $i$ (0 to 7):
                    *   **Culling:** If the child's minimum slab boundary $t_{i1}$ is less than the current best intersection $t_{min}$, or if the maximum slab boundary $t_{i0}$ is greater than $t_{max}$, skip the child [S4, unofficial].
                    *   **Push:** If the child is valid and not culled, push its slab boundaries and address onto the stack. Children are often pushed in reverse order (7 down to 0) so that the first child (closest in Morton order or spatial order) is popped first [S4, unofficial].

4.  **Backtracking (Greedy Variant):**
    *   If a false positive occurs (a parent node had the shortest distance, but its children do not intersect closer than the current best), backtrack to the parent [S2, unofficial].
    *   Use `skipListPerDepth` to mark visited children and `parentListPerDepth` to restore parent context.
    *   Decrement `currentDepth` and retrieve the next best sibling from the parent's children list [S2, unofficial].

5.  **Termination:**
    *   If the stack is empty and no hit was found, return -1 (or background color) [S4, unofficial].
    *   If a hit was found, return the associated color/depth [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_octree_traversal(ray, octree, max_iterations=100):
    # Preprocessing
    dir_mask = 0
    if ray.d.x == 0: ray.d.x += 0.001
    if ray.d.x < 0: ray.o.x = -ray.o.x; ray.d.x = -ray.d.x; dir_mask |= 4
    if ray.d.y == 0: ray.d.y += 0.001
    if ray.d.y < 0: ray.o.y = -ray.o.y; ray.d.y = -ray.d.y; dir_mask |= 2
    if ray.d.z == 0: ray.d.z += 0.001
    if ray.d.z < 0: ray.o.z = -ray.o.z; ray.d.z = -ray.d.z; dir_mask |= 1

    # Initial slab calculation
    tx0 = (octree.min_x - ray.o.x) / ray.d.x
    tx1 = (octree.max_x - ray.o.x) / ray.d.x
    ty0 = (octree.min_y - ray.o.y) / ray.d.y
    ty1 = (octree.max_y - ray.o.y) / ray.d.y
    tz0 = (octree.min_z - ray.o.z) / ray.d.z
    tz1 = (octree.max_z - ray.o.z) / ray.d.z
    
    # Scale to fixed point
    tnum_scale = 10000
    tx0 *= tnum_scale; tx1 *= tnum_scale
    ty0 *= tnum_scale; ty1 *= tnum_scale
    tz0 *= tnum_scale; tz1 *= tnum_scale

    # Stack initialization
    stack = []
    stack.append({
        'tx0': tx0, 'ty0': ty0, 'tz0': tz0,
        'tx1': tx1, 'ty1': ty1, 'tz1': tz1,
        'adr': octree.root_address,
        'idx': 0
    })
    
    min_dist = float('inf')
    hit_color = None

    for _ in range(max_iterations):
        if not stack:
            break
            
        current = stack.pop()
        node = octree.get_node(current['adr'])
        
        # Ray-AABB intersection for current node
        t_prime = ray_aabb_intersect(ray, node.bounds)
        
        if 0 < t_prime < min_dist:
            if node.is_leaf():
                min_dist = t_prime
                hit_color = node.color
                # In greedy variant, continue to find potentially closer leaves
            else:
                # Compute child midpoints
                tm_x = (current['tx0'] + current['tx1']) / 2
                tm_y = (current['ty0'] + current['ty1']) / 2
                tm_z = (current['tz0'] + current['tz1']) / 2
                
                # Push children in reverse order
                for i in range(7, -1, -1):
                    # Determine child slab boundaries based on index i and midpoints
                    # This logic depends on specific Morton code mapping
                    child_tx0, child_tx1, child_ty0, child_ty1, child_tz0, child_tz1 = compute_child_slabs(i, current, tm_x, tm_y, tm_z)
                    
                    # Culling check
                    if child_tx1 < min_dist * tnum_scale or child_ty1 < min_dist * tnum_scale or child_tz1 < min_dist * tnum_scale:
                        continue
                    if child_tx0 > min_dist * tnum_scale or child_ty0 > min_dist * tnum_scale or child_tz0 > min_dist * tnum_scale:
                        continue
                        
                    child_node = node.children[i]
                    if child_node is not None:
                        stack.append({
                            'tx0': child_tx0, 'ty0': child_ty0, 'tz0': child_tz0,
                            'tx1': child_tx1, 'ty1': child_ty1, 'tz1': child_tz1,
                            'adr': child_node.address,
                            'idx': 0
                        })
                        
    if min_dist == float('inf'):
        return -1
    return hit_color
```

## Subtopics

### [ESVO (Efficient Sparse Voxel Octrees)](3089-esvo-efficient-sparse-voxel-octrees.md)
GPU-based variant using compact 4-byte node encoding and z-curve layout.

### [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md)
Enforces uniform 32x32x32 leaf nodes for cache efficiency.

### [Octree Traversal with LOD & Constant Nodes](4181-octree-traversal-with-lod-constant-nodes.md)
CPU-based variant supporting LOD switching and constant-value nodes.

### [Ordered List Generation via N3-Tree Traversal](5023-ordered-list-generation-via-n3-tree-traversal.md)
Generates spatially ordered brick lists to avoid testing unloaded blocks.

### [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md)
Dynamically switches between voxel-level stepping and octree-level skipping.

### [Ray-Direction Child Ordering](5462-ray-direction-child-ordering.md)
Orders child nodes based on ray direction to minimize intersection tests.

### [Octree Iteration with Culling](11319-octree-iteration-with-culling.md)
Integrates BoxIntersector checks for early branch termination.

### [Single-kernel Octree Traversal](15600-single-kernel-octree-traversal.md)
Performs entire traversal within a single CUDA kernel per ray.

### Non-Intersectable Node Omission
An optimization where children are not added to the stack if their slab boundaries indicate they cannot intersect the ray closer than the current best intersection $t_{min}$ [S4, unofficial]. Specifically, if a child's minimum slab boundary $t_{i1}$ is less than $t_{min}$, or its maximum slab boundary $t_{i0}$ is greater than $t_{max}$, the child is skipped [S4, unofficial]. This reduces the number of intersection tests and improves performance [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(\log s)$ in the ideal case, where $s$ is the initial size of the octree [S2, unofficial]. The leaf raycasting is $O(1)$ as it processes a fixed volume (e.g., 32x32x32) [S2, unofficial]. In the worst case (e.g., diagonal traversal through dense regions), it can reach $O(s \cdot \log s)$ [S2, unofficial].
*   **Memory Complexity:** Depends on the stack size. A fixed-size stack (e.g., 100 elements) is often used to prevent stack overflow [S5, unofficial]. Auxiliary arrays for backtracking (e.g., `skipListPerDepth`) add constant overhead relative to tree depth [S2, unofficial].
*   **Strengths:** Avoids recursion limits in GPUs (HLSL) and CPU stack overflows; allows explicit control over traversal order and backtracking; efficient culling of non-intersecting nodes [S2, S4, unofficial].
*   **Limitations:** Fixed-size stack may require careful sizing; greedy backtracking can be complex to implement correctly; performance depends heavily on ray coherence and tree balance [S2, S4, unofficial].
*   **When to use:** When rendering on GPUs with recursion limits (HLSL) or when CPU stack space is constrained; for sparse volumes where culling is effective [S2, S4, unofficial].
*   **When not to use:** For very deep trees where stack size becomes prohibitive; for highly coherent rays where simpler DDA might suffice [S2, unofficial].

## Background
Iterative Octree Traversal originated as a solution to the limitations of recursive ray tracing in shader languages like HLSL, which do not support general recursion [S2, unofficial]. It was developed to enable efficient raycasting on GPUs by simulating the call stack with explicit data structures [S2, S4, unofficial]. The method builds on earlier discrete ray-tracing techniques [S3, unofficial] and has been adapted for various platforms, including CPU-based renderers [S1, unofficial] and GPU-based systems [S2, S4, unofficial]. It is widely used in real-time voxel rendering and global illumination calculations [S4, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S3] Discrete Ray-Tracing of Huge Voxel Spaces - doczz.net, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25 (unofficial)
[S4] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S5] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S6] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
