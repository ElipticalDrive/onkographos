# OpenVDB DDA

Also called: Digital Differential Analyzer for OpenVDB, Digital Differential Analyzer specialized for OpenVDB grids, PointTreeMarcher, VDB DDA

## Overview
OpenVDB DDA is a Digital Differential Analyzer traversal algorithm specialized for OpenVDB grids, conceptually similar to Bresenham's line algorithm applied to a 3D ray intersecting OpenVDB nodes or voxels [S2, S5, unofficial][S6]. It iterates through grid elements intersected by a ray using a fixed step-size defined by the template parameter `Log2Dim` [S1][S5, unofficial]. Unlike Hierarchical DDA (HDDA) which uses variable step-sizes to skip empty space, OpenVDB DDA marches through every node or voxel boundary along the ray path [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   `ray`: A ray object of type `RayT` [S1][S2, unofficial]. The `RayT` template class must provide:
    *   `test(time)`: Evaluates the ray at a given time [S1].
    *   `t0()`: Returns the start time parameter of the ray [S1].
    *   `t1()`: Returns the end time parameter of the ray [S1].
    *   `invDir()`: Returns the inverse direction vector [S1].
    *   `operator()(time)`: Returns the 3D position vector at a given time parameter [S1].
*   `startTime`: The initial parameter value along the ray where traversal begins [S2, unofficial]. Defaults to `ray.t0()` if not provided [S3].
*   `maxTime`: The maximum parameter value along the ray, limiting the traversal distance [S2, unofficial]. Defaults to `ray.t1()` if not provided [S3].
*   `Log2Dim`: An integer template parameter specifying the log2 of the grid element dimension [S2, unofficial][S6].
    *   `Log2Dim = 0`: Traverses individual voxels [S2, unofficial][S6].
    *   `Log2Dim > 0`: Traverses tree nodes (or tiles) of size $2^{\text{Log2Dim}}$ [S2, unofficial][S6].

**Outputs:**
*   Iterative access to the current intersected voxel/node index via `voxel()` [S1][S2, unofficial].
*   Iterative access to the current hit time via `time()` [S1][S2, unofficial].
*   Iterative access to the next hit time via `next()` [S1][S2, unofficial].
*   Boolean return from `step()` indicating if traversal is within bounds [S1][S2, unofficial].

### Data Structures
The `DDA` class maintains the following private member variables [S2, S5, unofficial]:
*   `mT0`: Current time parameter (RealT) [S2, S5, unofficial].
*   `mT1`: Maximum time parameter (RealT) [S2, S5, unofficial].
*   `mVoxel`: Current voxel/node index coordinates (Coord, 3D signed integer) [S2, S5, unofficial].
*   `mStep`: Step size per axis for `mVoxel` (Coord, 3D signed integer) [S2, S5, unofficial].
*   `mDelta`: Time increment per axis for `mNext` (Vec3T, 3D float) [S2, S5, unofficial].
*   `mNext`: Time of the next intersection per axis (Vec3T, 3D float) [S2, S5, unofficial].

Type aliases used within the class [S1][S2, unofficial]:
*   `RealT`: `typename RayT::RealType` (e.g., `float` or `double`) [S1][S2, unofficial].
*   `Vec3T`: `typename RayT::Vec3Type` (3D vector of `RealT`) [S1][S2, unofficial].
*   `Coord`: 3D signed integer coordinate type [S2, unofficial].

### Initialization
The DDA state is initialized via the `init()` method or constructor [S1][S2, unofficial][S3].

1.  Set `mT0 = startTime` and `mT1 = maxTime` [S2, S5, unofficial].
2.  Calculate `DIM = 1 << Log2Dim` [S2, S5, unofficial].
3.  Get ray position `pos = ray(mT0)`, direction `dir = ray.dir()`, and inverse direction `inv = ray.invDir()` [S2, S5, unofficial].
4.  Compute initial voxel index `mVoxel` [S2, S5, unofficial]:
    $$mVoxel = \text{Coord::floor}(pos) \ \& \ (\sim(DIM - 1))$$
    This aligns the starting voxel to the grid boundary of size `DIM` [S2, S5, unofficial].
5.  For each axis `i` in $\{0, 1, 2\}$:
    *   If `dir[i]` is zero (including $\pm 0$) [S2, S5, unofficial]:
        *   `mStep[i] = 0` (dummy value) [S2, S5, unofficial].
        *   `mNext[i] = std::numeric_limits<RealT>::max()` (disabled) [S2, S5, unofficial].
        *   `mDelta[i] = std::numeric_limits<RealT>::max()` (dummy value) [S2, S5, unofficial].
    *   Else if `inv[i] > 0` (ray moving in positive direction) [S2, S5, unofficial]:
        *   `mStep[i] = DIM` [S2, S5, unofficial].
        *   `mNext[i] = mT0 + (mVoxel[i] + DIM - pos[i]) * inv[i]` [S2, S5, unofficial].
        *   `mDelta[i] = mStep[i] * inv[i]` [S2, S5, unofficial].
    *   Else (`inv[i] < 0`, ray moving in negative direction) [S2, S5, unofficial]:
        *   `mStep[i] = -DIM` [S2, S5, unofficial].
        *   `mNext[i] = mT0 + (mVoxel[i] - pos[i]) * inv[i]` [S2, S5, unofficial].
        *   `mDelta[i] = mStep[i] * inv[i]` [S2, S5, unofficial].

### Traversal Loop
The traversal proceeds by repeatedly stepping to the next intersected element [S1][S2, unofficial][S6].

1.  Query current hit time: `t_current = time()` returns `mT0` [S1][S2, unofficial].
2.  Query current voxel index: `voxel_idx = voxel()` returns `mVoxel` [S1][S2, unofficial].
3.  Process the voxel/node at `voxel_idx` (e.g., check occupancy, sample data) [S2, S4, unofficial].
4.  Step to next intersection:
    *   Find `stepAxis = argmin(mNext[0], mNext[1], mNext[2])` [S2, S5, unofficial].
    *   Update `mT0 = mNext[stepAxis]` [S2, S5, unofficial].
    *   Update `mNext[stepAxis] += mDelta[stepAxis]` [S2, S5, unofficial].
    *   Update `mVoxel[stepAxis] += mStep[stepAxis]` [S2, S5, unofficial].
    *   If `mT0 <= mT1`, continue; else, stop [S2, S5, unofficial].
5.  Repeat from step 1.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def is_zero(x):
    return abs(x) < 1e-9

def min_index(v):
    return min(range(len(v)), key=lambda i: v[i])

class OpenVDBDDA:
    def __init__(self, ray, start_time=None, max_time=None, log2_dim=0):
        self.ray = ray
        self.log2_dim = log2_dim
        self.dim = 1 << log2_dim
        if start_time is None:
            start_time = ray.t0()
        if max_time is None:
            max_time = ray.t1()
        self.init(ray, start_time, max_time)

    def init(self, ray, start_time, max_time):
        assert start_time <= max_time
        self.t0 = start_time
        self.t1 = max_time
        pos = ray(start_time)
        dir = ray.dir()
        inv = ray.invDir()
        # Align voxel to grid boundary
        self.voxel = [int(math.floor(p)) & ~(self.dim - 1) for p in pos]
        self.delta = [0.0, 0.0, 0.0]
        self.next = [0.0, 0.0, 0.0]
        self.step = [0, 0, 0]
        for axis in range(3):
            if is_zero(dir[axis]):
                self.step[axis] = 0
                self.next[axis] = float('inf')
                self.delta[axis] = float('inf')
            elif inv[axis] > 0:
                self.step[axis] = self.dim
                self.next[axis] = self.t0 + (self.voxel[axis] + self.dim - pos[axis]) * inv[axis]
                self.delta[axis] = self.step[axis] * inv[axis]
            else:
                self.step[axis] = -self.dim
                self.next[axis] = self.t0 + (self.voxel[axis] - pos[axis]) * inv[axis]
                self.delta[axis] = self.step[axis] * inv[axis]

    def step(self):
        step_axis = min_index(self.next)
        self.t0 = self.next[step_axis]
        self.next[step_axis] += self.delta[step_axis]
        self.voxel[step_axis] += self.step[step_axis]
        return self.t0 <= self.t1

    def time(self):
        return self.t0

    def voxel(self):
        return self.voxel

    def next(self):
        # Returns the time of the second hit (next boundary intersection)
        # Incurs computational overhead compared to time() and voxel() [S1, S3]
        return min(self.next)
```

### Parameters
*   `Log2Dim`: Integer template parameter. Default is 0 [S2, unofficial][S6].
*   `Dim`: Fixed step-size dimension, calculated as $2^{\text{Log2Dim}}$ [S1][S5, unofficial]. Default is 1 [S1].

### Edge Cases and Pitfalls
*   **Zero Direction:** If a ray direction component is zero (including $\pm 0$), the corresponding axis step is disabled by setting `mNext` and `mDelta` to `std::numeric_limits<RealT>::max()` to prevent infinite loops or incorrect steps [S2, S5, unofficial].
*   **Computational Overhead:** The `next()` method incurs a small computational overhead, whereas `time()` and `voxel()` incur no computational overhead [S1, S3].
*   **Precision:** The algorithm uses floating-point arithmetic for time calculations [S2, S5, unofficial].
*   **Boundary Conditions:** The initial voxel is aligned to the grid boundary using bitwise AND with `~(DIM-1)` [S2, S5, unofficial]. This assumes integer coordinates for `Coord` and power-of-two dimensions [G].

## Subtopics

### [Incremental Ray Traversal through OpenVDB Frustum Grids](3039-incremental-ray-traversal-through-openvdb-frustum-grids.md)
A method for traversing OpenVDB frustum grids by generalizing the 3D DDA algorithm to account for the non-linearity of the perspective transform, allowing efficient ray traversal for any ray (not just camera rays) without auxiliary grids.

## Operating characteristics
*   **Time Complexity:** Linear in the number of voxels/nodes intersected by the ray [S2, S5, unofficial].
*   **Memory Complexity:** Constant space $O(1)$ for the DDA state, independent of the grid size [S2, S5, unofficial].
*   **Performance:** The `time()` and `voxel()` methods are efficient with no computational overhead [S1, S3]. The `next()` method has a small overhead [S1, S3].
*   **Limitations:** Does not skip empty space, making it less efficient for sparse grids compared to HDDA [S1].

## Background
OpenVDB DDA is part of the OpenVDB library, an open-source C++ library for the management and manipulation of sparse volumetric data [S2, S5, unofficial]. The DDA class is specialized for traversing OpenVDB grids, leveraging the grid's hierarchical structure and fixed node sizes [S2, S5, unofficial]. It is conceptually similar to Bresenham's line algorithm [S2, S5, unofficial][S6]. The method is used in various applications within the OpenVDB ecosystem, including volume rendering and level set operations [S2, S4, unofficial].

## Sources
[S1] OpenVDB: DDA< RayT, CoordT, Dim > Class Template Reference, https://openvdb.org/documentation/doxygen/classnanovdb_1_1math_1_1DDA.html, fetched 2026-10-24
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S3] OpenVDB: DDA< RayT, Log2Dim > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classopenvdb_1_1v9__0_1_1math_1_1DDA.html, fetched 2026-10-24
[S4] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S5] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
[S6] OpenVDB: DDA< RayT, Log2Dim > Class Template Reference, https://www.openvdb.org/documentation/doxygen/classopenvdb_1_1v13__0_1_1math_1_1DDA.html, fetched 2026-10-26
