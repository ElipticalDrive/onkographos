# VolumeHDDA

Also called: Hierarchical DDA for Generic Volumes, Hierarchical Digital Differential Analyzer for Volume, Volume Hierarchical DDA (VolumeHDDA), VolumeHDDA (Hierarchical DDA for Volumes), VolumeHDDA< TreeT, RayT, 0 >, ChildLevel>, ChildNodeLevel>

## Overview
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer (HDDA) designed for ray intersections against generic volume trees [S1, S2, unofficial]. It traverses from a specified `ChildNodeLevel` down to the leaf level (specialized at 0), performing hierarchical ray-marching by entering child nodes when present and processing active/inactive tiles at leaf levels [S1, S2, unofficial]. It sets itself apart from its sibling [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by being explicitly templated for volume tree structures (handling `TreeT`, `RayT`, and `ChildNodeLevel`) and managing time spans for active/inactive tile sequences, whereas the generic HDDA focuses on geometric intersection tests [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A `RayT` object representing the ray with methods `valid()`, `setTimes(t0, t1)`, `invDir()`, and `operator()(time)` [S1, S2, unofficial]. An `AccessorT` object providing `probeConstNode<NodeT>(coord)` and `isValueOn(coord)` [S1, S2, unofficial]. A `TimeSpanT` object `t` tracking the current active segment, initialized to `(-1, -1)` [S1, S2, unofficial].
*   **Outputs:** For `march`, a `TimeSpanT` representing the first continuous active segment or `(-1, -1)` if none [S1, S2, unofficial]. For `hits`, a list of `TimeSpanT` intervals collected into `ListT` [S1, S2, unofficial].

**Data Structures:**
The class `VolumeHDDA<TreeT, RayT, int ChildNodeLevel>` contains private members `math::DDA<RayT, NodeT::TOTAL> mDDA` and `VolumeHDDA<TreeT, RayT, ChildNodeLevel-1> mHDDA` [S1, S2, unofficial]. `NodeT` is defined as `typename ChainT::template Get<ChildNodeLevel>`, where `ChainT` is `typename TreeT::RootNodeType::NodeChainType` [S1, S2, unofficial].

**Parameters:**
*   `TreeT`: The tree type being traversed [S2, unofficial].
*   `RayT`: The ray type used for traversal [S2, unofficial].
*   `ChildNodeLevel`: Template argument specifying the entry upper node level for hierarchical ray-marching; the final lowest level is always the leaf node level (specialized at 0), not the voxel level [S1, S2, unofficial].
*   `LeafT`: Defined as `typename TreeT::LeafNodeType` [S2, unofficial].
*   `TimeSpanT`: Defined as `typename RayT::TimeSpan` [S2, unofficial].

**Algorithm Steps:**
The traversal uses a recursive descent where the general case (`ChildNodeLevel > 0`) and the base case (`ChildNodeLevel == 0`) share similar logic but differ in node probing [S1, S2, unofficial].
1. **Initialization:** Initialize `mDDA` with the ray [S1, S2, unofficial]. In the base case specialization, `mDDA` is initialized with `LeafT::TOTAL` [S1, unofficial].
2. **Loop:** Iterate using `mDDA.step()` [S1, S2, unofficial].
    * **Child Node Check:** General case checks `acc.probeConstNode<NodeT>(mDDA.voxel())` for non-null [S1, S2, unofficial]. Base case checks `acc.probeConstNode<LeafT>(mDDA.voxel())` or `acc.isValueOn(mDDA.voxel())` [S1, S2, unofficial]. Action: Set ray times to `(mDDA.time(), mDDA.next())` and recursively call `mHDDA.march` (or `hits`) [S1, S2, unofficial]. If the recursive call returns `true` (for `march`) or collects data (for `hits`), handle termination/collection and continue or return [S1, S2, unofficial].
    * **Active Tile Check:** If `acc.isValueOn(mDDA.voxel())` is true, set `t.t0 = mDDA.time()` if `t.t0 < 0` [S1, S2, unofficial].
    * **Inactive Tile / Segment Closure:** If `t.t0 >= 0` (an active segment is open) and the current tile is inactive, set `t.t1 = mDDA.time()` [S1, S2, unofficial]. If `t.valid()` (both `t0` and `t1` are set), return `true` (for `march`) or push `t` to `times` (for `hits`) [S1, S2, unofficial]. Reset `t.set(-1, -1)` [S1, S2, unofficial].
3. **Termination:** After the loop, if `t.t0 >= 0` (the ray ended while in an active segment), set `t.t1 = mDDA.maxTime()` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Self-contained type definitions required for implementation
class TimeSpan:
    def __init__(self, t0, t1):
        self.t0 = t0
        self.t1 = t1
    def valid(self):
        return self.t0 >= 0 and self.t1 >= 0
    def set(self, t0, t1):
        self.t0 = t0
        self.t1 = t1

class DDA:
    def __init__(self, ray, log2dim):
        self.ray = ray
        self.log2dim = log2dim
        self.voxel_coord = None
        self.current_time = 0.0
        self.next_time = 0.0
        self.max_time = ray.maxTime()
        self._init_dda_state()  # [G] Standard DDA initialization for ray and node size 2^log2dim

    def init(self, ray):
        self.ray = ray
        self._init_dda_state()

    def voxel(self):
        return self.voxel_coord

    def time(self):
        return self.current_time

    def next(self):
        return self.next_time

    def maxTime(self):
        return self.max_time

    def step(self):
        # [G] Advance DDA state to next voxel along ray; returns False when ray exits bounds
        pass

# General Case: VolumeHDDA<TreeT, RayT, ChildNodeLevel> where ChildNodeLevel > 0
class VolumeHDDA:
    def __init__(self, tree, ray_type, child_node_level):
        self.child_node_level = child_node_level
        self.mDDA = DDA(ray_type, tree.node_size_log2(child_node_level))
        self.mHDDA = VolumeHDDA(tree, ray_type, child_node_level - 1) if child_node_level > 0 else None
        self.NodeT = tree.get_node_type(child_node_level)

    def march(self, ray, acc):
        t = TimeSpan(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(self.NodeT, voxel) is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                if self.mHDDA._march_recursive(ray, acc, t):
                    return True
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
            if not self.mDDA.step():
                break
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(self.NodeT, voxel) is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                self.mHDDA.hits(ray, acc, times, t)
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.append(t)
                t.set(-1, -1)
            if not self.mDDA.step():
                break
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()

# Base Case Specialization: VolumeHDDA<TreeT, RayT, 0>
class VolumeHDDALeaf:
    def __init__(self, tree, ray_type):
        self.mDDA = DDA(ray_type, tree.leaf_size_log2)
        self.LeafT = tree.leaf_node_type

    def march(self, ray, acc):
        t = TimeSpan(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(self.LeafT, voxel) is not None or acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
            if not self.mDDA.step():
                break
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(self.LeafT, voxel) is not None or acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.append(t)
                t.set(-1, -1)
            if not self.mDDA.step():
                break
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
```

**Edge Cases and Pitfalls:**
*   **Time Span Management:** The `TimeSpanT` `t` is initialized to `(-1, -1)` [S1, S2, unofficial]. `t.t0` marks the start of an active segment, `t.t1` marks the end [S1, S2, unofficial]. If an inactive tile is hit after an active segment, the segment is closed [S1, S2, unofficial]. If the ray ends while still in an active segment (`t.t0 >= 0` after loop), `t.t1` is set to `maxTime` [S1, S2, unofficial].
*   **Precision:** The DDA relies on floating-point arithmetic for ray parameterization [G]. Precision issues may arise with very long rays or small nodes, though the DDA implementation typically handles this via clamping [G].
*   **Boundaries:** The ray must be valid (`ray.valid()`) before traversal [S1, S2, unofficial]. If the ray does not intersect the volume bounds, the loop may not execute or return immediately [S1, S2, unofficial].

## Subtopics

### [HDDA Iterator Reorganization](5239-hdda-iterator-reorganization.md)
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators that separates the traversal of all active values from the traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations. It sets itself apart from its parent [VolumeHDDA](2353-volumehdda.md) by providing a specialized leaf-only traversal path that avoids processing active coarse tiles, addressing the latent mis-indexing bugs that occurred when callers assumed the previous single iterator yielded only leaf voxels.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of nodes visited along the ray path, skipping inactive subtrees [S1, S2, unofficial]. Exact asymptotic bounds are Not given in the sources.
*   **Memory Complexity:** Low, as it uses recursion and does not store the entire traversal path [G]. The stack depth is proportional to the tree depth [G]. Exact memory limits are Not given in the sources.
*   **Strengths:** Efficient for sparse volumes due to hierarchical skipping [S1, S2, unofficial]. Handles active/inactive tile sequences correctly [S1, S2, unofficial].
*   **Limitations:** Overhead of recursive calls and template instantiation [G]. Performance depends on the tree structure and ray direction [G].
*   **When to use:** When traversing volume trees for ray-marching or intersection tests, especially when dealing with sparse data or needing to skip inactive regions [S1, S2, unofficial].
*   **When not to use:** For dense volumes where linear traversal might be simpler or for non-tree-based volume representations [G].

## Background
The class is defined in `openvdb/math/DDA.h` [S1, S2, unofficial]. Historical development, specific use cases beyond volume tree traversal, and organizational origins are Not given in the sources.

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
[S3] GitHub - openvdbdev/openvdb, https://github.com/openvdbdev/openvdb, fetched 2026-10-27 (unofficial)
