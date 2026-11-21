# Sparse Voxel Octree Node Structure

Also called: SparseVoxelOctreeNode struct

## Overview
The Sparse Voxel Octree (SVO) node structure defines the memory layout, connectivity, and state management for individual nodes within a sparse voxel octree. It solves the problem of efficiently representing volumetric data by storing only occupied regions, using spatial bounds, parent/child links, and directional neighbor pointers to enable fast traversal and ray casting. Unlike static octrees, this implementation dynamically splits leaves when a minimum dimension threshold is reached and merges identical children via simplification, reducing memory overhead for uniform regions [S1, unofficial]. It builds on the general [Octree Data Structure Layout](4518-octree-data-structure-layout.md) by specifying concrete field types and operational semantics for node mutation.

## How it works
### Inputs and Outputs
- **Input**: A target 3D position (integer coordinates), a data value of type $T$, and a minimum leaf dimension parameter $d_{min}$.
- **Output**: A modified octree structure where the node at the specified position contains the data, or `None` if the position is out of bounds.

### Data Structures

#### Node Layout (Implementation A)
Based on [S1, unofficial], the `Node<T>` struct contains the following fields:
1.  `ty`: `NodeType<T>` enum.
2.  `bounds`: `Bounds`, an array of two `Vector3<u32>` representing the inclusive lower bound and exclusive upper bound of the node's spatial extent.
3.  `children`: Array of 8 `Box<Option<Node<T>>>`, representing the 8 octant children.

**`NodeType<T>` Enum**:
-   `Leaf(T)`: Stores the data value $T$.
-   `Internal`: Marks a split node with children.
-   `Simplified`: Marks a compressed node (implementation detail for merging).

**`Octant` Enum**:
Maps 8 spatial subdivisions to integers 0–7:
-   0: LeftRearBase
-   1: RightRearBase
-   2: LeftRearTop
-   3: RightRearTop
-   4: LeftFrontBase
-   5: RightFrontBase
-   6: LeftFrontTop
-   7: RightFrontTop

#### Node Layout (Implementation B)
Based on [S2, unofficial], the `SparseVoxelOctreeNode` struct contains:
1.  `position`: `UVec3` (3D unsigned integer vector).
2.  `size`: `u32`.
3.  `parent`: `Option<SparseVoxelOctreeLink>`.
4.  `first_child`: `Option<SparseVoxelOctreeLink>`.
5.  `is_leaf`: `bool`.
6.  `neighbors`: Array of 6 `Option<SparseVoxelOctreeLink>` (directional neighbors).

*Note: The two implementations differ significantly in layout. Implementation A uses embedded bounds and an array of child pointers, while Implementation B uses explicit position/size and linked pointers with neighbor caching. The algorithmic steps below follow Implementation A [S1, unofficial] as it provides complete operational logic.*

### Parameters
-   `min_dimension` ($d_{min}$): `u32`. The minimum spatial extent (in voxels/units) for a leaf node. Controls tree depth and granularity. Typical value: 1 [S1, unofficial].

### Numbered Steps

#### 1. Insertion
1.  Check if the target `position` is within the node's `bounds`. If not, return error or `None` [S1, unofficial].
2.  Calculate the current node `dimension` as $|bounds[0].x - bounds[1].x|$ [S1, unofficial].
3.  If `dimension` equals `min_dimension`:
    -   Set `ty` to `Leaf(data)` [S1, unofficial].
    -   Return success.
4.  Otherwise, compute `child_info`:
    -   Calculate half-dimension: $d_{half} = dimension / 2$.
    -   Calculate midpoint: $P_{mid} = bounds[0] + (d_{half}, d_{half}, d_{half})$.
    -   Determine `octant` by comparing `position` to $P_{mid}$ using `vector_diff` [S1, unofficial].
5.  Retrieve or create the child node at the calculated `octant` index.
6.  Recursively insert the data into the child node [S1, unofficial].

#### 2. Splitting (Implicit in Insertion)
When a leaf node reaches `min_dimension` during insertion, it splits into 8 children [S1, unofficial]:
1.  The existing leaf data is copied to all 8 children.
2.  The child corresponding to the target octant receives the new data.
3.  The parent node's `ty` is set to `Internal` [S1, unofficial].

#### 3. Simplification
Iterates over all 8 children to merge identical leaves:
1.  Initialize `data` as `None`.
2.  For each child $i$ from 0 to 7:
    -   If child exists and is a leaf:
        -   If `data` is `None`, set `data` to the child's leaf data.
        -   If `data` is not `None` and differs from the child's data, return `false` [S1, unofficial].
    -   If child is missing and `ty` is `Internal`, return `false` [S1, unofficial].
3.  If `data` is not `None`:
    -   Set `ty` to `Leaf(data)`.
    -   Clear the `children` array (set all to `None`) [S1, unofficial].
4.  Return `true` if successful [S1, unofficial].

#### 4. Level of Detail (LOD)
Downsamples the node for coarser representations:
1.  Iterate over all 8 children.
2.  For each child:
    -   If it is a leaf, collect its data into an array.
    -   If it is not a leaf, recursively call `lod()` on it.
    -   If any child is missing, return early without modifying the node [S1, unofficial].
3.  Count frequencies of each data value in the collected array using a hash map.
4.  Sort the counts in descending order.
5.  Select the most frequent value.
6.  Set `ty` to `Leaf(most_frequent_value)`.
7.  Clear the `children` array [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class Bounds:
    def __init__(self, min_pos, max_pos):
        self.min = min_pos  # Inclusive
        self.max = max_pos  # Exclusive

class NodeType:
    LEAF = 0
    INTERNAL = 1
    SIMPLIFIED = 2

class Node:
    def __init__(self, bounds, data=None):
        self.ty = NodeType.LEAF if data is not None else NodeType.INTERNAL
        self.bounds = bounds
        self.children = [None] * 8  # Array of 8 Box<Option<Node<T>>>
        self.data = data

    def dimension(self):
        # Returns the spatial extent in one dimension (assumes cubic nodes)
        return abs(self.bounds.max.x - self.bounds.min.x)

    def min_position(self):
        return self.bounds.min

    def contains(self, position):
        return (position.x >= self.bounds.min.x and position.x < self.bounds.max.x and
                position.y >= self.bounds.min.y and position.y < self.bounds.max.y and
                position.z >= self.bounds.min.z and position.z < self.bounds.max.z)

    def leaf_data(self):
        if self.ty == NodeType.LEAF:
            return self.data
        return None

    def is_leaf(self):
        return self.ty == NodeType.LEAF

    def child_info(self, position):
        if self.contains(position):
            dim = self.dimension() // 2
            dim_3d = Vector3(dim, dim, dim)
            midpoint = Vector3(
                self.min_position().x + dim,
                self.min_position().y + dim,
                self.min_position().z + dim
            )
            # Determine octant based on position relative to midpoint
            # Assuming standard octant mapping:
            # bit 0: x >= midpoint.x
            # bit 1: y >= midpoint.y
            # bit 2: z >= midpoint.z
            octant = 0
            if position.x >= midpoint.x: octant |= 1
            if position.y >= midpoint.y: octant |= 2
            if position.z >= midpoint.z: octant |= 4
            return {
                'dimension': dim,
                'dimension_3d': dim_3d,
                'octant': octant
            }
        return None

    def child_bounds(self, dim_3d, octant):
        lower = Vector3(
            self.min_position().x + (dim_3d.x if octant & 1 else 0),
            self.min_position().y + (dim_3d.y if octant & 2 else 0),
            self.min_position().z + (dim_3d.z if octant & 4 else 0)
        )
        upper = Vector3(
            lower.x + dim_3d.x,
            lower.y + dim_3d.y,
            lower.z + dim_3d.z
        )
        return Bounds(lower, upper)

    def insert(self, position, data, min_dimension):
        if not self.contains(position):
            return False
        
        dim = self.dimension()
        if dim == min_dimension:
            self.ty = NodeType.LEAF
            self.data = data
            return True
        
        info = self.child_info(position)
        if info is None:
            return False
        
        octant = info['octant']
        if self.children[octant] is None:
            child_bounds = self.child_bounds(info['dimension_3d'], octant)
            self.children[octant] = Node(child_bounds)
        
        # If this node was a leaf, it must now become internal and split
        if self.is_leaf():
            self.ty = NodeType.INTERNAL
            # Copy existing data to all children
            for i in range(8):
                if self.children[i] is None:
                    child_bounds = self.child_bounds(info['dimension_3d'], i)
                    self.children[i] = Node(child_bounds)
                self.children[i].data = self.data
                self.children[i].ty = NodeType.LEAF
        
        return self.children[octant].insert(position, data, min_dimension)

    def simplify(self):
        data = None
        for i in range(8):
            child = self.children[i]
            if child is not None:
                if child.is_leaf():
                    child_data = child.leaf_data()
                    if data is None:
                        data = child_data
                    elif data != child_data:
                        return False
            elif self.ty == NodeType.INTERNAL:
                return False
        
        if data is not None:
            self.ty = NodeType.LEAF
            self.data = data
            self.children = [None] * 8
            return True
        return False

    def lod(self):
        all_data = [None] * 8
        for i in range(8):
            c = self.children[i]
            if c is None:
                return  # Return early if any child is missing
            if c.is_leaf():
                all_data[i] = c.leaf_data()
            else:
                c.lod()  # Recursively downsample non-leaf children
        
        # Count frequencies
        counts = {}
        for d in all_data:
            if d is not None:
                counts[d] = counts.get(d, 0) + 1
        
        if counts:
            # Sort by count descending
            sorted_counts = sorted(counts.items(), key=lambda x: x[1], reverse=True)
            self.ty = NodeType.LEAF
            self.data = sorted_counts[0][0]
        
        self.children = [None] * 8
```

### Edge Cases and Pitfalls
-   **Missing Children in Simplify**: If a node is marked `Internal` but has missing children, `simplify()` immediately returns `false`, preventing invalid compression [S1, unofficial].
-   **Missing Children in LOD**: `lod()` returns early without modifying the node if any child is missing or is not a leaf, preserving tree integrity during downsampling [S1, unofficial].
-   **Precision**: Coordinates are stored as `u32`. Ensure that `min_dimension` and `max_dimension` are powers of 2 to avoid fractional midpoints [S1, unofficial].

## Operating characteristics
-   **Time Complexity**:
    -   Insertion: $O(\log N)$ where $N$ is the total number of voxels in the root domain, assuming balanced splits.
    -   Simplification: $O(1)$ per node (constant 8 children).
    -   LOD: $O(1)$ per node (constant 8 children).
-   **Memory Complexity**:
    -   Implementation A: Each internal node stores 8 pointers. Leaf nodes store data. Memory usage is proportional to the number of nodes, which is significantly less than a dense grid for sparse data.
    -   Implementation B: Stores explicit position, size, parent, child, and 6 neighbor links. Neighbor links add overhead but accelerate traversal.
-   **Strengths**:
    -   Efficient for sparse data.
    -   Dynamic splitting and merging allow for adaptive resolution.
    -   Neighbor links (in Implementation B) enable fast ray marching without recursive traversal.
-   **Limitations**:
    -   Pointer overhead for dense data.
    -   Complexity in maintaining neighbor links (not detailed in [S1, unofficial]).
    -   Not given in the sources: Specific performance metrics on hardware.

## Background
The Sparse Voxel Octree is a fundamental data structure in volume rendering, enabling efficient storage and traversal of large volumetric datasets. The implementation described here is based on the `svo-rs` library [S1, S2, unofficial]. It builds on the general [Octree Data Structure Layout](4518-octree-data-structure-layout.md) by providing concrete operational semantics for node mutation, including dynamic splitting and simplification.

## Sources
[S1] Adam-Gleave/svo-rs, https://github.com/Adam-Gleave/svo-rs, fetched 2026-10-26 (unofficial)
[S2] mkmarek/svo-rs, https://github.com/mkmarek/svo-rs, fetched 2026-10-26 (unofficial)
