# OctreeVoxelGrid


## Overview
OctreeVoxelGrid is a sparse, multi-resolution voxel data structure that implements the `VoxelGrid` protocol, allowing it to replace uniform grids in existing engine code without modification [S1]. It solves the memory inefficiency of dense grids for sparse scenes by storing only active regions, achieving $O(\text{active cells} \times \text{layers})$ memory complexity instead of $O(\text{grid}^3)$ [S1]. It distinguishes itself from sibling structures by supporting dynamic refinement/coarsening policies and inheriting parent values to children upon split to avoid interpolation artifacts at refinement boundaries [S1].

## How it works
**Inputs and Outputs:**
The structure is initialized with world dimensions (`dimensions: tuple[int, int, int]`), cell size (`cell_size: float`), and a maximum refinement depth (`max_depth: int`, default 4) [S1]. It provides methods to read (`get`), write (`set`), modify (`add`), query spatial overlap (`query_overlap`), iterate (`walk_layer`), convert coordinates (`world_to_grid`), and manage dirty state (`get_delta_packet`) [S1].

**Data Structures:**
The core unit is the `OctreeNode`, which contains:
- `is_leaf: bool`: True if the node stores data, False if it has children.
- `value: dict[str, float] | None`: A mapping from layer name to float value. Present only if `is_leaf` is True.
- `children: list[OctreeNode]`: A list of up to 8 child nodes. Present only if `is_leaf` is False.
- `bounds: tuple[float, ...]`: The world-space Axis-Aligned Bounding Box (AABB) of the node, stored as `(min_x, min_y, min_z, max_x, max_y, max_z)` [S1].

**Numbered Steps:**

1.  **Initialization**: Create the root `OctreeNode` covering the full `dimensions` at depth 0, with `is_leaf=True`, `value={}`, and `children=[]` [S1].
2.  **Coordinate Conversion**: Convert world coordinates $(wx, wy, wz)$ to coarse grid coordinates using $gx = \lfloor wx / \text{cell\_size} \rfloor$, $gy = \lfloor wy / \text{cell\_size} \rfloor$, $gz = \lfloor wz / \text{cell\_size} \rfloor$ via `world_to_grid` [S1].
3.  **Get Value (`get`)**: Descend to the finest resolution at $(x, y, z)$, returning `value[layer]` or `DEFAULT_VALUE` if the path is missing or depth is exceeded [S1].
4.  **Set Value (`set`)**: Descend to the leaf node corresponding to $(x, y, z)$, creating missing nodes or splitting leaves if `depth < max_depth`, update `value[layer]`, and mark the node dirty [S1].
5.  **Add Value (`add`)**: Perform `get(layer, x, y, z)` to retrieve the current value, then perform `set(layer, x, y, z, current_value + delta)` [S1].
6.  **Query Overlap (`query_overlap`)**: Recursively descend; if the node AABB is fully outside the query sphere, skip the subtree; if fully inside, collect all leaves in the subtree; otherwise, recurse into children [S1].
7.  **Walk Layer (`walk_layer`)**: Perform a depth-first search over leaf nodes only, calling `callback(x, y, z, value[layer])` for each [S1].
8.  **Delta Packet (`get_delta_packet`)**: Traverse the tree to collect all nodes marked dirty, return the changes, and clear the dirty flags and internal buffer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
DEFAULT_VALUE = 0.0

class OctreeNode:
    def __init__(self, bounds, is_leaf=True, value=None):
        self.bounds = bounds  # (min_x, min_y, min_z, max_x, max_y, max_z)
        self.is_leaf = is_leaf
        self.value = value if value is not None else {}
        self.children = []
        self.dirty = False

    def split(self):
        """Split a leaf node into 8 children, inheriting parent value."""
        if not self.is_leaf:
            return
        center_w = tuple((self.bounds[i] + self.bounds[i+3]) / 2.0 for i in range(3))
        half_w = tuple((self.bounds[i+3] - self.bounds[i]) / 2.0 for i in range(3))
        self.is_leaf = False
        self.children = []
        for i in range(8):
            min_b = []
            max_b = []
            for d in range(3):
                bit = (i >> d) & 1
                if bit == 0:
                    min_b.append(center_w[d] - half_w[d])
                    max_b.append(center_w[d])
                else:
                    min_b.append(center_w[d])
                    max_b.append(center_w[d] + half_w[d])
            child_bounds = tuple(min_b) + tuple(max_b)
            child = OctreeNode(child_bounds, is_leaf=True, value=dict(self.value))
            if self.dirty:
                child.dirty = True
            self.children.append(child)

class OctreeVoxelGrid:
    def __init__(self, dimensions, cell_size, max_depth=4):
        self.dimensions = dimensions
        self.cell_size = cell_size
        self.max_depth = max_depth
        self.root = OctreeNode((0, 0, 0, dimensions[0]*cell_size, dimensions[1]*cell_size, dimensions[2]*cell_size))

    def world_to_grid(self, wx, wy, wz):
        return (int(wx / self.cell_size), int(wy / self.cell_size), int(wz / self.cell_size))

    def _get_child_index(self, node, gx, gy, gz):
        min_g = tuple(b / self.cell_size for b in node.bounds[:3])
        max_g = tuple(b / self.cell_size for b in node.bounds[3:])
        center_g = tuple((min_g[d] + max_g[d]) / 2.0 for d in range(3))
        idx = 0
        if gx >= center_g[0]: idx |= 1
        if gy >= center_g[1]: idx |= 2
        if gz >= center_g[2]: idx |= 4
        return idx

    def _get_node_at(self, node, gx, gy, gz, depth):
        if node.is_leaf:
            return node
        if depth >= self.max_depth:
            return node
        idx = self._get_child_index(node, gx, gy, gz)
        if idx < len(node.children):
            return self._get_node_at(node.children[idx], gx, gy, gz, depth + 1)
        return None

    def get(self, layer, x, y, z):
        node = self._get_node_at(self.root, x, y, z, 0)
        if node is None or not node.is_leaf:
            return DEFAULT_VALUE
        return node.value.get(layer, DEFAULT_VALUE)

    def set(self, layer, x, y, z, value):
        self._set_recursive(self.root, x, y, z, 0, layer, value)

    def _set_recursive(self, node, x, y, z, depth, layer, value):
        if node.is_leaf:
            if depth < self.max_depth:
                node.split()
            else:
                node.value[layer] = value
                node.dirty = True
            return
        idx = self._get_child_index(node, x, y, z)
        if idx >= len(node.children):
            center_w = tuple((node.bounds[i] + node.bounds[i+3]) / 2.0 for i in range(3))
            half_w = tuple((node.bounds[i+3] - node.bounds[i]) / 2.0 for i in range(3))
            min_b = []
            max_b = []
            for d in range(3):
                bit = (idx >> d) & 1
                if bit == 0:
                    min_b.append(center_w[d] - half_w[d])
                    max_b.append(center_w[d])
                else:
                    min_b.append(center_w[d])
                    max_b.append(center_w[d] + half_w[d])
            child_bounds = tuple(min_b) + tuple(max_b)
            node.children.append(OctreeNode(child_bounds))
        self._set_recursive(node.children[idx], x, y, z, depth + 1, layer, value)

    def add(self, layer, x, y, z, delta):
        current = self.get(layer, x, y, z)
        self.set(layer, x, y, z, current + delta)

    def _aabb_fully_outside_sphere(self, bounds, center, radius):
        min_b = bounds[:3]
        max_b = bounds[3:]
        closest = []
        for i in range(3):
            if center[i] < min_b[i]: closest.append(min_b[i])
            elif center[i] > max_b[i]: closest.append(max_b[i])
            else: closest.append(center[i])
        dist_sq = sum((closest[i] - center[i])**2 for i in range(3))
        return dist_sq > radius * radius

    def _aabb_fully_inside_sphere(self, bounds, center, radius):
        min_b = bounds[:3]
        max_b = bounds[3:]
        farthest = []
        for i in range(3):
            if center[i] < (min_b[i] + max_b[i]) / 2:
                farthest.append(max_b[i])
            else:
                farthest.append(min_b[i])
        dist_sq = sum((farthest[i] - center[i])**2 for i in range(3))
        return dist_sq <= radius * radius

    def query_overlap(self, center, radius):
        results = []
        self._query_recursive(self.root, center, radius, results)
        return results

    def _query_recursive(self, node, center, radius, results):
        if self._aabb_fully_outside_sphere(node.bounds, center, radius):
            return
        if self._aabb_fully_inside_sphere(node.bounds, center, radius):
            self._collect_leaves(node, results)
            return
        for child in node.children:
            self._query_recursive(child, center, radius, results)

    def _collect_leaves(self, node, results):
        if node.is_leaf:
            min_g = tuple(b / self.cell_size for b in node.bounds[:3])
            results.append((int(min_g[0]), int(min_g[1]), int(min_g[2])))
            return
        for child in node.children:
            self._collect_leaves(child, results)

    def walk_layer(self, layer, callback):
        self._walk_recursive(self.root, layer, callback)

    def _walk_recursive(self, node, layer, callback):
        if node.is_leaf:
            min_g = tuple(b / self.cell_size for b in node.bounds[:3])
            callback(int(min_g[0]), int(min_g[1]), int(min_g[2]), node.value.get(layer, DEFAULT_VALUE))
            return
        for child in node.children:
            self._walk_recursive(child, layer, callback)

    def get_delta_packet(self):
        packet = {}
        self._collect_dirty(self.root, packet)
        self._clear_dirty(self.root)
        return packet

    def _collect_dirty(self, node, packet):
        if node.dirty and node.is_leaf:
            min_g = tuple(b / self.cell_size for b in node.bounds[:3])
            packet[(int(min_g[0]), int(min_g[1]), int(min_g[2]))] = dict(node.value)
        if not node.is_leaf:
            for child in node.children:
                self._collect_dirty(child, packet)

    def _clear_dirty(self, node):
        node.dirty = False
        if not node.is_leaf:
            for child in node.children:
                self._clear_dirty(child)
```

**Parameters:**
- `max_depth`: Integer, default 4. Limits the maximum refinement level [S1].
- `cell_size`: Float. Size of the coarsest voxel [S1].

**Edge Cases and Pitfalls:**
- **Interpolation Artifacts**: Avoided because parent values are inherited by children upon split; no interpolation is required for writes, only for reads if bilinear/trilinear filtering is applied on top of the grid values [S1].
- **Memory Overhead**: For small, dense worlds, the octree overhead may exceed a uniform grid; the uniform grid remains the default and the octree is opt-in via configuration [S1].
- **Thrashing**: Aggressive refinement policies can cause performance regression; configurable tick thresholds are required for coarsening [S1].

## Operating characteristics
- **Memory Complexity**: $O(\text{active cells} \times \text{layers})$, avoiding the $O(\text{grid}^3)$ footprint of uniform grids [S1].
- **Time Complexity**: `get` and `set` operations scale with tree depth, bounded by `max_depth` [S1]. `query_overlap` prunes subtrees via AABB-sphere tests, yielding sub-linear performance in sparse regions [G]. `walk_layer` visits only existing leaf nodes, scaling linearly with leaf count [S1].
- **Performance**: Benchmarks show improvement over uniform grids for sparse worlds, but dense worlds may suffer from traversal overhead [S1].
- **Strengths**: Zero integration cost via `VoxelGrid` protocol compliance, adaptive resolution, and memory bounded by `max_depth` regardless of world size [S1].
- **Limitations**: Not optimal for very dense, uniform volumes due to pointer and traversal overhead; requires opt-in configuration to avoid performance regression [S1].

## Background
OctreeVoxelGrid was developed as Phase 3 and 4 of a multi-resolution voxel grid implementation, specifically targeting an octree-based grid with configurable refinement and coarsening policies [S1]. It was designed to satisfy the `VoxelGrid` protocol, enabling seamless swap-in at engine initialization with zero changes to existing handlers or engine code [S1].

## Sources
[S1] multi-resolution voxel grid: octree implementation + refinement policy (#64 phases 3-4), https://github.com/hellolifeforms/lila/issues/68, fetched 2026-10-26
