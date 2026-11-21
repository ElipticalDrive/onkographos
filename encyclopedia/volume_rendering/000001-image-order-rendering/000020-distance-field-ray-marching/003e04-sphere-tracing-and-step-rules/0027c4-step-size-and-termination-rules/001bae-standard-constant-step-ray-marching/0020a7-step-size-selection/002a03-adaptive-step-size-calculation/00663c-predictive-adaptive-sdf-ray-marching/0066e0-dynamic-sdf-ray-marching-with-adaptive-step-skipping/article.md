# Dynamic SDF Ray Marching with Adaptive Step Skipping

Also called: Ray Advancing Kernel, SDF Marching

## Overview
A GPU-based ray marching technique for Signed Distance Fields (SDFs) that advances ray origins by predicted distances, skips empty space using an octree, handles floor reflections, and tracks shadow visibility metrics for soft shadows [S1, unofficial]. Unlike standard SDF marching which uses `fminf(distance, max_step)`, this method uses a predicted distance from previous steps (`prev_distances`) to determine step size and visibility, allowing for adaptive skipping of empty space via the octree [S1, unofficial].

## How it works
**Inputs and Outputs:**
The kernel operates on device buffers.
*   `positions`: `Vector3f*` array of current ray origins.
*   `distances`: `float*` array of raw distance field values retrieved at the current position.
*   `payloads`: `SdfPayload*` array containing ray state (`alive`, `dir`, `n_steps`, `idx`).
*   `prev_distances`: `float*` array storing the distance estimate from the previous step for each ray.
*   `total_distances`: `float*` array accumulating total distance traveled (used for visibility metrics).
*   `min_visibility`: `float*` array tracking minimum visibility values.
*   `octree_nodes`: Optional reference to a `TriangleOctree` structure for spatial acceleration.
*   `floor_y`: Float constant defining the floor plane height.
*   `distance_scale`: Float multiplier for distance scaling.
*   `zero_offset`: Float offset subtracted from raw distance.
*   `maximum_distance`: Float threshold for ray termination.
*   `k`: Constant factor for visibility calculation (value not specified in sources).

**Data Structures:**
*   `SdfPayload`:
    *   `alive`: `bool` (or equivalent flag) indicating if the ray is active.
    *   `dir`: `Vector3f` ray direction.
    *   `n_steps`: `uint32_t` or `int` step counter.
    *   `idx`: `uint32_t` thread/element index.

**Algorithm Steps:**
1.  Calculate thread index `i`. If `i >= n_elements`, terminate.
2.  Check `payloads[i].alive`. If false, return.
3.  Retrieve raw distance `d_raw = distances[i]`. Compute adjusted distance: `distance = (d_raw - zero_offset) * distance_scale`.
4.  Advance position: `pos = positions[i] + distance * payloads[i].dir`.
5.  **Octree Skip:** If `octree_nodes` exists and `pos` is not contained in the octree (`!TriangleOctree::contains(octree_nodes, pos)`):
    *   Calculate intersection distance `octree_distance = TriangleOctree::ray_intersect(octree_nodes, pos, payloads[i].dir)`.
    *   Add epsilon to prevent self-intersection: `octree_distance += 1e-6f`.
    *   Update distance and position: `distance += octree_distance`, `pos += octree_distance * payloads[i].dir`.
6.  **Floor Reflection:** If `pos.y() < floor_y` and `payloads[i].dir.y() < 0.0f`:
    *   Calculate floor intersection: `floor_dist = -(pos.y() - floor_y) / payloads[i].dir.y()`.
    *   Update distance and position: `distance += floor_dist`, `pos += floor_dist * payloads[i].dir`.
    *   Terminate ray: `payloads[i].alive = false`.
7.  Update position buffer: `positions[i] = pos`.
8.  **Visibility Metric Update:** If `total_distances` is not null and `distance > 0.0f`:
    *   Calculate intermediate values:
        $$ y = \frac{distance^2}{2 \cdot prev\_distances[i]} $$
        $$ d = \sqrt{distance^2 - y^2} $$
    *   Calculate total distance so far: `total_distance = total_distances[i]`.
    *   Update minimum visibility:
        $$ min\_visibility[i] = \min(min\_visibility[i], \frac{k \cdot d}{\max(0.0, total\_distance - y)}) $$
    *   Update history: `prev_distances[i] = distance`, `total_distances[i] += distance`.
9.  **Survival Check:** Determine if ray should continue:
    *   Condition: `stay_alive = (distance > maximum_distance) && (fabsf(distance / 2) > 3 * maximum_distance)`.
    *   If not `stay_alive`, set `payloads[i].alive = false` and return.
10. **AABB Containment:** If `!aabb.contains(pos)`, set `payloads[i].alive = false` and return.
11. Increment step counter: `payloads[i].n_steps++`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def advance_pos_kernel_sdf(positions, distances, payloads, prev_distances, 
                           total_distances, min_visibility, octree_nodes, 
                           floor_y, distance_scale, zero_offset, maximum_distance, 
                           k, aabb, n_elements):
    i = get_thread_index()
    if i >= n_elements:
        return

    payload = payloads[i]
    if not payload.alive:
        return

    # Step 3: Retrieve and scale distance
    raw_dist = distances[i]
    distance = (raw_dist - zero_offset) * distance_scale
    
    # Step 4: Advance position
    pos = positions[i] + distance * payload.dir
    
    # Step 5: Octree skip
    if octree_nodes is not None and not triangle_octree_contains(octree_nodes, pos):
        octree_distance = triangle_octree_ray_intersect(octree_nodes, pos, payload.dir)
        octree_distance += 1e-6f  # Epsilon to prevent self-intersection
        distance += octree_distance
        pos += octree_distance * payload.dir
        
    # Step 6: Floor reflection
    if pos.y < floor_y and payload.dir.y < 0.0f:
        floor_dist = -(pos.y - floor_y) / payload.dir.y
        distance += floor_dist
        pos += floor_dist * payload.dir
        payload.alive = False
        
    # Step 7: Update position
    positions[i] = pos
    
    # Step 8: Visibility metrics
    if total_distances is not None and distance > 0.0f:
        prev_d = prev_distances[i]
        # Avoid division by zero if prev_d is 0, though logic implies prev_d > 0 for valid history
        if prev_d > 0.0f:
            y = (distance * distance) / (2.0f * prev_d)
            d_val = sqrt(max(0.0f, distance * distance - y * y))
            total_dist = total_distances[i]
            denom = max(0.0f, total_dist - y)
            if denom > 0.0f:
                visibility_val = (k * d_val) / denom
                min_visibility[i] = min(min_visibility[i], visibility_val)
        prev_distances[i] = distance
        total_distances[i] += distance
        
    # Step 9: Survival check
    # Note: The source condition "distance > maximum_distance && fabsf(distance / 2) > 3*maximum_distance"
    # implies the ray must be far from the surface AND the step size must be large relative to the threshold.
    # If distance <= maximum_distance, the ray is killed (stay_alive = False).
    stay_alive = (distance > maximum_distance) and (abs(distance / 2.0f) > 3.0f * maximum_distance)
    
    if not stay_alive:
        payload.alive = False
        return
        
    # Step 10: AABB containment
    if not aabb_contains(aabb, pos):
        payload.alive = False
        return
        
    # Step 11: Increment steps
    payload.n_steps += 1
```

**Parameters:**
*   `MARCH_ITER`: Constant set to 10000 (likely max iterations in testbed context) [S1, unofficial].
*   `zero_offset`: Float offset subtracted from raw distance [S1, unofficial].
*   `distance_scale`: Float multiplier applied to distance [S1, unofficial].
*   `maximum_distance`: Float threshold for ray termination [S1, unofficial].
*   `k`: Constant for visibility calculation (value not specified in sources) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-intersection:** When skipping octree nodes, a small epsilon `1e-6f` is added to the intersection distance to handle precision issues [S1, unofficial].
*   **Floor Termination:** If the ray hits the floor (`pos.y() < floor_y` and moving downwards), the ray is marked as dead immediately after calculating the intersection [S1, unofficial].
*   **Division by Zero:** In the visibility formula, `prev_distances[i]` and `total_distance - y` are used as denominators. The source code uses `fmaxf(0.0f, ...)` for the latter, but explicit checks for zero `prev_distances` are recommended to prevent NaN propagation [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and steps, but adaptive skipping via octree reduces the number of SDF evaluations in empty space.
*   **Memory Complexity:** Requires storage for `prev_distances`, `total_distances`, and `min_visibility` buffers in addition to standard ray buffers.
*   **Strengths:** Adaptive step size based on previous estimates allows for larger steps in empty space. Octree integration enables efficient skipping of large empty volumes. Visibility metrics support soft shadow calculation.
*   **Limitations:** Requires maintaining history buffers (`prev_distances`). Floor reflection logic is hardcoded and may not generalize to all scenes. The survival condition `distance > maximum_distance` kills rays that are close to surfaces, which might be undesirable for high-precision surface rendering if `maximum_distance` is set too high [S1, unofficial].
*   **When to use:** Scenes with large empty spaces where octree acceleration is effective, and soft shadow visibility metrics are required.
*   **When not to use:** Scenes without a defined floor plane, or where high-precision surface proximity requires smaller, more frequent steps than the adaptive predictor allows.

## Background
Originating from the Dynamic-NGPs project, this method is part of a pipeline for training and rendering dynamic neural graphics primitives [S1, unofficial]. It is used in the rendering phase (`render.py`) to generate frames from trained models, incorporating spatial acceleration and shadow visibility tracking [S1, unofficial].

## Sources
[S1] jaeyeol816/Dynamic-NGPs, https://github.com/jaeyeol816/Dynamic-NGPs, fetched 2026-10-27 (unofficial)
