# Sparse Voxel DAG Construction with Subtree Caching

Also called: DAG-based Voxel Grid Compression, Sparse Voxel Octree (SVO) and DAG Compression, Subtree Merging Voxel Engine

## Overview
Sparse Voxel DAG Construction with Subtree Caching is a method for compressing a 3D voxel grid into a Directed Acyclic Graph (DAG) by detecting and merging identical subtrees, thereby reducing storage overhead while maintaining real-time decoding capability [S1, S2, unofficial]. It builds on [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md) by replacing the explicit octree structure with a shared graph structure where identical spatial configurations are represented by a single node instance [S1, S2, unofficial]. This approach significantly reduces memory usage compared to standard Sparse Voxel Octrees (SVOs) by eliminating redundant node allocations for repeated geometric patterns [S1, S2, unofficial].

## How it works
The method converts a dense or sparse voxel grid into a DAG through recursive subdivision and subtree hashing [S2, unofficial]. The core mechanism is the use of a cache to store and reuse nodes representing identical subtrees, ensuring that any two spatially identical regions in the voxel grid share the same node object in memory [S2, unofficial].

### Inputs and Outputs
- **Input**: A 3D voxel grid `byte[,,] voxelGrid` of size $(1 \ll \text{gridDepth}) \times (1 \ll \text{gridDepth}) \times (1 \ll \text{gridDepth})$ [S2, unofficial].
- **Output**: A root `VoxelNode` representing the DAG, and a `subtreeCache` dictionary mapping subtree keys to node instances [S2, unofficial].

### Data Structures
**VoxelNode**:
- `IsLeaf` (bool): True if the node represents a leaf voxel; False if it has children [S2, unofficial].
- `Value` (byte): 0 if empty, 1 if filled. Only valid if `IsLeaf` is true [S2, unofficial].
- `Children` (VoxelNode[8]): Array of 8 child nodes. Only valid if `IsLeaf` is false [S2, unofficial].
- `Position` (Vector3): World-space position of the voxel/cube center [S2, unofficial].
- `Size` (float): Side length of the voxel/cube [S2, unofficial].
- `NodeColor` (Color): Optional material property [S2, unofficial].

**subtreeCache**:
- `Dictionary<string, VoxelNode>`: Maps a unique string key (hash of subtree content) to a `VoxelNode` instance. Used to detect and merge identical subtrees [S2, unofficial].

### Algorithm Steps
1. **Initialization**: Create a voxel grid of size $(1 \ll \text{gridDepth})^3$ and fill with geometry data [S2, unofficial].
2. **Recursive Construction**: Call `BuildDAG(voxelGrid, gridDepth)` to construct the DAG from the root [S2, unofficial].
3. **Subtree Key Generation**: For each recursive call, generate a unique key for the current grid subtree using `GenerateSubtreeKey(grid)` [S2, unofficial].
4. **Cache Lookup**: Check if the key exists in `subtreeCache`. If `TryGetValue` returns true, return the existing `VoxelNode` to reuse it [S2, unofficial].
5. **Leaf Detection**: If `depth == 0` or the grid is homogeneous (all voxels have the same value), create a new leaf node with `IsLeaf=true` and `Value=grid[0,0,0]`. Store it in `subtreeCache` with the key and return it [S2, unofficial].
6. **Non-Leaf Construction**: If not a leaf, create a new non-leaf node with `IsLeaf=false`. Calculate `newDepth = depth - 1` and `size = 1 << newDepth`. Iterate $i$ from 0 to 7: extract octant using `GetOctant(grid, i, size)`, recursively call `BuildDAG(octant, newDepth)`, and assign result to `node.Children[i]`. Store the new node in `subtreeCache` with the key and return it [S2, unofficial].
7. **LOD Generation**: For Level of Detail (LOD) generation, downsample the grid by marking a voxel as containing geometry only if any of its composing voxels contain geometry [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import hashlib

class VoxelNode:
    def __init__(self, is_leaf=False, value=0, children=None, position=None, size=1.0, color=None):
        self.IsLeaf = is_leaf
        self.Value = value
        self.Children = children if children else [None] * 8
        self.Position = position
        self.Size = size
        self.NodeColor = color

class SparseVoxelDAG:
    def __init__(self, grid_depth):
        self.grid_depth = grid_depth
        self.subtree_cache = {}
        self.root_node = None
        self.voxel_grid = None
        self.vertices = []
        self.triangles = []

    def build_dag(self, grid, depth):
        key = self.generate_subtree_key(grid)
        if key in self.subtree_cache:
            return self.subtree_cache[key]
        if depth == 0 or self.is_homogeneous(grid):
            node = VoxelNode(is_leaf=True, value=grid[0, 0, 0])
            self.subtree_cache[key] = node
            return node
        node = VoxelNode(is_leaf=False)
        new_depth = depth - 1
        size = 1 << new_depth
        for i in range(8):
            octant = self.get_octant(grid, i, size)
            node.Children[i] = self.build_dag(octant, new_depth)
        self.subtree_cache[key] = node
        return node

    def generate_subtree_key(self, grid):
        if self.is_homogeneous(grid):
            return f"Homogeneous_{grid[0, 0, 0]}"
        key_builder = []
        grid_size = grid.shape[0]
        for z in range(grid_size):
            for y in range(grid_size):
                for x in range(grid_size):
                    key_builder.append(str(grid[x, y, z]))
        key_string = "".join(key_builder)
        hash_algorithm = hashlib.sha256()
        hash_algorithm.update(key_string.encode('utf-8'))
        return hash_algorithm.hexdigest()

    def is_homogeneous(self, grid):
        first_value = grid[0, 0, 0]
        grid_size = grid.shape[0]
        for z in range(grid_size):
            for y in range(grid_size):
                for x in range(grid_size):
                    if grid[x, y, z] != first_value:
                        return False
        return True

    def get_octant(self, grid, octant_index, size):
        x_start = (octant_index & 1) * size
        y_start = (octant_index & 2) * size
        z_start = (octant_index & 4) * size
        return grid[x_start:x_start+size, y_start:y_start+size, z_start:z_start+size]

    def traverse_svdag(self, node, position, size):
        if node is None:
            return
        if node.IsLeaf:
            if node.Value == 1:
                self.add_voxel_mesh_data(position, size, node.NodeColor)
        else:
            child_size = size * 0.5
            for i in range(8):
                child_position = self.calculate_child_position(position, child_size, i)
                self.traverse_svdag(node.Children[i], child_position, child_size)

    def calculate_child_position(self, parent_position, child_size, octant_index):
        offset_x = (octant_index & 1) * child_size
        offset_y = (octant_index & 2) * (child_size / 2)
        offset_z = (octant_index & 4) * (child_size / 4)
        return (parent_position[0] + offset_x, parent_position[1] + offset_y, parent_position[2] + offset_z)

    def add_voxel_mesh_data(self, position, size, color):
        local_vertices = [
            (-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, 0.5, 0.5), (-0.5, 0.5, 0.5),
            (-0.5, -0.5, -0.5), (0.5, -0.5, -0.5), (0.5, 0.5, -0.5), (-0.5, 0.5, -0.5)
        ]
        vertex_start_index = len(self.vertices)
        for vx, vy, vz in local_vertices:
            self.vertices.append((position[0] + vx * size, position[1] + vy * size, position[2] + vz * size))
        indices = [
            0, 2, 1, 0, 3, 2, 2, 3, 6, 3, 7, 6, 1, 2, 6, 1, 6, 5,
            0, 7, 3, 0, 4, 7, 0, 1, 5, 0, 5, 4, 4, 5, 6, 4, 6, 7
        ]
        for idx in indices:
            self.triangles.append(vertex_start_index + idx)

    def initialize(self):
        grid_size = 1 << self.grid_depth
        self.voxel_grid = [[[0] * grid_size for _ in range(grid_size)] for _ in range(grid_size)]
        for x in range(grid_size):
            for y in range(grid_size):
                for z in range(grid_size):
                    self.voxel_grid[x][y][z] = (x + y + z) % 2
        self.root_node = self.build_dag(self.voxel_grid, self.grid_depth)
```

### Parameters
- **gridDepth**: The logarithm of the grid dimension. For a $256^3$ grid, `gridDepth` is 8 [S2, unofficial].
- **subtreeCache**: A dictionary used to cache and lookup existing nodes by their subtree key to enable reuse [S2, unofficial].

### Edge Cases and Pitfalls
- **Hash Collisions**: The method uses SHA256 hashing for subtree keys [S2, unofficial].
- **Homogeneity Check**: The `is_homogeneous` check is critical for leaf detection [S2, unofficial].
- **Memory Overhead**: The `subtreeCache` dictionary stores all unique subtrees, which can consume memory proportional to the number of unique nodes [S2, unofficial].
- **LOD Generation**: The source mentions downsampling for LODs by marking a voxel as containing geometry if any composing voxels do [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Strengths**: Significant memory savings for scenes with repetitive geometry. Real-time decoding is possible because the DAG structure is similar to an SVO [S1, S2, unofficial].
- **Limitations**: Construction requires hashing and dictionary lookups for every subtree. The cache grows with the number of unique subtrees [S1, S2, unofficial].
- **When to Use**: Ideal for voxel-based applications with repetitive structures. Not ideal for highly dynamic, non-repetitive data where construction overhead outweighs memory benefits [S1, S2, unofficial].

## Background
The concept of Sparse Voxel Octrees (SVOs) was introduced to address the memory inefficiency of dense voxel grids [S1, unofficial]. Kampe et al. realized that merging identical leaves could significantly compact the final level of an SVO [S1, unofficial]. The extension to Directed Acyclic Graphs (DAGs) by merging identical subtrees (not just leaves) further reduces memory usage [S1, S2, unofficial]. This method is used in voxel engines and real-time rendering applications where memory efficiency and fast traversal are critical [S1, S2, unofficial].

## Sources
[S1] On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
[S2] Building a High-Performance Voxel Engine in Unity: A Step ..., https://medium.com/@adamy1558/building-a-high-performance-voxel-engine-in-unity-a-step-by-step-guide-part-6-high-resolution-e0217efd56d2, fetched 2026-10-27 (unofficial)
