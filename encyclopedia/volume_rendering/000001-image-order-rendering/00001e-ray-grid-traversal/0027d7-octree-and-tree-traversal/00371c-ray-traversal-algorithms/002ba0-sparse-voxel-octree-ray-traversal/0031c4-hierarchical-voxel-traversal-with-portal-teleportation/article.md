# Hierarchical Voxel Traversal with Portal Teleportation

Also called: Ray Marching through Octree, Voxel Ray Tracing

## Overview
A ray traversal algorithm that navigates a hierarchical voxel structure (octree) to find surface intersections, supporting teleportation through portals defined in the voxel data [S1, unofficial]. It extends standard sparse voxel traversal by integrating portal logic directly into the step loop, allowing rays to jump between disconnected spatial regions [S1, unofficial]. Unlike standard traversal which terminates at the first hit or empty space, this method maintains a transformation matrix to track the ray's orientation relative to the portal pair, enabling seamless visual continuity across non-adjacent volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin `r.pos` and direction `r.dir` in world space; voxel grid parameters (`voxel_uniforms.texture_size`, `VOXELS_PER_METER`); portal definitions (position, normal, transformation matrix); physics distance limit; material flags.
*   **Output:** `HitInfo` struct containing hit status, voxel data, material index, world-space position, reprojection position, surface normal, portal transformation matrix, and step count. Returns a miss if the ray exits bounds or exceeds the physics distance.

**Data Structures:**
*   **Voxel Data:** A 32-bit integer where the lower 8 bits (`data & 0xFFu`) store the material index, and bits 8-31 store flags.
*   **Portal Structure:** Contains `position` (world space), `normal` (world space), and `transformation` (4x4 matrix mapping portal A space to portal B space).
*   **Traversal State:** `tcpotr` (current position in voxel space), `steps` (counter), `normal` (surface normal), `portal_mat` (accumulated portal transformation).

**Parameters:**
*   `VOXELS_PER_METER`: World-to-voxel scale factor.
*   `physics_distance`: Maximum traversal distance in world units.
*   `flags`: Bitmask for material filtering.
*   `should_portal_skip`: Boolean flag indicating if the current voxel is a portal entry/exit.

**Step-by-Step Method:**

1.  **Scale Conversion:** Compute world-to-render (`wtr`) and render-to-world (`rtw`) scales:
    $$ wtr = \frac{VOXELS\_PER\_METER \times 2.0}{texture\_size} $$
    $$ rtw = \frac{texture\_size}{VOXELS\_PER\_METER \times 2.0} $$
    [S1, unofficial]

2.  **Ray Initialization:** Convert ray origin to voxel space:
    $$ pos = r.pos \times wtr $$
    Handle zero-direction rays by adding epsilon:
    $$ dir\_mask = (r.dir == vec3(0.0)) $$
    $$ dir = r.dir + dir\_mask \times 0.000001 $$
    [S1, unofficial]

3.  **Bounding Box Check:** If `pos` is out of bounds, compute distance to octree bounding box $[-1, 1]^3$ using `ray_box_dist`. If distance is 0, return miss. Otherwise, advance ray:
    $$ pos = pos + dir \times dist $$
    $$ distance = distance + dist $$
    [S1, unofficial]

4.  **State Setup:** Initialize traversal state:
    $$ r\_sign = sign(dir) $$
    $$ tcpotr = pos $$
    $$ steps = 0 $$
    $$ normal = trunc(pos \times 1.00001) $$
    [S1, unofficial]

5.  **Traversal Loop:** Iterate while `steps < 1000`:
    a.  **Fetch Voxel:** Retrieve voxel data at `tcpotr` using `get_value(tcpotr)`.
    b.  **Hit Check:** If `(voxel.data & 0xFFu) != 0u` AND `!should_portal_skip` AND `((voxel.data >> 8u) & flags) > 0u` (or `flags == 0u`), break loop.
    c.  **Step Size:** Compute voxel size:
        $$ voxel\_size = \frac{2.0}{voxel.grid\_size} $$
        Compute exit times for each axis:
        $$ t\_max = \frac{voxel.pos - pos + r\_sign \times \frac{voxel\_size}{2.0}}{dir} $$
        [S1, unofficial]
    d.  **Normal Determination:** Identify the first axis hit:
        $$ mask = (t\_max.xyz \le min(t\_max.yzx, t\_max.zxy)) $$
        $$ normal = mask \times -r\_sign $$
        [S1, unofficial]
    e.  **Advance Ray:**
        $$ t\_current = min(t\_max.x, min(t\_max.y, t\_max.z)) $$
        $$ tcpotr = pos + dir \times t\_current - normal \times 0.000002 $$
        $$ reprojection\_pos = r.pos + (t\_current + distance) \times r.dir \times rtw $$
        [S1, unofficial]
    f.  **Portal Teleportation:** If `should_portal_skip` is true:
        i.  Compute intersection with portal plane:
            $$ intersection = ray\_plane(Ray(pos \times rtw, dir), portal.position + portal.normal \times 0.00002, portal.normal) $$
        ii. If `intersection.w != 0.0` AND `intersection.w \times wtr < t_current`:
            $$ pos = (portal.transformation \times vec4(intersection.xyz - portal.normal \times 0.00004, 1.0)).xyz \times wtr $$
            $$ dir = (portal.transformation \times vec4(dir, 0.0)).xyz $$
            $$ r\_sign = sign(dir) $$
            $$ tcpotr = pos $$
            $$ portal\_mat = portal.transformation \times portal\_mat $$
        [S1, unofficial]
    g.  **Physics Limit:** If `t_current + distance > physics_distance \times wtr` AND `physics_distance > 0.0`, return miss.
    h.  **Bounds Check:** If `!in_bounds(tcpotr)`, return miss or intersect scene based on `physics_distance`.
    i.  **Increment:** `steps = steps + 1u`.

6.  **Return Hit:** On loop break, return:
    $$ HitInfo(true, voxel.data, voxel\_uniforms.materials[voxel.data \& 0xFFu], tcpotr \times rtw + normal \times 0.0001, reprojection\_pos, normal, portal\_mat, steps) $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_voxel_portal(ray_origin, ray_dir, voxel_uniforms, physics_distance, flags):
    # 1. Scale Conversion
    wtr = (VOXELS_PER_METER * 2.0) / voxel_uniforms.texture_size
    rtw = voxel_uniforms.texture_size / (VOXELS_PER_METER * 2.0)

    # 2. Ray Initialization
    pos = ray_origin * wtr
    dir_mask = vec3(1.0 if ray_dir == vec3(0.0) else 0.0)
    dir = ray_dir + dir_mask * 0.000001

    # 3. Bounding Box Check
    if not in_bounds(pos):
        dist = ray_box_dist(pos, dir)
        if dist == 0:
            return HitInfo(False, 0, None, vec3(0), vec3(0), vec3(0), mat4(1), 0)
        pos = pos + dir * dist
        distance = dist
    else:
        distance = 0.0

    # 4. State Setup
    r_sign = sign(dir)
    tcpotr = pos
    steps = 0
    normal = trunc(pos * 1.00001)
    portal_mat = mat4(1)

    # 5. Traversal Loop
    while steps < 1000:
        voxel = get_value(tcpotr)
        
        # 5b. Hit Check
        material_id = voxel.data & 0xFFu
        voxel_flags = (voxel.data >> 8u)
        is_portal = should_portal_skip(voxel)
        
        if material_id != 0 and not is_portal and ((voxel_flags & flags) > 0 or flags == 0):
            break

        # 5c. Step Size
        voxel_size = 2.0 / voxel.grid_size
        t_max = (voxel.pos - pos + r_sign * (voxel_size / 2.0)) / dir

        # 5d. Normal Determination
        mask = vec3(1.0 if t_max[i] <= min(t_max[(i+1)%3], t_max[(i+2)%3]) else 0.0 for i in 0..3)
        normal = mask * -r_sign

        # 5e. Advance Ray
        t_current = min(t_max.x, min(t_max.y, t_max.z))
        tcpotr = pos + dir * t_current - normal * 0.000002
        reprojection_pos = ray_origin + (t_current + distance) * ray_dir * rtw

        # 5f. Portal Teleportation
        if is_portal:
            portal = get_portal_for_voxel(voxel)
            ray_world = Ray(ray_origin, ray_dir)
            plane_pos = portal.position + portal.normal * 0.00002
            intersection = ray_plane(ray_world, plane_pos, portal.normal)
            
            if intersection.w != 0.0 and intersection.w * wtr < t_current:
                hit_point = intersection.xyz - portal.normal * 0.00004
                pos = (portal.transformation * vec4(hit_point, 1.0)).xyz * wtr
                dir = (portal.transformation * vec4(dir, 0.0)).xyz
                r_sign = sign(dir)
                tcpotr = pos
                portal_mat = portal.transformation * portal_mat

        # 5g. Physics Limit
        if t_current + distance > physics_distance * wtr and physics_distance > 0.0:
            return HitInfo(False, 0, None, vec3(0), vec3(0), vec3(0), mat4(1), steps)

        # 5h. Bounds Check
        if not in_bounds(tcpotr):
            return HitInfo(False, 0, None, vec3(0), vec3(0), vec3(0), mat4(1), steps)

        # 5i. Increment
        steps += 1

    # 6. Return Hit
    return HitInfo(
        True,
        voxel.data,
        voxel_uniforms.materials[material_id],
        tcpotr * rtw + normal * 0.0001,
        reprojection_pos,
        normal,
        portal_mat,
        steps
    )
```

**Edge Cases and Pitfalls:**
*   **Zero Direction:** Rays with zero direction must be perturbed by epsilon to avoid division by zero in step size calculation [S1, unofficial].
*   **Portal Precision:** Portal intersection checks use small offsets (`0.00002`, `0.00004`) to avoid self-intersection or precision errors at the portal plane [S1, unofficial].
*   **Step Limit:** The loop is capped at 1000 steps to prevent infinite loops in complex portal configurations or deep octrees [S1, unofficial].
*   **Portal Pairing:** Portals must be defined in pairs with inverse transformations to ensure correct teleportation direction [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of steps taken, capped at 1000. Each step involves constant-time arithmetic and a voxel fetch.
*   **Memory Complexity:** $O(1)$ auxiliary memory for traversal state. Voxel data is accessed via texture lookup or buffer read.
*   **Strengths:** Supports non-Euclidean spaces via portal teleportation; efficient for sparse data; integrates seamlessly with GPU-based rendering pipelines.
*   **Limitations:** Limited by the 1000-step cap; portal logic adds computational overhead per step; requires pre-computed portal transformations.
*   **When to Use:** Ideal for games or simulations with disconnected areas (e.g., portals, wormholes) where standard ray traversal would fail or require complex scene graph management.
*   **When Not to Use:** For simple convex scenes without portals, standard SVO traversal is simpler and faster.

## Background
This method is implemented in the `bevy-voxel-engine` project by ria8651 [S1, unofficial]. It is designed for real-time voxel rendering in the Bevy game engine, supporting features like character physics and portal-based level design. The approach combines hierarchical voxel traversal with explicit portal handling, allowing for seamless navigation between non-adjacent voxel volumes.

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
