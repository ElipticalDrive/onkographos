# Octree
Also called: Octree Storage, Sparse Voxel Octree (SVO) [S1, unofficial]
## Overview
An octree is a hierarchical spatial partitioning structure that recursively subdivides 3D space into eight child nodes to efficiently represent sparse volumetric data [S1, unofficial][S2][S3, unofficial]. It solves the memory inefficiency of dense voxel grids by allocating storage only to occupied regions, omitting unoccupied branches entirely [S3, S6, unofficial]. Unlike KD-trees, which use arbitrary splitting planes, octrees maintain a regular, axis-aligned cubic subdivision, making them map directly to regular-spaced grid data and providing predictable memory consumption [S1, S5, unofficial]. This regularity supports efficient multi-resolution traversal and mipmapping, which is critical for rendering large-scale scenes or distant views without traversing excessive tree depths [S5, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of volumetric data (voxels) defined within a bounding volume, typically a cube. Each voxel has a position and attributes (e.g., density, color, SDF value) [G].
*   **Output:** A tree structure where internal nodes represent subdivided space and leaf nodes store the actual voxel data or pointers to it [G].

**Data Structures:**
The octree is a non-balanced tree where each node has up to 8 children [S1, unofficial].
*   **Node Layout:** A node typically contains:
    *   A bounding box (defined by min/max coordinates or center/size) [G].
    *   An array of 8 pointers indicating the presence of children [S1, S5, unofficial].
    *   If a leaf node: the voxel data [S6, unofficial].
    *   Optional: A pointer to the parent node to facilitate upward traversal and neighbor finding [S5, S6, unofficial].

**Formulas:**
The subdivision split points are calculated as $x_{mid} = (x_{min} + x_{max}) / 2$, $y_{mid} = (y_{min} + y_{max}) / 2$, $z_{mid} = (z_{min} + z_{max}) / 2$, where $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ define the current node's bounding box bounds [S1, unofficial][G].

**Algorithm Steps:**
1.  **Initialization:** Define the root node representing the entire bounding volume of the scene [S2].
2.  **Subdivision:** Recursively subdivide the 3D space into 8 equal-sized cubic sub-regions (children) for any node that contains data exceeding a threshold or requires higher resolution [S2][S3, unofficial]. The subdivision cuts space using split points at the midpoints of the current node's dimensions along the X, Y, and Z axes [S1, unofficial].
3.  **Data Insertion:** For each voxel in the input data, traverse the tree from the root [G]. If the current node is a leaf and the voxel falls within its bounds, store the data [G]. If the node is already a leaf but needs further subdivision (due to new data or resolution requirements), split it into 8 children and move existing data to the appropriate children [S2][S3, unofficial].
4.  **Memory Allocation:** Allocate memory only for nodes that contain data or are ancestors of data nodes. Empty branches are omitted [S2][S3, unofficial].
5.  **Coordinate Reference:** To reduce storage size, coordinates within a sub-cube can be stored relative to the sub-cube's origin, allowing for smaller average bit-lengths for position data [S2].
6.  **Traversal (Rendering):** To render, traverse the tree from the root [G]. Use mipmapping to limit traversal depth if the view distance is large, avoiding the cost of traversing deep levels (e.g., 24 levels for Earth-scale maps) [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self, bounds, depth=0):
        self.bounds = bounds  # Tuple (min_x, min_y, min_z, max_x, max_y, max_z)
        self.depth = depth
        self.children = [None] * 8  # 8 pointers to child nodes
        self.data = None  # Stored in leaf nodes

    def insert(self, voxel):
        # voxel is a dict/object with 'pos' (x,y,z) and 'value'
        if self.is_leaf():
            if self.data is not None:
                self.split()
            child_idx = self.get_child_index(voxel.pos)
            if self.children[child_idx] is None:
                self.create_child(child_idx)
            self.children[child_idx].insert(voxel)
        else:
            child_idx = self.get_child_index(voxel.pos)
            if self.children[child_idx] is None:
                self.create_child(child_idx)
            self.children[child_idx].insert(voxel)

    def is_leaf(self):
        return all(c is None for c in self.children)

    def split(self):
        min_x, min_y, min_z, max_x, max_y, max_z = self.bounds
        mid_x = (min_x + max_x) / 2
        mid_y = (min_y + max_y) / 2
        mid_z = (min_z + max_z) / 2
        for i in range(8):
            x_min = mid_x if (i & 1) else min_x
            x_max = max_x if (i & 1) else mid_x
            y_min = mid_y if (i & 2) else min_y
            y_max = max_y if (i & 2) else mid_y
            z_min = mid_z if (i & 4) else min_z
            z_max = max_z if (i & 4) else mid_z
            self.children[i] = OctreeNode((x_min, y_min, z_min, x_max, y_max, z_max), self.depth + 1)
        if self.data is not None:
            child_idx = self.get_child_index(self.data.pos)
            if self.children[child_idx] is not None:
                self.children[child_idx].data = self.data
                self.data = None

    def create_child(self, index):
        min_x, min_y, min_z, max_x, max_y, max_z = self.bounds
        mid_x = (min_x + max_x) / 2
        mid_y = (min_y + max_y) / 2
        mid_z = (min_z + max_z) / 2
        x_min = mid_x if (index & 1) else min_x
        x_max = max_x if (index & 1) else mid_x
        y_min = mid_y if (index & 2) else min_y
        y_max = max_y if (index & 2) else mid_y
        z_min = mid_z if (index & 4) else min_z
        z_max = max_z if (index & 4) else mid_z
        self.children[index] = OctreeNode((x_min, y_min, z_min, x_max, y_max, z_max), self.depth + 1)

    def get_child_index(self, pos):
        x, y, z = pos
        min_x, min_y, min_z, max_x, max_y, max_z = self.bounds
        mid_x = (min_x + max_x) / 2
        mid_y = (min_y + max_y) / 2
        mid_z = (min_z + max_z) / 2
        index = 0
        if x >= mid_x: index |= 1
        if y >= mid_y: index |= 2
        if z >= mid_z: index |= 4
        return index

    def traverse(self, ray_origin, ray_direction):
        if self.is_leaf():
            return self.data.value if self.data is not None else 0
        child_idx = self.get_ray_child_index(ray_origin, ray_direction)
        if child_idx != -1:
            return self.children[child_idx].traverse(ray_origin, ray_direction)
        return 0

    def get_ray_child_index(self, origin, direction):
        # Standard ray-box intersection to find the first child entered [G]
        min_x, min_y, min_z, max_x, max_y, max_z = self.bounds
        tx1 = (min_x - origin[0]) / direction[0] if direction[0] != 0 else -float('inf')
        tx2 = (max_x - origin[0]) / direction[0] if direction[0] != 0 else float('inf')
        ty1 = (min_y - origin[1]) / direction[1] if direction[1] != 0 else -float('inf')
        ty2 = (max_y - origin[1]) / direction[1] if direction[1] != 0 else float('inf')
        tz1 = (min_z - origin[2]) / direction[2] if direction[2] != 0 else -float('inf')
        tz2 = (max_z - origin[2]) / direction[2] if direction[2] != 0 else float('inf')
        t_enter = max(min(tx1, tx2), min(ty1, ty2), min(tz1, tz2))
        t_exit = min(max(tx1, tx2), max(ty1, ty2), max(tz1, tz2))
        if t_exit < 0 or t_enter > t_exit:
            return -1
        entry_x = origin[0] + direction[0] * t_enter
        entry_y = origin[1] + direction[1] * t_enter
        entry_z = origin[2] + direction[2] * t_enter
        return self.get_child_index((entry_x, entry_y, entry_z))
```

**Parameters:**
*   **Max Depth:** Limits the recursion depth to prevent excessive memory overhead [G]. For Earth-sized maps, 24 levels may be too costly for GPU traversal, so mipmapping is used to limit effective depth [S5, unofficial].
*   **Threshold:** The criteria for splitting a node (e.g., number of voxels, variance in data). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Overhead at Fine Resolutions:** If many leaf cells are located at the finest resolutions, the octree incurs huge overhead due to node pointers and structure [S6, unofficial].
*   **GPU Memory Limits:** GPUs generally lack the memory capacity to handle extremely large octrees (e.g., Earth-sized maps with trillions of voxels); implementation should be done on the CPU to avoid system bus bottlenecks [S5, unofficial].
*   **Traversal Cost:** Deep trees (e.g., 24 levels) result in gargantuan branching costs for both GPUs and CPUs [S5, unofficial]. Mipmapping is necessary to mitigate this [S5, unofficial].
## Subtopics

### [Spherical Binoctree](18953-spherical-binoctree.md)
A variant designed for spherical volumes, using binary subdivision in the radial direction and octree-like subdivision in angular directions to avoid sphoxel elongation. Choose this for omnidirectional rendering where radial distortion is a concern.

### [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md)
Discusses variations including pointer trees and DAGs, which optimize storage by sharing identical subtrees. Choose DAGs when there is significant redundancy in the spatial structure.

## Operating characteristics
*   **Time Complexity:** Traversal is $O(\log N)$ in the best case for sparse data, but can degrade to $O(N)$ in the worst case if the tree is deep and unbalanced [S5, S6, unofficial]. Ray casting requires checking up to 8 children per node [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of occupied nodes, not the total volume [S3, unofficial]. It can reduce memory usage by orders of magnitude compared to dense arrays for sparse data [S3, unofficial]. However, overhead from pointers and structure can be significant if many leaf cells are located at the finest resolutions [S6, unofficial].
*   **Measured Performance:** For Earth-sized maps (510 trillion voxels at 1m resolution), an octree with mipmapping is recommended to manage storage and rendering cost [S5, unofficial].
*   **Strengths:** Efficient for sparse scenes, regular structure for predictable memory, easy navigation to neighbors/parents/children, supports mipmapping [S1, S5, S6, unofficial].
*   **Limitations:** High overhead for dense data or fine-resolution leaves, difficult to implement stackless traversal on GPUs compared to KD-trees, limited GPU memory for very large scales [S1, S5, S6, unofficial].
*   **When to Use:** Sparse volumetric data, large-scale scenes requiring multi-resolution rendering, point cloud compression, NeRF acceleration [S2][S5, S6, unofficial].
*   **When Not to Use:** Dense volumetric data where 3D arrays might be more efficient for traversal costs, or when hardware lacks sufficient memory for the tree structure [S5, S6, unofficial].
## Background
The octree was introduced as a spatial partitioning technique for computer graphics [S2]. It has been widely applied to point cloud compression and speeding up Neural Radiance Field (NeRF) rendering [S2]. Laine's paper is referenced for sparse octree implementation [S5, unofficial]. The debate between KD-trees and octrees continues, with octrees being preferred for regular-spaced grid data and KD-trees for irregular geometry [S1, unofficial].
## Sources
[S1] Rendering Large Volumetric, https://forceflow.be/wp-content/uploads/2012/02/vr_overview.pdf, fetched 2026-10-24 (unofficial)
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S3] volume rendering, https://grokipedia.com/page/Volume_rendering, fetched 2026-10-26 (unofficial)
[S4] Voxel Compression: A Detailed Overview - flyriver.com, https://www.flyriver.com/g/voxel-compression, fetched 2026-10-27 (unofficial)
[S5] What kind of hardware would be required to render an Earth sized minecraft like map?, https://gamedev.stackexchange.com/questions/21951, fetched 2026-10-28 (unofficial)
[S6] How do you store voxel data?, https://stackoverflow.com/questions/27810937/how-do-you-store-voxel-data, fetched 2026-10-28 (unofficial)
