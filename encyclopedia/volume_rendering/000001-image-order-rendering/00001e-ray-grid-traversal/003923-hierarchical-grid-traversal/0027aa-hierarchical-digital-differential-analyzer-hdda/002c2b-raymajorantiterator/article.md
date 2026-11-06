# RayMajorantIterator

Also called: HDDA-based ray traversal for volume majorant

## Overview
The `RayMajorantIterator` is an iterator that traverses a sparse volume grid using a Hybrid Digital Differential Analyzer (HDDA) to step through voxels or leaves, computing the maximum density (majorant) for each segment where the majorant remains constant [S1, unofficial]. It solves the problem of efficiently determining integration segments for volume path tracing by avoiding per-voxel sampling in empty or uniform regions [S1, unofficial]. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by bundling consecutive steps with identical majorants into single segments, reducing overhead compared to standard voxel DDA which cannot skip inactive nodes or handle variable leaf sizes efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A ray (`RayT`), a grid accessor (`const GridT::AccessorType&`), and an optional debug buffer (`std::vector<DDAStep>*`) [S1, unofficial].
- **Output:** A sequence of `Segment` structs, each containing start time `t0`, end time `t1`, and the constant majorant density `d_maj` for that segment [S1, unofficial].

**Data Structures:**
The iterator maintains the following state fields [S1, unofficial]:
- `float m_scale`: Inverse of the ray direction length in index space, calculated as `1 / ray.dir().length()` [S1, unofficial].
- `RayT m_ray`: The ray object (typically `nanovdb::math::Ray<float>`) [S1, unofficial].
- `float m_majorant`: The current segment's majorant density, initialized to `std::numeric_limits<float>::signaling_NaN()` [S1, unofficial].
- `const GridT::AccessorType& m_acc`: Reference to the grid accessor for probing values [S1, unofficial].
- `nanovdb::math::HDDA<RayT> m_dda`: The HDDA instance managing the traversal step [S1, unofficial].
- `std::vector<DDAStep>* m_step_record_dst`: Pointer to a debug buffer for recording step details [S1, unofficial].

The `Segment` struct returned by the iterator contains [S1, unofficial]:
- `float t0`: Start time in voxel units [S1, unofficial].
- `float t1`: End time in voxel units [S1, unofficial].
- `float d_maj`: Majorant density value for the segment [S1, unofficial].

The `DDAStep` struct (used for debugging) contains [S1, unofficial]:
- `int dim`: HDDA dimension stepped [S1, unofficial].
- `float majorant`: Current majorant [S1, unofficial].
- `float t`: Current time [S1, unofficial].
- `Eigen::Vector3i voxel`: Current voxel coordinates [S1, unofficial].
- `Eigen::Vector3f hit`: World position [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   - Calculate `m_scale = 1.0f / m_ray.dir().length()` [S1, unofficial].
   - Initialize `m_majorant` to `std::numeric_limits<float>::signaling_NaN()` [S1, unofficial].
   - Initialize `m_dda` with the ray and an initial dimension [S1, unofficial]. The initial dimension is determined by `get_hdda_dim`, which returns `std::max<uint32_t>(8, acc.getDim(ijk, ray))` [S1, unofficial]. This ensures a minimum step dimension of 8 to avoid skipping active leaves [S1, unofficial].

2. **Iteration (`next()` method):**
   - **Termination Check:** If `m_dda.time() >= m_dda.maxTime()`, the ray has left the bounding box. Return `std::nullopt` [S1, unofficial].
   - **Start Time:** Set `ans.t0 = m_dda.time()` [S1, unofficial].
   - **Majorant Update:** If `m_majorant` is `NaN`, call `update_current_majorant()` and `record_step()` [S1, unofficial].
   - **Dimension Update:** Calculate the new HDDA dimension for the next step. The coordinate used is `m_ray(m_dda.time() + 1.0001f).floor()` [S1, unofficial]. The offset `1.0001f` ensures the coordinate is slightly ahead of the current step to get the correct dimension for the next step, preventing issues with pessimistic HDDA stepping in empty space [S1, unofficial]. Update `m_dda` with `m_dda.update(m_ray, new_dim)` [S1, unofficial].
   - **Loop for Constant Majorant:**
     - Call `update_current_majorant()` and `record_step()` [S1, unofficial].
     - If the new `m_majorant` equals `ans.d_maj` (the majorant of the current segment), continue the loop to bundle this step into the current segment [S1, unofficial].
     - If `m_majorant` differs, exit the loop [S1, unofficial].
   - **End Time:** Set `ans.t1 = m_dda.time()` [S1, unofficial].
   - **Return:** Return the `Segment` struct with `t0`, `t1`, and `d_maj` [S1, unofficial].

3. **Majorant Calculation (`update_current_majorant()`):**
   - Determine the current voxel index `ijk` from the HDDA state [G].
   - If the current node is not a leaf, attempt to probe the value using `m_acc.probeValue(ijk, value)` [S1, unofficial].
   - If `probeValue` returns true, set `m_majorant = value` and return [S1, unofficial].
   - Otherwise, set `m_majorant = 0.0f` (empty space) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import Optional, Tuple, List

class RayMajorantIterator:
    def __init__(self, ray, accessor, debug_buffer: Optional[List] = None):
        self.m_ray = ray
        self.m_acc = accessor
        self.m_step_record_dst = debug_buffer
        self.m_scale = 1.0 / ray.dir().length()
        self.m_majorant = float('nan')
        # Initial dimension calculation ensures minimum step of 8
        ijk_initial = ray.time(0).floor().astype(int)
        init_dim = max(8, accessor.getDim(ijk_initial, ray))
        self.m_dda = HDDA(ray, init_dim)

    def _get_hdda_dim(self, ijk, ray, accessor):
        return max(8, accessor.getDim(ijk, ray))

    def _update_current_majorant(self):
        ijk = self.m_dda.get_voxel_index()
        # probeValue returns (success, value)
        success, value = self.m_acc.probeValue(ijk)
        if success:
            self.m_majorant = value
        else:
            self.m_majorant = 0.0

    def _record_step(self):
        if self.m_step_record_dst is not None:
            step = DDAStep(
                dim=self.m_dda.get_dim(),
                majorant=self.m_majorant,
                t=self.m_dda.time(),
                voxel=self.m_dda.get_voxel_index(),
                hit=self.m_ray.time(self.m_dda.time())
            )
            self.m_step_record_dst.append(step)

    def next(self) -> Optional[Tuple[float, float, float]]:
        if self.m_dda.time() >= self.m_dda.max_time():
            return None

        ans_t0 = self.m_dda.time()
        
        if math.isnan(self.m_majorant):
            self._update_current_majorant()
            self._record_step()

        ans_d_maj = self.m_majorant
        
        while True:
            next_time_approx = self.m_dda.time() + 1.0001
            ijk_next = self.m_ray.time(next_time_approx).floor().astype(int)
            new_dim = self._get_hdda_dim(ijk_next, self.m_ray, self.m_acc)
            
            self.m_dda.update(self.m_ray, new_dim)
            
            self._update_current_majorant()
            self._record_step()
            
            if self.m_majorant != ans_d_maj:
                break
            
            if self.m_dda.time() >= self.m_dda.max_time():
                break

        ans_t1 = self.m_dda.time()
        
        return (ans_t0, ans_t1, ans_d_maj)

class HDDA:
    """Minimal stub matching the interface described in the parent article."""
    def __init__(self, ray, dim):
        self.ray = ray
        self.dim = dim
        self.current_time = 0.0
        # Bounding box max time not given in sources [G]
        self.max_time = 1000.0  
        
    def time(self):
        return self.current_time
        
    def max_time(self):
        return self.max_time
        
    def update(self, ray, dim):
        self.current_time += 1.0 / ray.dir().length()
        self.dim = dim
        
    def get_voxel_index(self):
        pos = self.ray.time(self.current_time)
        return pos.floor().astype(int)
        
    def get_dim(self):
        return self.dim

class DDAStep:
    def __init__(self, dim, majorant, t, voxel, hit):
        self.dim = dim
        self.majorant = majorant
        self.t = t
        self.voxel = voxel
        self.hit = hit
```

**Parameters and Typical Values:**
- **Minimum HDDA Dimension:** 8. This is hardcoded to avoid skipping active leaves in sparse grids [S1, unofficial].
- **Time Offset for Dimension Update:** `1.0001f`. This small epsilon ensures the coordinate used for dimension lookup is slightly ahead of the current step, handling pessimistic HDDA stepping in empty space [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Pessimistic HDDA Stepping:** HDDA stepping can be pessimistic in empty space, potentially stepping more frequently than necessary. The iterator mitigates this by bundling consecutive segments with the same majorant into one, reducing overhead [S1, unofficial].
- **Boundary Conditions:** The termination check `m_dda.time() >= m_dda.maxTime()` handles the case where the ray leaves the bounding box [S1, unofficial].
- **Precision:** The use of `1.0001f` offset is critical for correct dimension updates. Using `1.0f` might result in the same voxel being queried, leading to incorrect dimension updates or infinite loops in degenerate cases [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The iterator steps through the grid using HDDA, which allows skipping inactive nodes. The complexity is proportional to the number of distinct majorant segments along the ray, which is typically much less than the number of voxels intersected, especially in sparse or uniform regions [S1, unofficial].
- **Memory Complexity:** The iterator maintains a small constant amount of state per ray (HDDA state, current majorant, etc.). The debug buffer, if used, scales with the number of steps [S1, unofficial].
- **Strengths:** Efficiently handles sparse voxel grids by skipping empty space and variable leaf sizes. Bundling segments with the same majorant reduces the number of integration steps required for volume path tracing [S1, unofficial].
- **Limitations:** HDDA stepping can be pessimistic in empty space, although this is mitigated by segment bundling. The method relies on the underlying HDDA implementation's correctness and efficiency [S1, unofficial].
- **When to Use:** Ideal for volume path tracing in sparse voxel grids where empty space is common and majorant values are constant over large segments [S1, unofficial].
- **When Not to Use:** If the volume is dense and majorant values change frequently per voxel, the overhead of HDDA and segment bundling might not provide significant benefits over standard voxel DDA [S1, unofficial].

## Background
The `RayMajorantIterator` is part of the volume path tracer developed by Shibodd [S1, unofficial]. It leverages the NanoVDB library's HDDA implementation for efficient hierarchical traversal. The method was designed to optimize volume rendering by reducing the number of samples required for accurate integration, particularly in large, sparse scenes [S1, unofficial].

## Sources
[S1] Shibodd/volume_path_tracer, https://github.com/Shibodd/volume_path_tracer, fetched 2026-10-26 (unofficial)
