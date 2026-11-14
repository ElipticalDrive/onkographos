# VolumeHDDA

Also called: Hierarchical Digital Differential Analyzer for Volume, Volume Hierarchical DDA (VolumeHDDA), VolumeHDDA (Hierarchical DDA for Volumes), VolumeHDDA< TreeT, RayT, 0 >, ChildLevel>, ChildNodeLevel>

## Overview
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer for traversing volume trees, templated on the tree type, ray type, and the starting node level for traversal [S1, unofficial][S2, unofficial]. It builds on [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by implementing a recursive ray-marching approach that enters child nodes when present and processes active or inactive tiles at leaf levels [S2, unofficial]. It sets itself apart from siblings like [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by focusing on discrete tile/voxel intersection detection and time-span management rather than continuous sampling, and from [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) by explicitly handling the hierarchical node structure via template recursion [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `RayT` object (representing the ray with methods `valid()`, `setTimes(t0, t1)`, `time()`, `next()`, `maxTime()`, and `invDir()`), an `AccessorT` object (providing `probeConstNode<NodeT>(voxel)` to check for child nodes and `isValueOn(voxel)` to check if a tile is active), and a `TimeSpanT` object `t` (initialized to `(-1, -1)`) to track the current active segment [S1, unofficial][S2, unofficial].
*   **Output:** For `march`, a `TimeSpanT` representing the first active segment found (or `(-1, -1)` if none); for `hits`, a list of `TimeSpanT` intervals [S1, unofficial][S2, unofficial].

**Data Structures:**
The class `VolumeHDDA<TreeT, RayT, int ChildNodeLevel>` contains:
*   `math::DDA<RayT, NodeT::TOTAL> mDDA`: A Digital Differential Analyzer instance for the current node level. `NodeT` is defined as `typename ChainT::template Get<ChildNodeLevel>`, where `ChainT` is `typename TreeT::RootNodeType::NodeChainType` [S1, unofficial][S2, unofficial].
*   `VolumeHDDA<TreeT, RayT, ChildNodeLevel-1> mHDDA`: A recursive instance for the next lower level [S1, unofficial][S2, unofficial].

**Parameters:**
*   `ChildNodeLevel`: Template argument specifying the entry upper node level for hierarchical ray-marching. The recursion terminates at level 0, which is specialized for leaf nodes [S1, unofficial][S2, unofficial].
*   `TreeT`: The tree type being traversed.
*   `RayT`: The ray type used for traversal.
*   `LeafT`: Defined as `typename TreeT::LeafNodeType` [S2, unofficial].
*   `TimeSpanT`: Defined as `typename RayT::TimeSpan` [S2, unofficial].

**Algorithm Steps:**
The algorithm implements two modes, `march` (find first active segment) and `hits` (collect all active segments), both following an iterative DDA loop with recursive descent [S1, unofficial][S2, unofficial].

1.  **Initialization:** Initialize `mDDA` with the input ray (`mDDA.init(ray)`). Initialize `TimeSpanT t` to `(-1, -1)` [S1, unofficial][S2, unofficial].
2.  **Loop:** While `mDDA.step()` returns true:
    a.  **Check for Child Node:** If `acc.probeConstNode<NodeT>(mDDA.voxel())` is not null (for general case) or `acc.probeConstNode<LeafT>(mDDA.voxel())` is not null (for leaf specialization):
        i.  Update ray times: `ray.setTimes(mDDA.time(), mDDA.next())` [S1, unofficial][S2, unofficial].
        ii. Recursively call `mHDDA.march(ray, acc, t)` (for `march`) or `mHDDA.hits(ray, acc, times, t)` (for `hits`) [S1, unofficial][S2, unofficial].
        iii. If `march` returns true, return true immediately [S1, unofficial][S2, unofficial].
        iv. If `hits`, the recursive call appends to `times`; continue loop [S1, unofficial][S2, unofficial].
    b.  **Check for Active Tile:** Else if `acc.isValueOn(mDDA.voxel())` is true:
        i.  If `t.t0 < 0` (start of new segment), set `t.t0 = mDDA.time()` [S1, unofficial][S2, unofficial].
    c.  **Check for Inactive Tile after Active:** Else if `t.t0 >= 0` (we are in an active segment and hit an inactive tile):
        i.  Set `t.t1 = mDDA.time()` (end of segment) [S1, unofficial][S2, unofficial].
        ii. If `t.valid()` (both `t0` and `t1` are set):
            *   For `march`: Return true [S1, unofficial][S2, unofficial].
            *   For `hits`: Push `t` to `times` [S1, unofficial][S2, unofficial].
        iii. Reset `t` to `(-1, -1)` [S1, unofficial][S2, unofficial].
3.  **Post-Loop:** If `t.t0 >= 0` (ray ended while still in an active segment), set `t.t1 = mDDA.maxTime()` [S1, unofficial][S2, unofficial].
4.  **Return:** For `march`, return false (if no segment found) or the final `t` if valid (handled by wrapper); for `hits`, return void [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
        self._voxel = None
        self._time = 0.0
        self._next = 0.0
        self._max_time = 1.0
        self._step_count = 0
    def init(self, ray):
        self.ray = ray
        self._step_count = 0
    def step(self):
        self._step_count += 1
        return self._step_count < 10  # Simplified loop bound for demonstration
    def voxel(self):
        return self._voxel
    def time(self):
        return self._time
    def next(self):
        return self._next
    def maxTime(self):
        return self._max_time

class VolumeHDDA:
    def __init__(self, child_node_level):
        self.child_node_level = child_node_level
        self.mDDA = DDA(None, 0)
        self.mHDDA = VolumeHDDA(child_node_level - 1) if child_node_level > 0 else None

    def march(self, ray, acc):
        t = TimeSpan(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(voxel) is not None:
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
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times):
        t = TimeSpan(-1, -1)
        self._hits_recursive(ray, acc, times, t)

    def _hits_recursive(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(voxel) is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                self.mHDDA._hits_recursive(ray, acc, times, t)
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.append(t)
                t.set(-1, -1)
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()

class VolumeHDDALeaf:
    def __init__(self):
        self.mDDA = DDA(None, 0)

    def march(self, ray, acc):
        t = TimeSpan(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(voxel) is not None or acc.isValueOn(voxel):
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
        t = TimeSpan(-1, -1)
        self._hits_recursive(ray, acc, times, t)

    def _hits_recursive(self, ray, acc, times, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            if acc.probeConstNode(voxel) is not None or acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.append(t)
                t.set(-1, -1)
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
```

**Edge Cases and Pitfalls:**
*   **Time Span Management:** The `TimeSpanT` object `t` tracks active ray segments and is initialized to `(-1, -1)` [S1, unofficial][S2, unofficial]. `t.t0` marks the start of an active segment and `t.t1` marks the end [S1, unofficial][S2, unofficial]. If an inactive tile is hit after an active segment, the segment is closed [S1, unofficial][S2, unofficial]. If the ray ends while still in an active segment (`t.t0 >= 0` after loop), `t.t1` is set to `maxTime()` [S1, unofficial][S2, unofficial].
*   **Leaf Specialization:** The base case specialization (`ChildNodeLevel = 0`) treats leaf nodes and active tiles similarly, checking `acc.probeConstNode<LeafT>(voxel) || acc.isValueOn(voxel)` [S1, unofficial][S2, unofficial].
*   **Ray Validity:** The public `march` and `hits` methods check `ray.valid()` before calling the recursive helper [S1, unofficial][S2, unofficial].

## Subtopics

### [HDDA Iterator Reorganization](5239-hdda-iterator-reorganization.md)
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators that separates the traversal of all active values from the traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations. It sets itself apart from its parent [VolumeHDDA](2353-volumehdda.md) by providing a specialized leaf-only traversal path that avoids processing active coarse tiles, addressing the latent mis-indexing bugs that occurred when callers assumed the previous single iterator yielded only leaf voxels.

## Operating characteristics
Not given in the sources.

## Background
VolumeHDDA is part of the OpenVDB library, specifically in the `openvdb::math` namespace [S1, unofficial][S2, unofficial]. It was introduced to provide an efficient way to traverse hierarchical volume grids using a Digital Differential Analyzer approach [S1, unofficial][S2, unofficial]. The method is templated to support generic tree and ray types, making it flexible for various volume representations [S1, unofficial][S2, unofficial].

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
