# VolumeHDDA

Also called: Hierarchical Digital Differential Analyzer for Volume, Volume Hierarchical DDA (VolumeHDDA), VolumeHDDA (Hierarchical DDA for Volumes), VolumeHDDA< TreeT, RayT, 0 >, ChildLevel>, ChildNodeLevel>

## Overview
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer for traversing sparse voxel trees (such as OpenVDB grids) to find ray intersections with active values [S1, S2, unofficial]. It solves the problem of efficiently identifying contiguous active segments along a ray in a hierarchical data structure, skipping inactive regions and coarse tiles when necessary [S1, S2, unofficial]. It differs from its siblings by explicitly templating on a `ChildNodeLevel` to define the entry point for hierarchical recursion, ensuring the traversal terminates at the leaf node level rather than the voxel level [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `RayT` object (representing the ray origin, direction, and time bounds), an `AccessorT` object (providing access to the tree nodes and values), and a `TimeSpanT` object `t` (initialized to `(-1, -1)`) to track the current active segment [S1, S2, unofficial].
- **Output:** For `march`, a `TimeSpanT` representing the first active segment found (or `(-1, -1)` if none); for `hits`, a list of `TimeSpanT` objects containing all active segments [S1, S2, unofficial].

**Data Structures:**
The class `VolumeHDDA<TreeT, RayT, int ChildNodeLevel>` contains two private members [S1, S2, unofficial]:
1. `math::DDA<RayT, NodeT::TOTAL> mDDA`: A Digital Differential Analyzer instance specialized for the node size at `ChildNodeLevel` [S2, unofficial].
2. `VolumeHDDA<TreeT, RayT, ChildNodeLevel-1> mHDDA`: A recursive instance for the next lower level in the hierarchy [S1, S2, unofficial].

**Type Definitions:**
- `ChainT`: `typename TreeT::RootNodeType::NodeChainType` [S1, S2, unofficial].
- `NodeT`: `typename ChainT::template Get<ChildNodeLevel>` [S1, S2, unofficial].
- `LeafT`: `typename TreeT::LeafNodeType` (only in specialization `ChildNodeLevel=0`) [S1, S2, unofficial].
- `TimeSpanT`: `typename RayT::TimeSpan` [S1, S2, unofficial].

**Algorithm Steps:**
The traversal uses a DDA to step through nodes. For each voxel (node) visited by the DDA:
1. Initialize `mDDA` with the ray [S1, S2, unofficial].
2. Loop: Check if `acc.probeConstNode<NodeT>(mDDA.voxel())` returns a non-null pointer (child node exists) [S1, S2, unofficial].
   - Update the ray's time bounds to the current node's entry and exit times: `ray.setTimes(mDDA.time(), mDDA.next())` [S1, S2, unofficial].
   - Recursively call the appropriate method (`march` or `hits`) on `mHDDA` with the updated ray and accumulator [S1, S2, unofficial].
   - If `march` returns `true`, propagate `true` immediately (first segment found) [S1, S2, unofficial]. If `hits` is called, the recursive call appends segments to the `times` list [S1, S2, unofficial].
3. Check for Active Value: If no child node exists, check if the current node contains active values using `acc.isValueOn(mDDA.voxel())` [S1, S2, unofficial].
   - If active: If `t.t0 < 0` (start of a new segment), set `t.t0 = mDDA.time()` [S1, S2, unofficial].
4. Check for Inactive Value (Segment End): If the node is inactive (`!isValueOn`) and `t.t0 >= 0` (we are currently inside an active segment):
   - Set `t.t1 = mDDA.time()` to close the segment [S1, S2, unofficial].
   - If `t.valid()` (both `t0` and `t1` are set): For `march`, return `true`; for `hits`, push `t` to `times` [S1, S2, unofficial].
   - Reset `t` to `(-1, -1)` to prepare for the next segment [S1, S2, unofficial].
5. Step DDA: Call `mDDA.step()`. If it returns `false` (ray has exited the tree), exit the loop [S1, S2, unofficial].
6. Finalize Segment: After the loop, if `t.t0 >= 0` (the ray ended while still inside an active segment), set `t.t1 = mDDA.maxTime()` [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper types
class TimeSpanT:
    def __init__(self, t0, t1):
        self.t0 = t0
        self.t1 = t1
    def valid(self):
        return self.t0 >= 0 and self.t1 >= 0
    def set(self, t0, t1):
        self.t0 = t0
        self.t1 = t1

class DDA:
    def __init__(self, ray, node_total):
        self.ray = ray
        self.node_total = node_total
        self.current_voxel = None
        self.current_time = 0.0
        self.next_time = 0.0
        self.max_time = ray.max_time()
        self.done = False
    def init(self, ray):
        self.ray = ray
        self.done = False
    def step(self):
        if self.done: return False
        self.current_voxel = self._advance_voxel()
        self.current_time = self._get_entry_time()
        self.next_time = self._get_exit_time()
        if self.current_time > self.max_time:
            self.done = True
            return False
        return True
    def voxel(self): return self.current_voxel
    def time(self): return self.current_time
    def next(self): return self.next_time
    def maxTime(self): return self.max_time
    def _advance_voxel(self): pass
    def _get_entry_time(self): pass
    def _get_exit_time(self): pass

class NodeT:
    TOTAL = 1 << (ChildNodeLevel * 3)

class LeafT:
    TOTAL = 1

# General Case: VolumeHDDA<TreeT, RayT, ChildNodeLevel> where ChildNodeLevel > 0
class VolumeHDDA:
    def __init__(self, TreeT, RayT, ChildNodeLevel):
        self.mDDA = DDA(RayT, NodeT.TOTAL)
        self.mHDDA = VolumeHDDA(TreeT, RayT, ChildNodeLevel - 1)

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for child node
            if acc.probeConstNode(voxel, NodeT) is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                if self.mHDDA._march_recursive(ray, acc, t):
                    return True
            
            # Check for active tile
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile after active segment
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
        
        # Ray ended inside an active segment
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times):
        t = TimeSpanT(-1, -1)
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for child node
            if acc.probeConstNode(voxel, NodeT) is not None:
                ray.setTimes(self.mDDA.time(), self.mDDA.next())
                self.mHDDA.hits(ray, acc, times)
            
            # Check for active tile
            elif acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile after active segment
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    times.append(t)
                t.set(-1, -1)
        
        # Ray ended inside an active segment
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()

# Base Case Specialization: VolumeHDDA<TreeT, RayT, 0>
class VolumeHDDA_Specialized:
    def __init__(self, TreeT, RayT):
        self.mDDA = DDA(RayT, LeafT.TOTAL)

    def march(self, ray, acc):
        t = TimeSpanT(-1, -1)
        if ray.valid():
            self._march_recursive(ray, acc, t)
        return t

    def _march_recursive(self, ray, acc, t):
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            # Check for leaf node or active tile
            if acc.probeConstNode(voxel, LeafT) is not None or acc.isValueOn(voxel):
                if t.t0 < 0:
                    t.t0 = self.mDDA.time()
            
            # Check for inactive tile after active segment
            elif t.t0 >= 0:
                t.t1 = self.mDDA.time()
                if t.valid():
                    return True
                t.set(-1, -1)
        
        # Ray ended inside an active segment
        if t.t0 >= 0:
            t.t1 = self.mDDA.maxTime()
        return False

    def hits(self, ray, acc, times):
        t = TimeSpanT(-1, -1)
        self.mDDA.init(ray)
        while self.mDDA.step():
            voxel = self.mDDA.voxel()
            
            if acc.probeConstNode(voxel, LeafT) is not None or acc.isValueOn(voxel):
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

**Parameters:**
- `ChildNodeLevel`: Template argument specifying the entry upper node level for hierarchical ray-marching. The traversal always terminates at the leaf node level (specialization at 0), not the voxel level [S1, S2, unofficial].
- `TreeT`: The tree type being traversed [S2, unofficial].
- `RayT`: The ray type used for traversal [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Time Span Management:** The `TimeSpanT` object `t` tracks active ray segments. It is initialized to `(-1, -1)`. `t.t0` marks the start of an active segment, `t.t1` marks the end. If an inactive tile is hit after an active segment, the segment is closed. If the ray ends while still in an active segment (`t.t0 >= 0` after the loop), `t.t1` is set to `maxTime()` [S1, S2, unofficial].
- **Leaf vs. Voxel:** The base case specialization (`ChildNodeLevel=0`) operates on leaf nodes or tiles, not individual voxels. `LeafT::TOTAL` defines the DDA step size for the leaf node dimension [S1, S2, unofficial].

## Subtopics

### HDDA Iterator Reorganization
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators to separate traversal of all active values from traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations [S3, unofficial].
- **HDDAActiveValueIterator:** Walks every active value node, including active coarse tiles (`getDim > 1`) and active leaf voxels (`getDim == 1`) [S3, unofficial].
- **HDDALeafVoxelIterator:** Yields only active leaf voxels. Skips active tiles in a single coarse HDDA step [S3, unofficial].
- **Implementation:** Both are aliases over `HDDAValueIteratorImpl<AccT, ScalarT, bool LeafOnly>`. `LeafOnly` is a compile-time gate [S3, unofficial].
- **Optimizations:** `nextSegment` uses predicated `TimeSpan` setters to reduce warp divergence. `nextVoxel` uses a bounded `for (pass < 3)` level-realign loop instead of three unrolled passes, collapsing to a single descent in the common level-aligned case [S3, unofficial].
- **Pitfall:** Previously, callers indexing per-leaf-voxel buffers (like `gridScalars`) walked tiles they never needed, causing latent mis-indexing. The leaf iterator fixes this [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
VolumeHDDA is part of the OpenVDB library's math module, specifically designed for efficient ray traversal of sparse voxel grids [S1, S2, unofficial]. The recent reorganization of HDDA iterators (HDDAActiveValueIterator and HDDALeafVoxelIterator) aims to improve performance and correctness for leaf-specific operations, inspired by techniques from NanoVDB [S3, unofficial].

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
[S3] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
