# Sparse Voxel Octree (SVO)

Also called: Efficient Sparse Voxel Octree (ESVO), Pointerless SVO

## Overview
A Sparse Voxel Octree (SVO) is a hierarchical spatial subdivision data structure that stores voxel data sparsely by only retaining nodes containing geometry, enabling efficient ray-traced rendering and Level-of-Detail (LoD) management [S2, S3, S4, unofficial]. It distinguishes itself from dense grids by eliminating empty space overhead and from KD-trees by providing uniform subdivision at every level, which ensures smoothly-degrading LoD and predictable processing costs [S2, unofficial]. Unlike sibling methods like [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md) which optimize for cache linearization via flattening, the standard SVO prioritizes tight compression of sparse spaces and direct pointer-based traversal [S2, unofficial].

## How it works
**Inputs:** A 3D volume or signed distance field (SDF) representing geometry.
**Outputs:** A tree structure where internal nodes define spatial hierarchy and leaf nodes (or nodes with valid children) store voxel data.

**Data Structures:**
The SVO is composed of nodes. Each node contains:
1.  **Valid Mask:** A bitmask indicating which of the 8 children contain geometry. If the mask is 0, the node is a leaf (empty). If the mask is non-zero, the node has children.
2.  **Child Pointer:** A pointer (or index) to the first child in a contiguous list of children that actually contain geometry. The number of allocated children equals the number of set bits in the valid mask [S3, S5, unofficial].
3.  **Voxel Data (Leaf/Content):** For leaf nodes or nodes storing surface data, this includes:
    *   **SDF Value:** Stored as an 8-bit signed normalized integer (snorm). A value of 1 encodes 0.5 interior voxel diagonals. Values are clamped to $[-1, 1]$ [S1, unofficial].
    *   **Contours (Optional):** Two parallel planes bounding the surface within the voxel, used as tighter-fitting bounding volumes for ray intersection [S5, unofficial].

**Layout Details:**
*   Nodes are stored compactly. The bitmasks describe the spatial relation between parent and children [S5, unofficial].
*   The tree is often split into 'blocks' with relative pointers to support out-of-core streaming [S5, unofficial].
*   Leaf voxels are not stored directly as separate entities; their state is implied by the parent's valid mask and the absence of further children [S5, unofficial].

**Algorithm: Construction**
1.  Initialize an empty root node.
2.  For each voxel in the input volume:
    a. Determine the path from the root to the leaf corresponding to the voxel's position.
    b. If the path does not exist, create intermediate nodes as needed.
    c. At the leaf level, store the voxel data (SDF/Color).
    d. Update the `valid_mask` of all ancestor nodes along the path to include the new child.
    e. Update the `child_pointer` of ancestors if the new child is the first allocated child for that parent.
3.  Terminate the tree at any node where all potential leaf nodes contain the same data (all empty or all identical) [S4, unofficial].

**Algorithm: Ray Traversal (Shader Implementation)**
Traversal occurs entirely in shader code [S1, unofficial]. The index of the next node is found by flipping bits of the current node index based on ray direction [S5, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_svo(ray_origin, ray_direction, root_index, max_depth):
    # Stack to maintain the path from root to current node
    path_stack = [root_index]
    current_node_index = root_index
    
    # Initial bounds check (optional, depends on implementation)
    if not is_node_valid(current_node_index):
        return None

    for depth in range(max_depth):
        # 1. Get node data
        node = get_node_data(current_node_index)
        
        # 2. Check if leaf or has children
        if node.valid_mask == 0:
            # Leaf node: perform intersection test
            # If contours are stored, use them for tighter bounds
            hit = ray_intersect_voxel(ray_origin, ray_direction, node.data)
            if hit:
                return hit
            else:
                # Backtrack
                if not path_stack:
                    return None
                current_node_index = path_stack.pop()
                continue
        
        # 3. Determine which children to visit
        # Calculate child indices based on ray direction and current position
        # The index of the next node can be found by flipping bits of the current node index
        # depending on the ray direction [S5, unofficial]
        
        # Identify children intersected by ray
        intersected_children = get_intersected_children(current_node_index, ray_origin, ray_direction)
        
        # Filter by valid mask
        candidates = intersected_children & node.valid_mask
        
        if candidates == 0:
            # No valid children in this direction
            if not path_stack:
                return None
            current_node_index = path_stack.pop()
            continue
            
        # Push current node to stack before descending
        path_stack.append(current_node_index)
        
        # Select the first candidate child (or process all if needed for transparency)
        # Typically, we pick the closest child along the ray
        next_child_index = get_first_set_bit_index(candidates)
        
        # Update ray origin for next step (optional, depending on traversal style)
        # In many SVO implementations, the ray is kept global, and node indices encode position
        current_node_index = next_child_index

    return None
```

**Parameters:**
*   **SDF Precision:** 8-bit snorm [S1, unofficial].
*   **Subdivision Ratio:** 1:2 per axis (8 children per parent) [S2, unofficial].
*   **Max Depth:** Determined by required resolution and memory constraints.

**Edge Cases and Pitfalls:**
*   **Interpolation Artifacts:** Standard SVO lacks interpolation and filtering, causing popping artifacts when transitioning between levels and aliasing at distance or close to the viewpoint [S3, unofficial]. This is often mitigated by screen-space blurring or merging LODs via a stitching algorithm to achieve C0-continuity [S3, unofficial].
*   **Dynamic Updates:** Adding, deleting, or modifying blocks is expensive, potentially requiring a large part of the octree to be rebuilt [S4, unofficial].
*   **Cache Friendliness:** Octrees are harder to make cache-friendly compared to flat structures [S2, unofficial].
*   **Static Geometry Assumption:** Calculating contours adds overhead to construction speed, making the standard SVO better suited for static geometry [S5, unofficial].

## Subtopics

### [Voctopus Sparse Voxel Octree](11000-voctopus-sparse-voxel-octree.md)
A compact, flat-memory SVO optimized for WebGL/JavaScript traversal using pointer encoding and small kernels.

### [Sparse Voxel Octree (SVO) Spatial Partitioning and LOD Management](12070-sparse-voxel-octree-svo-spatial-partitioning-and-lod-managem.md)
Replaces flat chunk storage with hierarchical SVO for dynamic LOD selection based on camera distance.

### [Sparse Voxel Octree (SVO) GPU Implementation](12570-sparse-voxel-octree-svo-gpu-implementation.md)
Implements SVO on GPU using linked nodes in linear memory (2x2x2 tiles) and a 3D texture brick pool for interpolation.

## Operating characteristics
*   **Time Complexity:** Traversal is $O(\log N)$ relative to the number of voxels, but depends heavily on the depth of the tree and the density of geometry [S2, unofficial].
*   **Memory Complexity:** Highly efficient for sparse spaces, storing only nodes with geometry [S2, unofficial]. However, a complete octree of a certain depth can still consume significant memory due to node overhead [S2, unofficial].
*   **Performance:** Fast rendering via raytracing due to tight bounding volumes (contours) and efficient bit-flip traversal [S5, unofficial]. Slower construction and modification compared to flat grids [S4, S5, unofficial].
*   **Strengths:** Tight compression of sparse spaces, smooth LoD degradation, efficient ray-casting acceleration [S2, unofficial].
*   **Limitations:** Poor cache locality, complex implementation, expensive dynamic updates, interpolation artifacts without additional techniques [S2, S3, S4, unofficial].
*   **When to Use:** Static or semi-static large-scale voxel scenes requiring high-quality rendering and LoD [S4, S5, unofficial].
*   **When Not to Use:** Highly dynamic scenes requiring frequent updates, or when cache-friendly linear traversal is critical [S2, S4, unofficial].

## Background
The Sparse Voxel Octree was popularized by Crassin, Neyret, Lefebvre, and Eisemann in 2009 with GigaVoxels, which exploited SVOs for ray-guided streaming [S4, unofficial]. The "Efficient Sparse Voxel Octree" (ESVO) implementation details were further refined by Laine and Karras in 2010, focusing on memory efficiency and shader-based traversal [S1, S5, unofficial]. It has been used in projects like EverQuest Next [S4, unofficial], but is not used in Minecraft [S4, unofficial].

## Sources
[S1] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S2] When would you use an octree in a voxel Engine?, https://gamedev.stackexchange.com/questions/70831/when-would-you-use-an-octree-in-a-voxel-engine, fetched 2026-10-26 (unofficial)
[S3] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S4] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S5] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
