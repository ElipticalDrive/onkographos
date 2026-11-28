# ReadAccessor (Single-Level Node Caching)
Also called: Single-Level Node Caching

## Overview
ReadAccessor is a class template in OpenVDB/NanoVDB that provides optimized access to grid values and leaf nodes by caching nodes at a single tree level, eliminating the need for full tree traversal for every query [S1]. It distinguishes itself from multi-level caching strategies by maintaining a cache depth of exactly one level (`CacheLevels = 1`), which reduces memory overhead while still accelerating access to frequently queried coordinates [S1].

## How it works
**Inputs and Outputs:**
The accessor is constructed from a `const RootT &root`, `const GridT &grid`, or `const TreeT &tree` [S1]. It outputs cached node pointers and values for coordinate queries `ijk` (of type `CoordType`) [S1].

**Data Structures:**
The internal cache layout is not explicitly detailed in the source, but it is managed internally by the `ReadAccessor` instance [S1]. The class relies on friend access from `RootNode`, `InternalNode`, and `LeafNode` to insert nodes into this cache [S1].

**Parameters:**
*   `CacheLevels`: A static constant integer set to `1`, indicating the number of levels cached [S1].
*   `Policy`: A template parameter for `getDimAndActive`, defaulting to `ActiveExact`. An alternative is `ActiveOnLeafOnly` [S1].

**Steps:**
1.  **Initialization**: Construct the `ReadAccessor` with the root, grid, or tree. The cache is initially empty [S1].
2.  **Cache Reset**: Call `clear()` to reset the access state to its initial state with an empty cache [S1].
3.  **Value Retrieval**: Retrieve a value at coordinate `ijk` using `getValue(ijk)`, `getValue(i, j, k)`, `operator()(ijk)`, or `operator()(i, j, k)` [S1].
4.  **Active State Check**: Determine if a coordinate is active using `isActive(ijk)` [S1].
5.  **Probe Value**: Attempt to retrieve a value into a reference variable `v` without throwing or returning a default; returns `true` if successful, `false` otherwise [S1].
6.  **Probe Leaf**: Attempt to retrieve a pointer to the `LeafT` node containing coordinate `ijk`; returns `nullptr` if not found or not active [S1].
7.  **Dimension and Active Check**: Perform a fused tree walk to return both the HDDA (Hierarchical Distance Driven Adaptive) step size and the active state of `ijk`, avoiding a second descent, cache-check, or leaf `ValueMask` lookup [S1].
8.  **Generic Get/Set**: Use `get<OpT>(ijk, args...)` to apply an operation `OpT` to the value at `ijk`, and `set<OpT>(ijk, args...)` to modify it [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ReadAccessor:
    def __init__(self, root):
        # Initialize with root node. Cache is empty.
        self.root = root
        self.cache = {} # Internal cache structure, details not specified [S1]

    def clear(self):
        # Reset access state to initial state with empty cache [S1]
        self.cache = {}

    def getValue(self, ijk):
        # Retrieve value at coordinate ijk [S1]
        # Implementation involves tree traversal and cache lookup
        pass

    def isActive(self, ijk):
        # Determine if coordinate ijk is active [S1]
        pass

    def probeValue(self, ijk, v_ref):
        # Attempt to retrieve value into v_ref. Returns True if successful [S1]
        # Returns False if not found or inactive
        pass

    def probeLeaf(self, ijk):
        # Attempt to retrieve pointer to LeafT node containing ijk [S1]
        # Returns nullptr if not found or not active
        pass

    def getDimAndActive(self, ijk, ray, policy=ActiveExact):
        # Fused tree walk returning HDDA step size and active state [S1]
        # Avoids second descent/cache-check/leaf ValueMask lookup
        pass

    def get(self, ijk, op, *args):
        # Apply operation OpT to value at ijk [S1]
        pass

    def set(self, ijk, op, *args):
        # Modify value at ijk using operation OpT [S1]
        pass

# Friend classes allow nodes to insert themselves into the cache [S1]
# class RootNode:
#     def insert_into_cache(accessor, node): ...
# class InternalNode:
#     def insert_into_cache(accessor, node): ...
# class LeafNode:
#     def insert_into_cache(accessor, node): ...
```

**Edge Cases and Pitfalls:**
*   **Friend Access**: `RootNode`, `InternalNode`, and `LeafNode` classes are friends of `ReadAccessor` to allow nodes to insert themselves into the cache [S1]. This is a critical implementation detail for cache population [S1].
*   **Policy Contract**: The `getDimAndActive` method relies on the `Policy` template parameter (`ActiveExact` or `ActiveOnLeafOnly`) to define the contract for determining active states [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The single-level caching (`CacheLevels = 1`) implies a trade-off between memory usage and traversal speed compared to multi-level caching [S1].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths**: Optimizes access to grid values and leaf nodes without traversing the full tree hierarchy for every query [S1]. The fused `getDimAndActive` operation shaves off a second descent, cache-check, or leaf `ValueMask` lookup [S1].
*   **Limitations**: Caches only a single level, which may result in more frequent tree traversals for deeply nested structures compared to multi-level caching [S1].

**When to Use:**
Use when memory is constrained but frequent access to leaf nodes or specific tree levels is expected [S1].

**When Not to Use:**
Not specified in the sources.

## Background
**Origin:**
The `ReadAccessor` class template is part of the OpenVDB/NanoVDB library [S1]. It is documented in the OpenVDB 13.1.0 documentation [S1].

**History and Development:**
Not given in the sources.

**Usage:**
Used in OpenVDB/NanoVDB for efficient volume data access [S1].

## Sources
[S1] OpenVDB: ReadAccessor< BuildT, LEVEL0, -1, -1 > Class Template Reference, https://openvdb.org/documentation/doxygen/classnanovdb_1_1ReadAccessor_3_01BuildT_00_01LEVEL0_00_01-1_00_01-1_01_4.html, fetched 2026-10-24
