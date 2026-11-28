# SparseVoxelOctree


## Overview
SparseVoxelOctree (SVO) is a hierarchical data structure for representing 3D volumetric data, designed to efficiently store and access sparse voxel grids by omitting empty space [S1, unofficial]. It solves the memory inefficiency of uniform voxel grids by using an octree topology where only occupied nodes are stored [S1, unofficial]. Unlike its parent [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md) which defines the general tree concept, SVO specifically refers to the implementation using octal branching and specific memory layouts (such as flat arrays or hash maps) to enable fast traversal [S1, unofficial]. It is distinct from [GigaVoxels Data Synthesis](16702-gigavoxels-data-synthesis.md) which focuses on cache-friendly data synthesis for large scenes, and [Sparse Voxel Tree Data Layout](21134-sparse-voxel-tree-data-layout.md) which details specific storage formats.

## How it works
**Inputs:** A 3D voxel grid defined by `surfaceVoxelGridSize` (an `Eigen::Vector3i` representing dimensions in voxels) and a set of occupied voxel positions with associated data (e.g., color, density) [S1, unofficial].
**Outputs:** A serialized or in-memory representation of the octree, typically consisting of a node array and metadata [S1, unofficial].

**Data Structures:**
The core structure is defined by the `SparseVoxelOctree` class, which inherits from `BaseModel` [S1, unofficial].
*   `treeDepth`: `int`, stores the maximum depth of the tree (default 0) [S1, unofficial].
*   `numTreeNodes`: `size_t`, total count of nodes in the structure [S1, unofficial].
*   `surfaceVoxelGridSize`: `Eigen::Vector3i`, dimensions of the initial voxel grid [S1, unofficial].
*   `depthNumNodes`: `vector<size_t>`, stores the number of nodes at each tree depth level [S1, unofficial].
*   `SVONodes`: `vector<vector<SVONode>>`, a hierarchical container where each inner vector holds nodes for a specific depth [S1, unofficial].
*   `svoNodeArray`: `vector<SVONode>`, a flat array representation of all nodes [S1, unofficial].
*   `nodeVertexArray`: `vector<thrust::pair<Eigen::Vector3f, uint32_t>>`, stores vertex positions and indices [S1, unofficial].
*   `nodeEdgeArray`: `vector<thrust::pair<thrust_edge, uint32_t>>`, stores edge definitions (`thrust_edge` is a pair of `Eigen::Vector3f`) and indices [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize `treeDepth` to 0 and `numTreeNodes` to 0. Set `surfaceVoxelGridSize` to the input grid dimensions [S1, unofficial].
2.  **Tree Construction:** Traverse the input voxel grid. For each occupied voxel, insert it into the octree. If a node at the current depth has no children, create child nodes for its 8 octants. Mark octants containing data as occupied [S1, unofficial].
3.  **Node Storage:** Store each created node in `SVONodes` organized by depth and in `svoNodeArray` as a flat list [S1, unofficial].
4.  **Metadata Update:** Increment `numTreeNodes` for each node created. Update `depthNumNodes` to reflect the count of nodes at each depth level [S1, unofficial].
5.  **Vertex/Edge Extraction (Optional):** For rendering or mesh extraction, iterate through leaf nodes. Extract vertex positions (`Eigen::Vector3f`) and associate them with node indices (`uint32_t`) in `nodeVertexArray`. Extract edges (`thrust_edge`) and associate with indices in `nodeEdgeArray` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SparseVoxelOctree:
    def __init__(self, grid_size: Vector3i):
        self.treeDepth = 0
        self.numTreeNodes = 0
        self.surfaceVoxelGridSize = grid_size
        self.depthNumNodes = []
        self.SVONodes = []
        self.svoNodeArray = []
        self.nodeVertexArray = []
        self.nodeEdgeArray = []

    def insert_voxel(self, position: Vector3i, data: any):
        # [G] Helper to convert voxel position to tree path
        path = self._voxel_to_path(position)
        self._insert_recursive(path, 0, data)

    def _insert_recursive(self, path: list, depth: int, data: any):
        if depth == len(path):
            # Leaf node
            node = SVONode(data=data, children=[None]*8)
            self.svoNodeArray.append(node)
            self.numTreeNodes += 1
            # Ensure depthNumNodes has entry for this depth
            while len(self.depthNumNodes) <= depth:
                self.depthNumNodes.append(0)
            self.depthNumNodes[depth] += 1
            return node
        else:
            # Internal node logic would go here
            # [G] In a full implementation, this manages child pointers
            pass

    def _voxel_to_path(self, voxel: Vector3i) -> list:
        # [G] Converts 3D voxel coordinates to a sequence of octant indices (0-7)
        # representing the path from root to leaf.
        # Implementation depends on specific encoding (e.g., Morton code or direct traversal).
        # Not given in the sources.
        return []
```

**Parameters:**
*   `treeDepth`: Not given in the sources.
*   `surfaceVoxelGridSize`: Determined by input data resolution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point vertex positions in `nodeVertexArray` may suffer from precision loss if not handled carefully during extraction [S1, unofficial].
*   **Boundaries:** The `surfaceVoxelGridSize` defines the extent; queries outside this range are invalid [S1, unofficial].
*   **Degenerate Input:** Empty input grids result in a tree with `numTreeNodes = 0` [S1, unofficial].

## Subtopics

### [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md)
A Morton-coded Sparse Voxel Tree (SVO) is a flat-array sparse voxel tree that uses Morton codes (Z-order curve) for spatial ordering and an 8-bit mask per node to track children. It avoids recursive pointers by storing nodes in contiguous blocks of 8 elements, indexed via `node_format::_block_index`. Unlike its sibling [Linear Octree with Z-order curve (Morton tree)](7054-linear-octree-with-z-order-curve-morton-tree.md) which flattens the tree to store only relevant data trading traversal speed for memory efficiency, this method retains a hierarchical mask structure.

### [Hashed-Octree](7486-hashed-octree.md)
The Hashed-Octree is a sparse voxel representation that combines octrees with hash tables to provide constant-time data access complexity for huge 3D environments, addressing the $O(d)$ access time of standard Sparse Voxel Octrees (SVOs). It eliminates the need for explicit pointers by storing octants in a hash map using a unique 53-bit locational code key, allowing intermediate octants to be omitted. Unlike its siblings which may rely on Morton codes or pointer trees, this method uses a direct hash of coordinate-aligned root keys for top-level access, reducing data access complexity from $O(d)$ to $O(d_b)$ where $d_b$ is the depth of internal octrees (typically $d=1$).

### [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md)
The Spatio-Temporal Sparse Voxel Octree (STSVO) is a hierarchical data structure for representing the geometry of time-variable 3D scenes by treating the scene as a 4D static space (3 spatial dimensions + 1 time dimension). It uses a single SVO-like structure instead of a set of separate SVOs for each time frame. Compared to storing each time frame as a separate SVO (SVO set), STSVO provides a more compact binary representation, achieving 1.73 to 2.08 times more compactness.

### [n-Dimensional Sparse Voxel Tree (nD SVT)](7999-n-dimensional-sparse-voxel-tree-nd-svt.md)
The n-Dimensional Sparse Voxel Tree (nD SVT) is a hierarchical data structure that generalizes Sparse Voxel Octrees (SVOs) to $n$-dimensional grids, where $n > 3$, to achieve a more compact binary representation of voxelized scenes. It solves the problem of storage efficiency in 3D volume rendering by upscaling the dimensionality of the scene from 3D to $m$D ($m > 3$), reducing the total number of nodes and pointers required to represent the geometry. Unlike its sibling [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md) which remains strictly 3D, or [Hashed-Octree](7486-hashed-octree.md) which uses hashing for flat storage, the nD SVT trades increased per-node complexity (larger child masks) for a significantly reduced tree depth and node count.

### [Sparse Voxel Octree Raycasting](21492-sparse-voxel-octree-raycasting.md)
Sparse Voxel Octree (SVO) Raycasting is a hierarchical traversal algorithm designed to efficiently intersect rays with sparse voxel data structures, primarily for 3D navigation and visibility analysis. Unlike uniform grid raycasting, it skips empty space by traversing only occupied nodes from the highest layer down to the leaves. It distinguishes itself by handling arbitrary ray directions through a coordinate reflection technique, ensuring the traversal logic remains consistent regardless of the ray's orientation relative to the world axes.

#### Sparse Octree Raycasting Rendering
A rendering technique using raycasting implemented in HLSL to find which voxel occupies each pixel in a sparse octree structure, setting pixel color based on node data [S2, unofficial]. The process involves building a sparse octree from a modeling tool where each cubical non-empty leaf node is a voxel [S2, unofficial]. At render time, an HLSL raycaster determines which voxel occupies each pixel, and the pixel color is set to the value stored in the corresponding node, calculated by averaging texture values from the source model during tree generation [S2, unofficial]. A major pitfall is that voxel raytracing imposes a huge drain on the GPU due to the arithmetic and conditionals necessary, which are costly compared to screen-space effects [S2, unofficial]. Multiple bounces for radiosity are not viable for scenes with millions or billions of voxels, as each ray segment takes considerable effort to bounce and composite [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Access time for standard SVOs is $O(d)$ where $d$ is the depth [S1, unofficial]. Hashed-Octree reduces this to $O(d_b)$ where $d_b$ is typically 1 [S1, unofficial].
*   **Memory Complexity:** Depends on the layout. Flat arrays with Morton codes are memory efficient [S1, unofficial]. Hashed-Octrees use hash tables which may have overhead but allow omission of intermediate nodes [S2, unofficial].
*   **Performance:** Voxel raycasting is computationally expensive on GPUs due to arithmetic and conditionals [S2, unofficial].
*   **Strengths:** Collision detection, variable LOD, view space culling [S2, unofficial]. Compact representation for time-variable scenes (STSVO) [S1, unofficial].
*   **Limitations:** High GPU cost for raycasting [S2, unofficial]. Not viable for multiple bounces in large scenes [S2, unofficial].

## Background
The SparseVoxelOctree structure is used in voxel-based terrain engines and 3D navigation systems [S2, unofficial]. It was discussed in the context of generating environments from modeling tools like 3DSMAX [S2, unofficial]. The specific implementation details (class structure, LUTs for Morton decoding) are found in the GitHub repository `Alan-Leo-Wong/GPU-SVO` [S1, unofficial].

## Sources
[S1] GitHub - Alan-Leo-Wong/GPU-SVO, https://github.com/Alan-Leo-Wong/GPU-SVO, fetched 2026-10-29 (unofficial)
[S2] How are voxel terrain engines made?, https://gamedev.stackexchange.com/questions/16563/how-are-voxel-terrain-engines-made, fetched 2026-10-27 (unofficial)
