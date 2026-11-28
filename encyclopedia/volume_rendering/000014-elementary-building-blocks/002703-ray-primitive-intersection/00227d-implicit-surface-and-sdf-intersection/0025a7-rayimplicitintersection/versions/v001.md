# RayImplicitIntersection

Also called: ZeroCrossing, ZeroCrossingNode, nanovdb::ZeroCrossing

## Overview
Detects if a ray intersects a zero-crossing (surface) at the voxel level of a Signed Distance Field (SDF) grid in an accessor. It performs empty-space ray-marching at all levels of the tree using a Hierarchical Distance Field Acceleration (HDDA) [S1][S3, unofficial]. Unlike [Kalra & Barr Implicit Surface Intersection](1278-kalra-barr-implicit-surface-intersection.md) which may traverse all active nodes, this method uses a leaf-only iterator to skip coarse tiles entirely, optimizing for the specific case of finding the first sign change in the narrow band [S2, unofficial].

## How it works
**Inputs and Outputs:**
- `ray`: The ray to test, containing origin, direction, and time bounds $[t_0, t_1]$.
- `acc`: The grid accessor providing voxel values and hierarchy traversal.
- `ijk`: Output coordinate of the closest voxel after the intersection point.
- `v`: Output grid value at `ijk`.
- `t`: Output time of the intersection along the ray.
- `eps`: Epsilon threshold for zero-crossing detection. If `eps == 0.0f`, a compile-time specialization (`EpsZero`) is used to drop the per-voxel `deltaT < eps` branch [S2, unofficial].
- `starts_inside_surface`: Parameter controlling behavior for rays starting inside the surface (defaults to reporting the exit crossing) [S2, unofficial].

**Data Structures:**
- `HDDAValueIteratorImpl<AccT, ScalarT, bool LeafOnly>`: A template class where `LeafOnly` is a compile-time gate. If `LeafOnly` is true, the iterator skips active coarse tiles. If false, it yields all active values (tiles and leaves) [S2, unofficial].
- `ScalarT`: The data type of the SDF values. Supports fp32 and fp16 (`c10::Half`). For fp16 inputs, arithmetic is performed in fp32 (`at::opmath_type<ScalarT>`) and cast back only at store [S2, unofficial].

**Steps:**
1. **Clip Ray:** Clip the ray to the bounding box of the grid root. If the ray does not intersect the bounding box or $t_1$ exceeds a large threshold (e.g., $10^{20}$), return false [S3, unofficial].
   ```python
   if not ray.clip(acc.root().bbox()) or ray.t1() > 1e20:
       return False
   ```
2. **Seed Sign Reference:** Find the first valid (non-NaN) voxel along the ray. This voxel's sign becomes the reference sign. This makes the method SDF-sign-agnostic and ensures rays starting inside the surface report the exit crossing [S2, unofficial].
   ```python
   ijk = RoundDown<Coord>(ray.start())
   v0 = acc.getValue(ijk)
   if is_nan(v0):
       # Advance to first valid voxel
       while is_nan(acc.getValue(ijk)):
           ijk = next_voxel(ijk, ray)
           if ijk out of bounds: return False
       v0 = acc.getValue(ijk)
   reference_sign = sign(v0)
   ```
3. **Initialize HDDA:** Create an HDDA instance with the ray and the current dimension. Initialize `lastT1` to track continuity for interpolation [S3, unofficial].
   ```python
   Delta = 1.0001f
   hdda = HDDA<RayT, Coord>(ray, acc.getDim(ijk, ray))
   lastT1 = -inf
   ```
4. **Traverse and Sample:** Use the leaf-only iterator (`HDDALeafVoxelIterator`) to step through active leaf voxels. Skip active coarse tiles in a single coarse HDDA step, as tiles cannot contain the leaf-level zero crossing [S2, unofficial].
   ```python
   while hdda.step():
       ijk = RoundDown<Coord>(ray(hdda.time() + Delta))
       hdda.update(ray, acc.getDim(ijk, ray))
       
       # Skip tiles or inactive voxels
       if hdda.dim() > 1 or not acc.isActive(ijk):
           continue
           
       v = acc.getValue(hdda.voxel())
       
       # Detect sign flip
       voxel_sign = sign(v)
       if voxel_sign != reference_sign:
           # Sign flip detected
           t0 = hdda.time()
           
           # Interpolate hit time if contiguous
           if t0 == lastT1:
               # Sub-voxel interpolate the hit time
               t = interpolate_hit_time(ray, lastT1, prev_v, t0, v)
           else:
               # Fall back to bracket-entry time (gap between disjoint regions or NaN)
               t = t0
               
           ijk = hdda.voxel()
           return True
           
       lastT1 = hdda.time()
   return False
   ```
5. **Interpolation:** If a sign flip is detected, sub-voxel interpolate the hit time. Only interpolate if the previous voxel was contiguous along the ray (`t0 == lastT1`). If not contiguous (e.g., gap between disjoint regions or NaN tile values), fall back to the bracket-entry time [S2, unofficial].

**Parameters:**
- `Delta`: Static float constant set to `1.0001f`, used to step slightly past the current hit time to avoid re-hitting the same voxel due to precision [S3, unofficial].
- `eps`: Epsilon threshold. If `eps == 0.0f`, the `deltaT < eps` branch is dropped [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Inside-Surface Rays:** Rays starting inside the surface must report the exit crossing. The convention-agnostic seed (first valid voxel) ensures this, avoiding spurious results from baked-in 'positive outside, negative inside' sentinels [S2, unofficial].
- **Disjoint Regions/NaN Gaps:** Interpolation across empty space (gaps between disjoint SDF regions or runs of NaN tile values) is prevented by checking `t0 == lastT1` for continuity [S2, unofficial].
- **Tile Skipping:** The leaf-iterator switch is correct-by-construction for active tiles. The old iterator would have walked those into the per-voxel body (mis-indexing or reading `gridScalars[-1]`). The leaf iterator skips them [S2, unofficial].
- **FP16 Dispatch:** `c10::Half` is now dispatched. Code that caught `NotImplementedError` and fell back to fp32 conversion at the Python boundary should drop that branch [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the depth of the tree and the density of the narrow band. HDDA allows skipping empty space efficiently.
- **Memory Complexity:** Minimal additional memory per ray; uses cache hints to optimize L1/L2 usage [S2, unofficial].
- **Performance:** On RTX PRO 6000 Blackwell, 1M rays, dragon dataset (267 MB SDF), fp32, eps=0: Speedup 1.95x (1.263ms -> 0.646ms). Instructions reduced by 50.6% (1.29e+08 -> 6.35e+07). Registers reduced from 80 to 78 [S2, unofficial]. On emu dataset (1.0 GB SDF): Speedup 1.73x. On crawler dataset (1.6 GB SDF): Speedup 1.67x [S2, unofficial].
- **Strengths:** High performance due to leaf-only traversal and cache hints. Correctness for inside-surface rays and disjoint regions.
- **Limitations:** Requires HDDA infrastructure. FP16 support requires careful arithmetic handling [S2, unofficial].

## Background
Originated in the OpenVDB/fvdb-core project, specifically in the `ray_implicit_intersection` kernel optimization [S2, unofficial]. It is based on the `nanovdb::ZeroCrossing` method [S1]. The method was improved to fix correctness issues and enhance performance by reorganizing HDDA iterators and adding cache hints [S2, unofficial].

## Sources
[S1] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S2] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
[S3] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
