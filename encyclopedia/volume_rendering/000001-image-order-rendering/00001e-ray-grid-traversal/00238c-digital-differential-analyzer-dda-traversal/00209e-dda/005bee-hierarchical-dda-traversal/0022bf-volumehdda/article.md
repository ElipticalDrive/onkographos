# VolumeHDDA

Also called: Hierarchical Digital Differential Analyzer for Volume, Volume Hierarchical DDA (VolumeHDDA), VolumeHDDA (Hierarchical DDA for Volumes), VolumeHDDA< TreeT, RayT, 0 >, ChildLevel>, ChildNodeLevel>

## Overview
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer (HDDA) designed for ray intersections against generic volume trees [S1, S2, unofficial]. It traverses from a specified `ChildNodeLevel` down to the leaf level (specialized at 0), performing hierarchical ray-marching by entering child nodes when present and processing active/inactive tiles at leaf levels [S1, S2, unofficial]. It sets itself apart from its sibling [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by being explicitly templated for volume tree structures (handling `TreeT`, `RayT`, and `ChildNodeLevel`) and managing time spans for active/inactive tile sequences, whereas the generic HDDA focuses on geometric intersection tests [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A `RayT` object (representing the ray with methods `valid()`, `setTimes(t0, t1)`, `invDir()`, `operator()(time)`), an `AccessorT` object (providing `probeConstNode<NodeT>(coord)` and `isValueOn(coord)`), and a `TimeSpanT` object `t` (tracking the current active segment, initialized to `(-1, -1)`).
*   **Outputs:** For `march`, a `TimeSpanT` representing the first continuous active segment or `(-1, -1)` if none. For `hits`, a list of `TimeSpanT` intervals collected into `ListT`.

**Data Structures:**
The class `VolumeHDDA<TreeT, RayT, int ChildNodeLevel>` contains:
*   `math::DDA<RayT, NodeT::TOTAL> mDDA`: A DDA instance for the current node level. `NodeT` is defined as `typename ChainT::template Get<ChildNodeLevel>`, where `ChainT` is `typename TreeT::RootNodeType::NodeChainType`.
*   `VolumeHDDA<TreeT, RayT, ChildNodeLevel-1> mHDDA`: A recursive instance for the next lower level.

**Parameters:**
*   `TreeT`: The tree type being traversed [S2, unofficial].
*   `RayT`: The ray type used for traversal [S2, unofficial].
*   `ChildNodeLevel`: Template argument specifying the entry upper node level for hierarchical ray-marching. The final lowest level is always the leaf node level (specialized at 0), not the voxel level [S1, S2, unofficial].
*   `LeafT`: Defined as `typename TreeT::LeafNodeType` [S2, unofficial].
*   `TimeSpanT`: Defined as `typename RayT::TimeSpan` [S2, unofficial].

**Algorithm Steps:**
The traversal uses a recursive descent. The general case (for `ChildNodeLevel > 0`) and the base case (`ChildNodeLevel == 0`) share similar logic but differ in node probing.

1.  **Initialization:** Initialize `mDDA` with the ray. In the base case specialization, `mDDA` is initialized with `LeafT::TOTAL` [S1, unofficial].
2.  **Loop:** Iterate using `mDDA.step()`.
    *   **Child Node Check:**
        *   General Case: If `acc.probeConstNode<NodeT>(mDDA.voxel())` is not `nullptr` [S1, S2, unofficial].
        *   Base Case: If `acc.probeConstNode<LeafT>(mDDA.voxel())` is not `nullptr` OR `acc.isValueOn(mDDA.voxel())` [S1, S2, unofficial].
        *   Action: Set ray times to `(mDDA.time(), mDDA.next())`. Recursively call `mHDDA.march` (or `hits`). If the recursive call returns `true` (for `march`) or collects data (for `hits`), handle termination/collection and continue or return.
    *   **Active Tile Check:** If `acc.isValueOn(mDDA.voxel())` is true:
        *   If `t.t0 < 0`: Set `t.t0 = mDDA.time()` [S1, S2, unofficial].
    *   **Inactive Tile / Segment Closure:** If `t.t0 >= 0` (an active segment is open) and the current tile is inactive (or the child node check failed and we are in an inactive region):
        *   Set `t.t1 = mDDA.time()` [S1, S2, unofficial].
        *   If `t.valid()` (both `t0` and `t1` are set):
            *   For `march`: Return `true` [S1, S2, unofficial].
            *   For `hits`: Push `t` to `times` [S1, S2, unofficial].
        *   Reset `t.set(-1, -1)` [S1, S2, unofficial].
3.  **Termination:** After the loop, if `t.t0 >= 0` (the ray ended while in an active segment), set `t.t1 = mDDA.maxTime()` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# General Case: VolumeHDDA<TreeT, RayT, ChildNodeLevel> where ChildNodeLevel > 0
class VolumeHDDA:
    def __init__(self, TreeT, RayT, ChildNodeLevel):
        self.mDDA = DDA(RayT, NodeT.TOTAL)
        self.mHDDA = VolumeHDDA(TreeT, RayT, ChildNodeLevel - 1)
        self.NodeT = ChainT.Get(ChildNodeLevel)

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            
            # Check for child node
            if acc.probeConstNode(self.NodeT, voxel) is not None:
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
                    times.push_back(t)
                t.set(-1, -1)
            
            if not self.mDDA.step():
                break
        
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()

# Base Case Specialization: VolumeHDDA<TreeT, RayT, 0>
class VolumeHDDALeaf:
    def __init__(self, TreeT, RayT):
        self.mDDA = DDA(RayT, LeafT.TOTAL)
        self.LeafT = TreeT.LeafNodeType

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while True:
            voxel = self.mDDA.voxel()
            
            # Check for leaf node or active tile
            if acc.probeConstNode(self.LeafT, voxel) is not None or acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile closing an active segment
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
                    times.push_back(t)
                t.set(-1, -1)
            
            if not self.mDDA.step():
                break
        
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
```

**Edge Cases and Pitfalls:**
*   **Time Span Management:** The `TimeSpanT` `t` is initialized to `(-1, -1)`. `t.t0` marks the start of an active segment, `t.t1` marks the end. If an inactive tile is hit after an active segment, the segment is closed. If the ray ends while still in an active segment (`t.t0 >= 0` after loop), `t.t1` is set to `maxTime` [S1, S2, unofficial].
*   **Precision:** The DDA relies on floating-point arithmetic for ray parameterization. Precision issues may arise with very long rays or small nodes, though the DDA implementation typically handles this via clamping [G].
*   **Boundaries:** The ray must be valid (`ray.valid()`) before traversal. If the ray does not intersect the volume bounds, the loop may not execute or return immediately [S1, S2, unofficial].

## Subtopics

### [HDDA Iterator Reorganization](5239-hdda-iterator-reorganization.md)
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators that separates the traversal of all active values from the traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations. It sets itself apart from its parent [VolumeHDDA](2353-volumehdda.md) by providing a specialized leaf-only traversal path that avoids processing active coarse tiles, addressing the latent mis-indexing bugs that occurred when callers assumed the previous single iterator yielded only leaf voxels.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of nodes visited along the ray path, skipping inactive subtrees. The hierarchical nature reduces the number of visited nodes compared to linear traversal, especially for sparse volumes.
*   **Memory Complexity:** Low, as it uses recursion and does not store the entire traversal path. The stack depth is proportional to the tree depth.
*   **Strengths:** Efficient for sparse volumes due to hierarchical skipping. Handles active/inactive tile sequences correctly.
*   **Limitations:** Overhead of recursive calls and template instantiation. Performance depends on the tree structure and ray direction.
*   **When to use:** When traversing volume trees for ray-marching or intersection tests, especially when dealing with sparse data or needing to skip inactive regions.
*   **When not to use:** For dense volumes where linear traversal might be simpler or for non-tree-based volume representations.

## Background
VolumeHDDA is part of the OpenVDB library, developed by DreamWorks Animation and later by the Academy Software Foundation. It is designed for efficient storage and manipulation of large volumetric datasets, commonly used in visual effects and scientific visualization. The class is defined in `openvdb/math/DDA.h` [S1, S2, unofficial].

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
