# VolumeHDDA

Also called: Hierarchical Digital Differential Analyzer for Volume, Volume Hierarchical DDA (VolumeHDDA), VolumeHDDA (Hierarchical DDA for Volumes), VolumeHDDA< TreeT, RayT, 0 >, ChildLevel>, ChildNodeLevel>

## Overview
VolumeHDDA is a hierarchical ray traversal algorithm for sparse volume trees (such as OpenVDB) that combines the analytical boundary-crossing logic of the Digital Differential Analyzer (DDA) with tree-aware node skipping [S1, unofficial][S2, unofficial]. Unlike flat DDA methods that visit every voxel along a ray, VolumeHDDA recursively descends into child nodes only when present, allowing it to skip large inactive regions (tiles) efficiently [S1, unofficial][S2, unofficial]. It is specialized for determining active ray segments (time spans) within the volume, distinguishing between active data and inactive empty space [S1, unofficial][S2, unofficial].

## How it works
VolumeHDDA operates by traversing a tree structure from a specified upper node level down to the leaf level. It uses a base DDA instance to step through grid cells and an accessor to query the tree topology and data values.

### Inputs and Outputs
*   **Inputs:**
    *   `ray`: A ray object with methods `valid()`, `setTimes(t0, t1)`, `time()`, `next()`, `maxTime()`, and `invDir()`.
    *   `acc`: An accessor object providing `probeConstNode<NodeT>(coord)` to check for child nodes and `isValueOn(coord)` to check if a tile/voxel is active.
    *   `ChildNodeLevel`: A template parameter defining the starting level of the hierarchy.
*   **Outputs:**
    *   `TimeSpanT`: A structure containing `t0` (start) and `t1` (end) of active ray segments. Initialized to `(-1, -1)` to indicate invalid/empty.

### Data Structures
The class `VolumeHDDA<TreeT, RayT, ChildNodeLevel>` contains two private members [S1, unofficial][S2, unofficial]:
1.  `mDDA`: An instance of `math::DDA<RayT, NodeT::TOTAL>`. This handles the geometric stepping through the grid cells at the current node level. `NodeT::TOTAL` is the log2 dimension of the node at `ChildNodeLevel`.
2.  `mHDDA`: An instance of `VolumeHDDA<TreeT, RayT, ChildNodeLevel-1>`. This handles the recursive descent to the next lower level.

For the leaf specialization `VolumeHDDA<TreeT, RayT, 0>`:
1.  `mDDA`: An instance of `math::DDA<RayT, LeafT::TOTAL>`.
2.  No recursive `mHDDA` member exists.

### Algorithm Steps

#### 1. Initialization
The traversal begins by initializing the DDA instance with the ray [S1, unofficial][S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
mDDA.init(ray)
```
The `TimeSpanT t` is initialized to `(-1, -1)` [S1, unofficial][S2, unofficial].

#### 2. Traversal Loop
The algorithm iterates using `mDDA.step()` until the ray exits the current node's bounds or the loop terminates early [S1, unofficial][S2, unofficial].

Inside the loop, for each voxel coordinate `mDDA.voxel()`:

1.  **Check for Child Node:**
    If `acc.probeConstNode<NodeT>(mDDA.voxel())` returns a non-null pointer, a child node exists at this location [S1, unofficial][S2, unofficial].
    *   Update the ray's time bounds to the current node's entry and exit times: `ray.setTimes(mDDA.time(), mDDA.next())` [S1, unofficial][S2, unofficial].
    *   Recursively call the lower-level HDDA (`mHDDA.march` or `mHDDA.hits`) with the updated ray [S1, unofficial][S2, unofficial].
    *   If the recursive call returns `true` (for `march`) or successfully adds a time span (for `hits`), the current operation may terminate or continue depending on the mode. In `march`, return `true` immediately [S1, unofficial][S2, unofficial]. In `hits`, continue to collect other segments [S1, unofficial][S2, unofficial].

2.  **Check for Active Tile:**
    If no child node exists, check if the current tile/voxel is active using `acc.isValueOn(mDDA.voxel())` [S1, unofficial][S2, unofficial].
    *   If active and `t.t0 < 0` (no active segment currently open):
        *   Set `t.t0 = mDDA.time()` [S1, unofficial][S2, unofficial].

3.  **Check for Inactive Tile (Closing Segment):**
    If the tile is inactive (`!acc.isValueOn`) and an active segment is open (`t.t0 >= 0`):
    *   Set `t.t1 = mDDA.time()` [S1, unofficial][S2, unofficial].
    *   If `t.valid()` (both `t0` and `t1` are set):
        *   For `march`: Return `true` [S1, unofficial][S2, unofficial].
        *   For `hits`: Push `t` to the output list `times` [S1, unofficial][S2, unofficial].
    *   Reset `t` to `(-1, -1)` [S1, unofficial][S2, unofficial].

4.  **Step:**
    Call `mDDA.step()` to advance to the next voxel [S1, unofficial][S2, unofficial]. If `mDDA.step()` returns `false`, exit the loop [S1, unofficial][S2, unofficial].

#### 3. Finalization
After the loop terminates (ray exits node bounds):
*   If an active segment is still open (`t.t0 >= 0`):
    *   Set `t.t1 = mDDA.maxTime()` [S1, unofficial][S2, unofficial].
    *   For `march`: Return `false` (segment extends to end of node) [S1, unofficial][S2, unofficial].
    *   For `hits`: Push `t` to the output list `times` [S1, unofficial][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# General Case: VolumeHDDA<TreeT, RayT, ChildNodeLevel>
class VolumeHDDA:
    def __init__(self, ChildNodeLevel):
        self.ChildNodeLevel = ChildNodeLevel
        self.mDDA = DDA(RayT, NodeT::TOTAL)
        if ChildNodeLevel > 0:
            self.mHDDA = VolumeHDDA(ChildNodeLevel - 1)

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if not ray.valid():
            return t
        self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for child node
            child = acc.probeConstNode(NodeT, voxel)
            if child is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                if self.mHDDA._march_recursive(ray, acc, t):
                    return True
            
            # Check for active tile
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile closing an active segment
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
        
        # End of node reached
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times):
        t = TimeSpanT(-1, -1)
        if not ray.valid():
            return
        self._hits_recursive(ray, acc, times, t)

    def _hits_recursive(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for child node
            child = acc.probeConstNode(NodeT, voxel)
            if child is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                self.mHDDA._hits_recursive(ray, acc, times, t)
            
            # Check for active tile
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile closing an active segment
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.push_back(t)
                t.set(-1, -1)
        
        # End of node reached
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
            times.push_back(t)

# Base Case Specialization: VolumeHDDA<TreeT, RayT, 0>
class VolumeHDDALeaf:
    def __init__(self):
        self.mDDA = DDA(RayT, LeafT::TOTAL)

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if not ray.valid():
            return t
        self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for leaf node or active tile
            # Note: probeConstNode<LeafT> might return non-null if leaf structure exists,
            # but isValueOn checks the data value.
            has_leaf = acc.probeConstNode(LeafT, voxel) is not None
            is_active = acc.isValueOn(voxel)
            
            if has_leaf or is_active:
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
        
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times):
        t = TimeSpanT(-1, -1)
        if not ray.valid():
            return
        self._hits_recursive(ray, acc, times, t)

    def _hits_recursive(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            has_leaf = acc.probeConstNode(LeafT, voxel) is not None
            is_active = acc.isValueOn(voxel)
            
            if has_leaf or is_active:
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.push_back(t)
                t.set(-1, -1)
        
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
            times.push_back(t)
```

### Parameters
*   **ChildNodeLevel**: Specifies the entry upper node level for hierarchical ray-marching. The recursion stops at level 0 (leaf level) [S1, unofficial][S2, unofficial].
*   **TreeT**: The tree type being traversed.
*   **RayT**: The ray type used for traversal.
*   **LeafT**: Defined as `typename TreeT::LeafNodeType` [S2, unofficial].

### Edge Cases and Pitfalls
*   **Time Span Management**: The `TimeSpanT` object `t` tracks active segments. It is initialized to `(-1, -1)`. `t.t0` marks the start of an active segment, and `t.t1` marks the end. If an inactive tile is hit after an active segment, the segment is closed. If the ray ends while still in an active segment (`t.t0 >= 0` after the loop), `t.t1` is set to `maxTime()` [S1, unofficial][S2, unofficial].
*   **Leaf vs. Tile**: In the leaf specialization, the condition `acc.probeConstNode<LeafT>(voxel) || acc.isValueOn(voxel)` handles both cases where a leaf node structure exists and where the tile is active [S1, unofficial][S2, unofficial].

## Subtopics

### HDDA Iterator Reorganization
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators to separate traversal of all active values from traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations [S3, unofficial].
*   **HDDAActiveValueIterator**: Walks every active value node, including active coarse tiles (`getDim > 1`) and active leaf voxels (`getDim == 1`) [S3, unofficial].
*   **HDDALeafVoxelIterator**: Yields only active leaf voxels. Skips active tiles in a single coarse HDDA step [S3, unofficial].
*   **Implementation**: Both are aliases over `HDDAValueIteratorImpl<AccT, ScalarT, bool LeafOnly>`. `LeafOnly` is a compile-time gate [S3, unofficial].
*   **Optimizations**:
    *   `nextSegment`: Uses predicated `TimeSpan` setters instead of an `if/else` ladder, reducing warp divergence [S3, unofficial].
    *   `nextVoxel`: Uses a bounded `for (pass < 3)` level-realign loop instead of three unrolled passes, collapsing to a single descent in the common level-aligned case [S3, unofficial].
*   **Pitfall**: Previously, callers indexing per-leaf-voxel buffers (like `gridScalars`) walked tiles they never needed, causing latent mis-indexing. The leaf iterator fixes this [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Proportional to the number of active nodes and tiles visited along the ray, rather than the number of voxels. In sparse volumes, this is significantly faster than flat DDA [S1, unofficial][S2, unofficial].
*   **Memory Complexity**: Low, as it only stores the DDA state and recursion stack. The recursion depth is bounded by the tree height [S1, unofficial][S2, unofficial].
*   **Strengths**: Efficiently skips large inactive regions (tiles). Integrates seamlessly with sparse tree structures like OpenVDB [S1, unofficial][S2, unofficial].
*   **Limitations**: Overhead of recursive calls and node probing. Less efficient than flat DDA for dense volumes where most tiles are active [S1, unofficial][S2, unofficial].
*   **When to Use**: Sparse volumes with large inactive regions. When only active segments are needed [S1, unofficial][S2, unofficial].
*   **When Not to Use**: Dense volumes or when every voxel must be visited regardless of activity [S1, unofficial][S2, unofficial].

## Background
VolumeHDDA is part of the OpenVDB library's mathematical utilities, specifically in `openvdb/math/DDA.h` [S1, unofficial][S2, unofficial]. It was developed to provide efficient ray intersection capabilities for sparse volumetric data, building upon the standard DDA algorithm [S1, unofficial][S2, unofficial]. The hierarchical approach allows it to leverage the sparse nature of the data, making it suitable for large-scale simulations and visual effects [S1, unofficial][S2, unofficial]. The iterator reorganization described in [S3, unofficial] represents a later optimization to improve performance and correctness for leaf-specific operations.

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
[S3] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
