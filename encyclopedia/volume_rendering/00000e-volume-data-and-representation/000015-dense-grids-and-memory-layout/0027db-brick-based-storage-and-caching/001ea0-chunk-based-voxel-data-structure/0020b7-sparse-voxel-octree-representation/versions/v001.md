# Sparse Voxel Octree Representation

Also called: Basic SVO in rust, Octree Representation, Sparse Voxel Octree (SVO), Sparse Voxel Octree (SVO) - Basic Implementation, Voxel Octrees, pointer-based sparse octree

## Overview
A Sparse Voxel Octree (SVO) is a hierarchical data structure for representing 3D space that eliminates sparse memory by not storing empty volumes [S1, S4, S5, unofficial]. It enables efficient storage and traversal of voxel data, particularly for indirect illumination calculations at geometry-independent costs [S2]. Unlike dense grids or chunk-based structures, it adapts resolution to content density, trading traversal performance for memory efficiency [S4, S5, unofficial].

## How it works
**Inputs:** 3D coordinates $(x, y, z)$, voxel data, and a maximum depth $D$.
**Outputs:** Voxel reference or null.

**Data Structures:**
1.  **SparseVoxelOctree:** Contains `root` (Node) and `max_depth` (u32) [S1, unofficial].
2.  **Node:** Enum with two variants [S1, unofficial]:
    *   `Branch`: Contains `children: Box<[Node; 8]>`. Stores exactly 8 children in contiguous memory [S1, unofficial].
    *   `Leaf`: Contains `Option<Voxel>`. Represents a unitary-sized voxel or empty space [S1, S4, unofficial].
3.  **Voxel:** Contains `index` (u32) [S1, unofficial].

**Initialization:**
The tree is initialized with a specific depth $D$. A `Node::new(depth)` recursively creates a full tree where depth 0 results in a `Leaf(Some(Voxel::new()))` and depth > 0 results in a `Branch` with 8 children of depth-1 [S1, unofficial].

**Lookup (Iterative):**
1.  Start with `this = self`, coordinates $x, y, z$, and size $s = 2^{D-1}$ [S1, unofficial].
2.  Loop:
    *   If `this` is `Leaf`, return the voxel reference [S1, unofficial].
    *   If `this` is `Branch`, calculate child index:
        $$ index = ((x \ge s) \text{ as usize}) \ll 0 \mid ((y \ge s) \text{ as usize}) \ll 1 \mid ((z \ge s) \text{ as usize}) \ll 2 $$
    *   Update `this` to `children[index]` [S1, unofficial].
    *   Update coordinates: $x \%= s, y \%= s, z \%= s$ [S1, unofficial].
    *   Update size: $s /= 2$ [S1, unofficial].

**Lookup (Recursive):**
1.  If `self` is `Leaf`, return `voxel.as_ref()` [S1, unofficial].
2.  If `self` is `Branch`, calculate index:
    $$ index = ((x \ge size) \text{ as usize}) \ll 0 \mid ((y \ge size) \text{ as usize}) \ll 1 \mid ((z \ge size) \text{ as usize}) \ll 2 $$
3.  Recursively call `children[index].get(x % size, y % size, z % size, size / 2)` [S1, unofficial].

**Insertion:**
1.  If `depth` is 0, replace `self` with the new node [S1, unofficial].
2.  If `self` is `Branch`, calculate index:
    $$ index = ((x \ge size) \text{ as usize}) \ll 0 \mid ((y \ge size) \text{ as usize}) \ll 1 \mid ((z \ge size) \text{ as usize}) \ll 2 $$
3.  Recursively insert into `children[index]` with $x \% size, y \% size, z \% size, size / 2$, and $depth - 1$ [S1, unofficial].
4.  **Leaf to Branch Conversion:** If `self` is `Leaf`, clone the existing voxel. Replace `self` with a new `Branch` node where all 8 children are `Leaves` containing the cloned voxel. Then recursively call `insert` on the new `Branch` node with the same $x, y, z$, node, size, and depth [S1, unofficial]. This ensures no data is lost during the split [S1, unofficial].

**GPU Representation (Crassin [S2]):**
*   Nodes are stored in linear GPU memory, grouped into $2 \times 2 \times 2$ tiles. A single pointer stores all 8 children [S2].
*   Each node contains a pointer to a $3 \times 3 \times 3$ voxel brick in texture memory [S2].
*   Voxel centers are located at node corners, not centers, allowing interpolated values to be computed inside a brick covering $2 \times 2 \times 2$ nodes [S2].
*   This method uses less than half the memory of border-duplication methods [S2].
*   Neighbor pointers to spatially neighboring nodes and the parent are included [S2].

**Construction:**
*   Rasterize the mesh three times along the three main axes with viewport resolution corresponding to the maximum octree subdivision level (e.g., $512 \times 512$ for a $512^3$ octree) [S2].
*   Disable depth test to generate fragment shader threads for each potentially existing leaf node [S2].
*   Traverse the octree top-to-bottom and subdivide when needed [S2].
*   Write surface attributes (color, normal, material) to the leaf node [S2].

**Parameters:**
*   Viewport resolution for rasterization corresponds to the maximum level of subdivision, typically $512 \times 512$ pixels for a $512^3$ octree [S2].

**Pitfalls:**
*   Traversal involves overhead on conventional computers [S3, unofficial].
*   Octree encoding achieves less data compression with grey-level data than with binary data [S3, unofficial].
*   Maps poorly to GPU memory model; cannot benefit from GPU texture caching and filtering hardware in basic implementations [S4, unofficial].
*   Empty nodes must be collapsed to gain memory [S2].

## Subtopics

### [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md)
Stores filtered scene information (energy intensity, direction, occlusion, NDFs) with increasing precision by descending the hierarchy. Chosen for efficient indirect illumination.

### [Naive Octree Node Representation](2445-naive-octree-node-representation.md)
Direct memory representation using pointers for children and separate fields for leaf status and color. Chosen for simplicity.

### [SVRaster](2464-svraster.md)
Real-time, high-fidelity novel-view synthesis using explicit, neural-free sparse voxels and a custom rasterizer. Chosen for real-time rendering without neural networks.

### [Dynamic Update Optimization](4120-dynamic-update-optimization.md)
Optimizes the dynamic update of the sparse voxel structure. Chosen for scenes with moving parts.

## Operating characteristics
*   **Time Complexity:** Lookup and insertion require tree traversal, which is less efficient than direct index access in 3D textures [S4, unofficial]. Traversal overhead exists on conventional computers [S3, unofficial].
*   **Memory Complexity:** Eliminates sparse memory by not storing empty volumes [S4, unofficial]. Uses less than half the memory of border-duplication methods for the same sampling precision [S2].
*   **Performance:** Enables real-time frame rates for indirect illumination in complex scenes [S2]. Performance trade-off due to tree traversal [S5, unofficial].
*   **Strengths:** Effective memory use for sparse volumes [S5, unofficial]. Geometry-independent cost for indirect illumination [S2].
*   **Limitations:** Poor mapping to GPU memory model in basic implementations [S4, unofficial]. Less compression for grey-level data [S3, unofficial].
*   **When to use:** Scenes with large empty spaces or varying density where memory is a constraint [S5, unofficial].
*   **When not to use:** When direct texture access performance is critical and memory is not [S4, unofficial].

## Background
*   **Origin:** The concept extends the quadtree representation of 2D images to 3D [S3, unofficial].
*   **History:** Crassin presented a GPU implementation in his paper on voxel cone tracing global illumination [S4, S5, unofficial]. The structure is used in [LK10] and [CNLE09] [S2].
*   **Usage:** Used for indirect illumination, global illumination, and novel-view synthesis [S2][S4, S5, unofficial].

## Sources
[S1] HexExecute/sparse-voxel-octree, https://github.com/HexExecute/sparse-voxel-octree, fetched 2026-10-26 (unofficial)
[S2] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S3] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S4] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S5] Voxel Cone Traced Global Illumination - Leif Node, https://leifnode.com/2015/05/voxel-cone-traced-global-illumination/, fetched 2026-10-27 (unofficial)
