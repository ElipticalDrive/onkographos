# RayImplicitIntersection

Also called: ZeroCrossing, ZeroCrossingNode, nanovdb::ZeroCrossing

## Overview
Detects if a ray intersects a zero-crossing at the voxel level of a grid in the accessor [S1]. It performs empty-space ray-marching at all levels of the tree using an HDDA (Hierarchical Distance Field Acceleration) [S1]. The method is optimized for performance, precision, and correctness, and surfaces a shared cache-hint helper for other operations [S2, unofficial]. It is SDF-sign-agnostic and reports the EXIT crossing for rays that start inside the surface, matching nanovdb::ZeroCrossing behavior [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   `ray`: Ray object with origin $\mathbf{o}$, direction $\mathbf{d}$, and methods `t0()`, `t1()`, `invDir()`, `operator()(time)`, and `clip(bbox)` [S4, unofficial].
*   `acc`: Accessor providing `getValue(ijk)`, `isActive(ijk)`, `getDim(ijk, ray)`, and `root().bbox()` [S1].
*   `ijk`: Output coordinate of the closest voxel after the intersection point [S1].
*   `v`: Output grid value at the intersection coordinate `ijk` [S1].
*   `t`: Output time of the intersection along the ray [S1].
*   `eps`: Epsilon threshold for zero-crossing detection. If `eps == 0.0f`, a compile-time specialization (`EpsZero`) is used to drop the per-voxel `deltaT < eps` branch [S2, unofficial].
*   `starts_inside_surface`: Parameter controlling behavior for rays starting inside the surface [S2, unofficial].

**Data Structures:**
*   `HDDAValueIteratorImpl<AccT, ScalarT, bool LeafOnly>`: A template class where `LeafOnly` is a compile-time gate. If `LeafOnly` is true, the iterator skips active coarse tiles. If false, it yields all active values (tiles and leaves) [S2, unofficial].
*   `HDDALeafVoxelIterator`: Alias for `HDDAValueIteratorImpl` with `LeafOnly=true`. Used to skip active coarse tiles entirely in a single coarse HDDA step, as tiles cannot contain the leaf-level zero crossing [S2, unofficial].
*   `HDDA<RayT, CoordT>`: Digital Differential Analyzer specialized for grids. Conceptually similar to Bresenham's line algorithm applied to a 3D Ray intersecting nodes or voxels. `Log2Dim = 0` corresponds to a voxel [S4, unofficial].

**Steps:**
1.  **Clip Ray:** Clip ray to bounding box. If `!ray.clip(acc.root().bbox())` or `ray.t1() > 1e20`, return false [S3, unofficial].
2.  **Seed Sign Reference:** Find the first valid (non-NaN) voxel along each ray. This voxel's sign becomes the reference sign [S2, unofficial]. Set `ijk = RoundDown<Coord>(ray.start())` [S3, unofficial]. Get `v0 = acc.getValue(ijk)` [S3, unofficial].
3.  **Initialize HDDA:** Create HDDA instance with ray and current dimension: `HDDA<RayT, Coord> hdda(ray, acc.getDim(ijk, ray))` [S3, unofficial].
4.  **Outer Loop:** While `hdda.step()`:
    *   Update `ijk = RoundDown<Coord>(ray(hdda.time() + Delta))`, where `Delta` is a static float constant set to `1.0001f` to step slightly past the current hit time to avoid re-hitting the same voxel due to precision [S3, unofficial].
    *   Update `hdda.update(ray, acc.getDim(ijk, ray))` [S3, unofficial].
    *   If `hdda.dim() > 1` (tile) or `!acc.isActive(ijk)` (inactive voxel), continue to next step [S3, unofficial].
5.  **Inner Narrow-Band Loop:** While `hdda.step() && acc.isActive(hdda.voxel())`:
    *   Get `v = acc.getValue(hdda.voxel())` [S3, unofficial].
    *   Detect sign flip: Check if `v * v0 < 0` [S3, unofficial].
    *   If sign flip detected:
        *   Set `ijk = hdda.voxel()`, `t = hdda.time()`, `v = acc.getValue(ijk)` [S3, unofficial].
        *   **Interpolate Hit Time:** If a sign flip is detected, sub-voxel interpolate the hit time [S2, unofficial]. Only interpolate if the previous voxel was contiguous along the ray (`t0 == lastT1`) [S2, unofficial]. If not contiguous (e.g., gap between disjoint regions or NaN tile values), fall back to the bracket-entry time [S2, unofficial].
        *   Return true [S3, unofficial].
6.  **Return False:** If loop completes without intersection [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_implicit_intersection(ray, acc, eps=0.0):
    # 1. Clip ray to bounding box
    if not ray.clip(acc.root().bbox()) or ray.t1() > 1e20:
        return False, None, None, None

    Delta = 1.0001
    ijk = math.floor(ray.start())  # [G] standard grid coordinate rounding
    v0 = acc.get_value(ijk)

    # 2. Seed sign reference: first valid (non-NaN) voxel
    if math.isnan(v0):
        pass  # Sources do not specify fallback for all-NaN rays; assume valid seed exists [G]
    scalar_sign = math.copysign(1.0, v0)
    last_t1 = -1.0  # Continuity tracker

    # 3. Initialize HDDA
    hdda = HDDA(ray, acc.get_dim(ijk, ray))

    # 4. Traverse using leaf-only iterator to skip active coarse tiles
    iterator = HDDALeafVoxelIterator(acc, hdda)

    while iterator.has_next():
        if not hdda.step():
            break

        # Update ijk and HDDA state
        ijk = math.floor(ray(hdda.time() + Delta))
        hdda.update(ray, acc.get_dim(ijk, ray))

        # Skip tiles and inactive voxels
        if hdda.dim() > 1 or not acc.is_active(ijk):
            continue

        v = acc.get_value(ijk)

        # 5. Detect sign flip
        voxel_sign = math.copysign(1.0, v)
        if scalar_sign != voxel_sign:
            t = hdda.time()

            # 6. Interpolate hit time if contiguous
            if t == last_t1:
                t = interpolate_hit_time(ray, ijk, v0, v)  # [G] standard linear interpolation
            else:
                pass  # Fall back to bracket-entry time

            return True, ijk, v, t

        last_t1 = hdda.time()

    return False, None, None, None

# [G] Standard linear interpolation for zero-crossing hit time
def interpolate_hit_time(ray, ijk, v0, v):
    t0 = ray(hdda.time())  # Approximate entry time
    t1 = ray(hdda.time() + 1.0)  # Approximate exit time
    return t0 + (t1 - t0) * abs(v0) / (abs(v0) + abs(v))
```

## Subtopics

### Iso-Crossing Detection
Similar to zero-crossing but detects crossings of a specified iso-value [S4, unofficial]. It uses an HDDA for empty-space ray-marching [S4, unofficial]. If an intersection with the iso-surface is detected, it updates `ijk`, `v`, and `t` [S1]. The logic is identical to zero-crossing but compares `acc.getValue(ijk) - iso` against zero [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. HDDA allows skipping empty space efficiently [S1][S4, unofficial].
*   **Memory Complexity:** Not given in the sources. Low, primarily storing ray state and HDDA state per thread [G].
*   **Performance:** On RTX PRO 6000 Blackwell, 1M rays, dragon dataset (267 MB SDF), fp32, eps=0: Speedup 1.95x (1.263ms -> 0.646ms). Instructions reduced by 50.6% (1.29e+08 -> 6.35e+07). Registers reduced from 80 to 78 [S2, unofficial]. On emu dataset (1.0 GB SDF): Speedup 1.73x (1.832ms -> 1.059ms). On crawler dataset (1.6 GB SDF): Speedup 1.67x (3.586ms -> 2.143ms) [S2, unofficial].
*   **Strengths:** High performance due to tile-skipping and cache-hint optimizations. Correctness for rays starting inside the surface. Support for fp16 inputs [S2, unofficial].
*   **Limitations:** L1 hit rate dips (94.5% → 87.8% on dragon) due to streaming-store path bypassing L1 and tile-skipping removing trivially cache-friendly reads [S2, unofficial]. Instruction count rises on uniform_ray_samples count pass (+8.2% dragon) even as cycles fall [S2, unofficial].
*   **When to Use:** When high-performance SDF ray intersection is needed, especially for large grids with sparse active regions. When rays may start inside the surface. When fp16 support is required [S2, unofficial].
*   **When Not to Use:** When exact iso-value crossing is needed (use Iso-Crossing Detection). When grid contains active coarse tiles that must be processed per-voxel (use HDDAActiveValueIterator) [S2, unofficial].

## Background
Originated in OpenVDB/nanovdb as `nanovdb::ZeroCrossing` and `nanovdb::ZeroCrossingNode` [S1]. Optimized and hardened in fvdb-core as `ray_implicit_intersection` [S2, unofficial]. Used in volume rendering and implicit surface raycasting [S3, unofficial].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S2] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
[S3] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S4] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
