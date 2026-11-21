# Octree Insertion with Growth

Also called: Dynamic Octree Insertion with Boundary Growth

## Overview
Octree Insertion with Growth is the core algorithmic step for inserting a voxel into a Sparse Voxel Octree (SVO) while dynamically handling spatial boundaries. It solves the problem of maintaining a valid hierarchical index when a new voxel's coordinate falls outside the current octree's defined spatial extent. Unlike static voxelization methods that pre-allocate a fixed grid, this method grows the tree structure on-demand [S1, unofficial]. It is distinct from [Voxel Placement in Octree](3097-voxel-placement-in-octree.md) by explicitly managing the tree's global boundary conditions and expansion logic, whereas the sibling method focuses on the recursive descent and node expansion within a valid range [S1, S2, unofficial].

## How it works
**Inputs:**
*   $v$: The voxel to insert, containing position $p$ and data (e.g., color, bitstream).
*   $o$: The current octree, containing a root node and a current depth $d$.

**Outputs:**
*   $o'$: The updated octree with the voxel inserted and the root potentially updated if growth occurred.

**Data Structures:**
The implementation relies on the node structure defined in the parent context. A naive node representation (Fragment 1 in [S1, unofficial]) consists of:
*   `children`: An array of 8 pointers to child nodes (or null).
*   `voxelColorIndex`: An 8-bit integer storing the color index if the node is a leaf.
*   `isLeaf`: A boolean flag indicating if the node is a leaf.
*   *Note:* On a 64-bit processor, this structure consumes 72 bytes; on a 32-bit processor, 40 bytes [S1, unofficial].

**Parameters:**
*   $d$: The current depth of the octree.
*   Range Limit: The valid spatial range for depth $d$ is defined as $[-2^{d-1}, 2^{d-1})$ [S1, unofficial].

**Steps:**
1.  **Retrieve Position and Depth:** Get the position $p$ of the voxel $v$ and the current depth $d$ of the octree $o$.
2.  **Boundary Check:** Check if $p$ is within the valid range $[-2^{d-1}, 2^{d-1})$.
    *   If $p$ is **not** in this range, invoke the `grow(o)` function to expand the octree's spatial extent and update $d$.
3.  **Insertion:** Recursively insert the voxel into the octree starting from the root. This uses the voxel's bitstream to determine the path down the tree, creating intermediate nodes if they do not exist, and storing the voxel data at the leaf level corresponding to depth $d$.
4.  **Return:** Return the updated octree $o'$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def insert_with_growth(voxel: Voxel, octree: Octree) -> Octree:
    """
    Inserts a voxel into an octree, growing the tree if the voxel's position
    is outside the current valid range.
    
    Args:
        voxel: The voxel to insert, containing position and data.
        octree: The current octree structure.
        
    Returns:
        The updated octree.
    """
    p = voxel.position
    d = octree.depth
    
    # Step 2: Boundary Check
    # The valid range for depth d is [-2^(d-1), 2^(d-1))
    lower_bound = - (2 ** (d - 1))
    upper_bound = 2 ** (d - 1)
    
    if not (lower_bound <= p.x < upper_bound and
            lower_bound <= p.y < upper_bound and
            lower_bound <= p.z < upper_bound):
        # Grow the octree to encompass the new position
        octree = grow(octree)
        # Note: grow() typically updates the depth and root structure.
        # We assume grow() returns the new octree state.
        # The depth d might need to be refreshed if grow() changes it significantly,
        # but typically grow() ensures the current depth is sufficient or increases it.
        # For strict adherence to the source logic, we proceed with insertion 
        # using the potentially updated octree.
        
    # Step 3: Insertion
    # The source describes a recursive insertion using a bitstream.
    # Algorithm 4 in [S1, unofficial] details the recursive insertion:
    # insert_voxel_recursive(voxel, octree.root, voxel.bitstream, octree.depth)
    
    octree.root = insert_voxel_recursive(
        voxel=voxel,
        node=octree.root,
        bitstream=voxel.bitstream,
        depth=octree.depth
    )
    
    return octree

def insert_voxel_recursive(voxel: Voxel, node: Node, bitstream: Bitstream, depth: int) -> Node:
    """
    Recursive helper to insert a voxel into the octree at a specific depth.
    Based on Algorithm 4 from [S1, unofficial].
    """
    # Base case: if depth is 0, we are at the leaf level
    if depth == 0:
        # Create a leaf node containing the voxel
        return create_leaf_node(voxel)
    
    # Determine which child index to traverse
    # group(b, d, 3) extracts 3 bits from the bitstream corresponding to the current depth
    child_index = extract_bits(bitstream, depth, 3)
    
    # Get or create the child node
    child_node = node.children[child_index]
    if child_node is None:
        child_node = create_internal_node()
        node.children[child_index] = child_node
    
    # Recurse to the next depth level
    child_node = insert_voxel_recursive(
        voxel=voxel,
        node=child_node,
        bitstream=bitstream,
        depth=depth - 1
    )
    
    # Update the parent's child pointer
    node.children[child_index] = child_node
    
    return node

def extract_bits(bitstream: Bitstream, depth: int, num_bits: int) -> int:
    """
    Extracts num_bits from the bitstream at the position determined by depth.
    This is a placeholder for the specific bitstream packing scheme.
    [G] Typically, this involves shifting and masking based on the bitstream's 
    internal representation (e.g., Morton code or specific bit ordering).
    """
    # Implementation depends on the specific bitstream format.
    # For a standard Morton code or similar, this would extract the 
    # 3-bit coordinate offset at the current level.
    raise NotImplementedError("Bitstream extraction logic depends on specific encoding.")

def create_leaf_node(voxel: Voxel) -> Node:
    node = Node()
    node.is_leaf = True
    node.voxel_color_index = voxel.color_index
    node.children = [None] * 8
    return node

def create_internal_node() -> Node:
    node = Node()
    node.is_leaf = False
    node.children = [None] * 8
    return node

def grow(octree: Octree) -> Octree:
    """
    Expands the octree to include positions outside the current range.
    [G] The specific implementation of grow() is not detailed in [S1, unofficial] beyond 
    the call. It typically involves creating a new root with 8 children, 
    where the old root is one of the children, and the new children are 
    empty or initialized to cover the expanded range.
    """
    # This is a high-level placeholder for the growth logic.
    # See [S4, unofficial] for a detailed example of dynamic expansion logic.
    raise NotImplementedError("Growth logic implementation varies; see dynamic expansion variants.")
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for boundary checks ($p \in [-2^{d-1}, 2^{d-1})$) must handle edge cases where $p$ is exactly on the boundary. The interval is half-open, so $p = 2^{d-1}$ triggers growth, while $p = -2^{d-1}$ does not [S1, unofficial].
*   **Bitstream Consistency:** The bitstream used for insertion must be consistent with the coordinate system. If the octree grows, the bitstream interpretation for existing nodes must remain valid relative to the new root [S1, unofficial].
*   **Memory Allocation:** Creating new nodes during insertion and growth can lead to memory fragmentation. Efficient allocators or pool-based allocation are recommended [S1, unofficial].

## Subtopics

### [Octree-based Mesh Voxelization](3071-octree-based-mesh-voxelization.md)
Octree-based mesh voxelization converts a 3D triangle mesh into a sparse hierarchical voxel representation by recursively subdividing the mesh's bounding volume into eight equal sub-cubes and marking those intersecting the mesh as active. Unlike uniform grid voxelization, which wastes memory on empty space, this method builds a sparse structure where only occupied regions are stored, making it suitable for large or complex geometries. It differs from [Voxel Placement in Octree](3097-voxel-placement-in-octree.md) by focusing on the initial construction from geometry rather than dynamic insertion, and from [Dynamic Sparse Voxel Octree with Morton Codes](4099-dynamic-sparse-voxel-octree-with-morton-codes.md) by using a recursive subdivision approach rather than Morton-code-based spatial hashing for the initial fill.

### [Voxel Placement in Octree](3097-voxel-placement-in-octree.md)
Voxel Placement in Octree is the fundamental recursive operation that inserts a colored voxel into a Sparse Voxel Octree (SVO) at a specific 3D coordinate, handling tree depth limits and node expansion. It solves the problem of dynamically maintaining a hierarchical spatial index where internal nodes may be collapsed (merged into uniform leaves) and must be correctly expanded before further descent. It is distinct from its siblings: [Octree Insertion with Growth](2387-octree-insertion-with-growth.md) provides the broader context of boundary handling and tree growth, while [Dynamic Sparse Voxel Octree with Morton Codes](4099-dynamic-sparse-voxel-octree-with-morton-codes.md) focuses on linearization and cache efficiency rather than the raw recursive insertion logic.

### [Dynamic Sparse Voxel Octree with Morton Codes](4099-dynamic-sparse-voxel-octree-with-morton-codes.md)
A single-header C++ implementation of a dynamic sparse voxel octree using integer coordinates and Morton codes, designed for micro-optimized CPU-based voxel physics simulations in chunked procedural game worlds. It distinguishes itself by providing bulk allocation and deallocation operations alongside point queries, optimized for high-throughput voxel manipulation rather than just static storage. It builds on [Octree Insertion with Growth](2387-octree-insertion-with-growth.md) by providing the concrete data structures and Morton code integration required for efficient spatial indexing.

### [Octree Node Insertion and Parent Finding](12106-octree-node-insertion-and-parent-finding.md)
This method traverses a sparse voxel octree to locate or create the parent node for a specific coordinate $(x,y,z)$ at a given detail level, effectively growing the tree structure dynamically as needed. It differs from static voxelization by allocating nodes on-demand during insertion rather than pre-allocating a fixed grid, and it serves as the foundational traversal logic for the broader [Octree Insertion with Growth](2387-octree-insertion-with-growth.md) process.

### [Sparse Voxel Octree Layer Construction](14731-sparse-voxel-octree-layer-construction.md)
Sparse Voxel Octree Layer Construction constructs the next hierarchical level of a sparse voxel octree by grouping eight child nodes from the current layer into a single parent node. It differs from sibling methods like [Octree Insertion with Growth](2387-octree-insertion-with-growth.md) by operating on bulk layers rather than individual dynamic insertions, and from [Voxel Placement in Octree](3097-voxel-placement-in-octree.md) by handling the structural promotion of existing nodes rather than initial placement. This method is essential for building the tree structure from leaf nodes upward.

### Octree Building Pass
*   **Definition:** Builds the octree from voxel fragments, averaging leaf node values when multiple voxels fall into the same node [S2, unofficial].
*   **Steps:** Build octree using steps described in OpenGL Insights Chapter. Average octree leaf node values when different voxels fall into the same node [S2, unofficial].
*   **Layout:**
    *   **Octree node buffer:** Stores octree/voxel node data. 8 nodes are grouped into 1 tile. Each octree node is 28 bytes: first 4 bytes store child tile index, next 24 bytes store neighbor node byte offset from start of octree node buffer [S2, unofficial].
    *   **Octree leaf node:** Stores voxel data directly, using 16 bytes. First 4 bytes store bit flag indicating if voxel is from static mesh (not overwritten by dynamic). 1 byte counter stored with voxel normal for averaging when different voxel fragments fall into same voxel [S2, unofficial].
    *   **3D texture:** Stores reflected radiance and alpha from direct lighting at that voxel. Assumes all surface reflect light diffusely, so only 1 3D texture is used (different from paper storing incoming radiance). Indexed by child node tile index [S2, unofficial].

### Surfel Addition to Octree
*   **Definition:** Adds a surfel to the octree, recursively descending to leaf nodes, and subdividing if the leaf exceeds MAX_SURFEL_COUNT [S3, unofficial].
*   **Steps:**
    1.  Check if surfel is inside node using `inside(obj)`; if not, return 0 [S3, unofficial].
    2.  If node has children, loop over 8 children, recursively call add on each, sum results, increment `_surfelCount`, and return sum [S3, unofficial].

### Dynamic Octree Expansion
*   **Steps:**
    1.  Call `EnlargeCorners(dir)` to compute `newCornerIdx`, `oldCornerIdx`, and offset based on the expansion direction [S4, unofficial].
    2.  Initialize an array of 8 child pointers (`rootChildren`) to `nullptr`, then assign the current root node to `rootChildren[oldCornerIdx]` [S4, unofficial].
    3.  Create a new empty octree node at `rootChildren[newCornerIdx]` with size `m_size`, position `m_position + offset`, and depth 1 [S4, unofficial].
    4.  Construct a new `OctreeChildren` struct with a bitmask containing only the old and new children, and the `rootChildren` array [S4, unofficial].
    5.  If expanding in a negative direction (xminus, yminus, or zminus), update the root position by adding the offset [S4, unofficial].
    6.  Double the octree size (`m_size *= 2`) and replace the root node with a new Octree initialized with the new children, updated size, and updated position [S4, unofficial].
*   **Pitfall:** The offset calculation for `xplus` uses `glm::vec3(m_size, 0, 0)` but is marked as unused in a comment, indicating a potential bug or placeholder that should be zeroed or properly utilized [S4, unofficial].

### Octree Node Management
*   **Definition:** Methods to add or remove blocks (filled cells) in a quadtree, automatically splitting full nodes into children or merging empty children into a full/empty node [S5, unofficial].
*   **Steps:**
    1.  In `add_block`, if node is full, return immediately [S5, unofficial].
    2.  If node size is 1, set `is_full = true` and return [S5, unofficial].
    3.  If node is a leaf (no children) and not full, split it by calling `devide(false)` (note typo in source), then recursively add to the correct child based on relative position [S5, unofficial].
    4.  Determine child index for position `pos`: calculate `rel_pos = pos - self.position`. Child 0 if `rel_pos.x < size/2 && rel_pos.y < size/2`. Child 1 if `rel_pos.x < size && rel_pos.y < size/2`. Child 2 if `rel_pos.x < size/2 && rel_pos.y < size`. Child 3 otherwise [S5, unofficial].
    5.  If node has children, recursively add to the correct child. After recursion, check if all children are full. If so, merge: set `is_full = true`, `children = None` [S5, unofficial].
    6.  If node size is 1, set `is_full = false` and return [S5, unofficial].
*   **Layout:** `QuadtreeNode` struct: `is_full: bool`, `children: Option<[Box<QuadtreeNode>;4]>`, `size: i32`, `position: IVec2` [S5, unofficial].

### Variable Error Octree Construction
*   **Definition:** An octree construction strategy that uses a variable error bound depending on the distance to the surface, reducing memory consumption far from the surface while maintaining accuracy near it [S6].
*   **Steps:** During sphere marching, move forward using $f(x)(1 - \delta)$ to ensure the step size is an underestimation of the real distance, preventing skipping the isosurface [S6].
*   **Formula:** The maximum allowed error is redefined as $\epsilon + \delta |f(x)|$, where $\epsilon$ is the base error, $f$ is the input distance function, $x$ is the current position, and $\delta$ is the hyperparameter [S6].
*   **Parameter:** Hyperparameter $\delta$ controls the variable error term. Typical value: 0.1 [S6].

### Voxel Tree Injection
*   **Definition:** Deterministically places voxel trees into chunks by stamping tree blocks, ensuring cross-chunk safety and gameplay consistency [S7, unofficial].
*   **Steps:** Compute tree positions for the chunk and neighboring chunks within tree extent range. Filter by per-template placement rules (material, biome, slope, elevation). Stamp tree blocks only within the chunk's voxel bounds [S7, unofficial].
*   **Layout:** `FVoxelTreeInstance`: BaseVoxel (FIntVector), TemplateIndex (int32), Seed (uint32). `FVoxelTreeVoxel`: GlobalVoxel (FIntVector), MaterialID (uint8), bTrunk (bool) [S7, unofficial].

### OBJ Voxelization
*   **Definition:** Converts a Wavefront .obj mesh into a set of voxels and inserts them into a SparseVoxelOctree [S8, unofficial].
*   **Layout:** The OBJ loader exposes vertex positions via `obj.data.position`, an array of 3D coordinates. Polygon vertex indices are stored in `poly.0` as a list of index tuples [S8, unofficial].
*   **Code:** Fan triangulation iterates over polygon vertices, constructing triangles by connecting the first vertex to each subsequent pair, and pushes them to a triangle buffer [S8, unofficial].
*   **Pitfalls:**
    *   Fan triangulation assumes all input polygons are convex; non-convex polygons will produce incorrect, intersecting triangles or holes [S8, unofficial].
    *   Polygons with fewer than three vertices are invalid and must be skipped to prevent indexing errors [S8, unofficial].
    *   Vertex indices in polygon definitions must be validated against the position array length to avoid out-of-bounds memory access [S8, unofficial].
    *   Large OBJ files can consume significant memory if all triangles are collected before voxelization; batch processing or streaming is recommended for production [S8, unofficial].
*   **Usage:** Implemented in `client/src/plugins/environment/systems/voxels/octree.rs` as a public method on `SparseVoxelOctree` [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(d)$ where $d$ is the depth of the octree, assuming constant-time node creation and bitstream extraction. Growth operations may involve restructuring the root, which is $O(1)$ if only the root is updated, but may require updating references if the tree is shared.
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty nodes. The naive node structure uses 72 bytes (64-bit) or 40 bytes (32-bit) per node [S1, unofficial]. More compact representations (e.g., 28 bytes per node in tile-based structures) can reduce this overhead [S2, unofficial].
*   **Scaling:** The method scales well with sparse data, as it only allocates nodes for occupied regions. However, frequent growth operations can lead to memory fragmentation if not managed with a pool allocator.
*   **Strengths:** Dynamic boundary handling allows for arbitrary scene sizes without pre-allocation. Simple recursive structure is easy to implement.
*   **Limitations:** Naive node structures have high memory overhead. Bitstream management can be complex. Growth operations may invalidate existing pointers if not handled carefully.
*   **When to Use:** Suitable for dynamic scenes where the spatial extent is unknown or changes frequently.
*   **When Not to Use:** Not suitable for static scenes with known bounds where a fixed grid or pre-allocated octree would be more memory-efficient.

## Background
The concept of dynamic octree insertion with growth is rooted in the need for efficient spatial indexing in computer graphics, particularly for global illumination techniques like voxel cone tracing. The specific algorithm described in [S1, unofficial] originates from a thesis on Sparse Voxel Octree compression and efficiency for global illumination calculation [S1, unofficial]. The approach of growing the octree dynamically to handle boundary conditions is a common technique in dynamic scene rendering, allowing for the integration of new geometry without rebuilding the entire structure. Variants of this method are used in various rendering engines and simulation tools, often optimized with Morton codes or tile-based structures for better cache performance [S2, S4, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
[S3] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S4] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S5] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
[S6] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
[S7] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S8] Add OBJ voxelization helper, https://github.com/eliasstepanik/voxel-simulation/pull/47, fetched 2026-10-26 (unofficial)
