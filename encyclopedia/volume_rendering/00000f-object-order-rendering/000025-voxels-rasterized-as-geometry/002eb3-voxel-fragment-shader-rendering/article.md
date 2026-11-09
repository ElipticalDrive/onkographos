# Voxel Fragment Shader Rendering
## Overview
Voxel Fragment Shader Rendering is a technique that renders voxels as solid cubes by performing ray-AABB intersection in screen space [S1, unofficial]. It computes surface normals, samples colors, and applies fog entirely within the fragment shader [S1, unofficial].
## How it works
**Inputs and Outputs:**
Vertex inputs include `vColor` (vec3), `vVert` (vec3, voxel center), and `ndotl` (float, dot product of normal and light) [S1, unofficial]. The fragment shader outputs `out_Color` (vec4) [S1, unofficial]. Uniforms include `u_inv_proj_view_rot` (inverse projection-view rotation matrix), `u_far` (far plane distance), `uViewport` (viewport dimensions), `uTex` (texture sampler), and `u_lod` (level of detail/size parameter) [S1, unofficial].

**Data Structures:**
The voxel is defined as an AABB centered at `vxl` (derived from `vVert`) with bounds:
$$
minV = vxl - 0.5 \times u\_lod
$$
$$
maxV = vxl + 0.5 \times u\_lod
$$
[S1, unofficial].

**Steps:**
1. **Ray Direction Calculation:** Compute the world-space ray direction `ray` from the fragment screen coordinates [S1, unofficial]. Normalize screen coordinates to $[0,1]$ using `uViewport` [S1, unofficial]. Convert to clip space with $z=-1$ [S1, unofficial]. Transform by `u_inv_proj_view_rot` and normalize the resulting vector [S1, unofficial].
2. **Ray-AABB Intersection:** Perform ray-AABB intersection against the voxel defined by `minV` and `maxV` [S1, unofficial]. The ray origin is $(0,0,0)$ in the local coordinate system [S1, unofficial].
3. **Hit Point Calculation:** Calculate the hit point `hit` relative to the voxel center:
   $$
   hit = vxl - ray \times t_0
   $$
   where $t_0$ is the entry intersection distance [S1, unofficial]. This point lies in the range $[-0.5 \times u\_lod, 0.5 \times u\_lod]$ [S1, unofficial].
4. **Normal Determination:** Determine the surface normal by finding the maximum absolute component of `hit` [S1, unofficial]. The normal is a unit vector along the axis corresponding to the maximum absolute dimension, signed by the sign of the hit coordinate [S1, unofficial].
5. **Texture Coordinate Calculation:** Calculate UV coordinates based on the normal [S1, unofficial]. If $normal.y \neq 0$, $uv.y = hit.x$, $uv.x = hit.z \times normal.y$ [S1, unofficial]. If $normal.x \neq 0$, $uv.y = hit.y$, $uv.x = hit.z \times normal.x$ [S1, unofficial]. If $normal.z \neq 0$, $uv.y = hit.y$, $uv.x = hit.x \times -normal.z$ [S1, unofficial]. Offset $uv$ by $0.5$ [S1, unofficial].
6. **Depth Setting:** Set the fragment depth to $t_0 / u\_far$ [S1, unofficial].
7. **Fog Application:** Calculate fog start as $256.0 / u\_far$ [S1, unofficial]. Calculate fog factor:
   $$
   fog = \max(gl\_FragDepth - fog\_start, 0.0001)^{0.5}
   $$
   Mix the calculated color with fog color $(0.5, 0.5, 0.7)$ using the fog factor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxel_fragment_shader(gl_FragCoord, vColor, vVert, ndotl, 
                          u_inv_proj_view_rot, u_far, uViewport, u_lod):
    # 1. Ray Direction Calculation
    x = gl_FragCoord.x / uViewport.x
    y = gl_FragCoord.y / uViewport.y
    clip_pos = (x * 2 - 1, y * 2 - 1, -1, 1)
    
    # Transform by inverse projection-view rotation
    transformed = tuple(sum(a * b for a, b in zip(u_inv_proj_view_rot[i], clip_pos)) for i in range(4))
    
    # Normalize to get ray direction
    w = transformed[3]
    if w == 0:
        return None  # Degenerate projection
    ray = tuple(v / w for v in transformed[:3])
    ray_len = math.sqrt(sum(v*v for v in ray))
    if ray_len == 0:
        return None
    ray = tuple(v / ray_len for v in ray)
    
    # 2. Ray-AABB Intersection
    minV = tuple(v - 0.5 * u_lod for v in vVert)
    maxV = tuple(v + 0.5 * u_lod for v in vVert)
    ro = (0.0, 0.0, 0.0)
    rd = ray
    
    # Standard AABB intersection with zero-direction guard [G]
    invR = tuple(1.0 / d if d != 0 else 1e30 for d in rd)
    tbot = tuple((minV[i] - ro[i]) * invR[i] for i in range(3))
    ttop = tuple((maxV[i] - ro[i]) * invR[i] for i in range(3))
    
    tmin = tuple(min(ttop[i], tbot[i]) for i in range(3))
    tmax = tuple(max(ttop[i], tbot[i]) for i in range(3))
    
    t0 = max(tmin[0], tmin[1], tmin[2])
    t1 = min(tmax[0], tmax[1], tmax[2])
    
    if t0 > t1:
        return None  # No intersection
    
    # 3. Hit Point Calculation
    hit = tuple(vVert[i] - ray[i] * t0 for i in range(3))
    
    # 4. Normal Determination
    abs_hit = tuple(abs(h) for h in hit)
    if abs_hit[0] >= abs_hit[1] and abs_hit[0] >= abs_hit[2]:
        normal = (math.copysign(1, hit[0]), 0.0, 0.0)
    elif abs_hit[1] >= abs_hit[0] and abs_hit[1] >= abs_hit[2]:
        normal = (0.0, math.copysign(1, hit[1]), 0.0)
    else:
        normal = (0.0, 0.0, math.copysign(1, hit[2]))
        
    # 5. Texture Coordinate Calculation
    uv = (0.0, 0.0)
    if normal[1] != 0:
        uv = (hit[2] * normal[1], hit[0])
    elif normal[0] != 0:
        uv = (hit[2] * normal[0], hit[1])
    elif normal[2] != 0:
        uv = (hit[0] * -normal[2], hit[1])
    uv = (uv[0] + 0.5, uv[1] + 0.5)
    
    # 6. Depth Setting
    gl_FragDepth = t0 / u_far
    
    # 7. Fog Application
    fog_start = 256.0 / u_far
    fog_factor = math.pow(max(gl_FragDepth - fog_start, 0.0001), 0.5)
    fog_color = (0.5, 0.5, 0.7)
    
    # Base color from vertex input
    final_color = vColor
    
    # Mix with fog
    out_Color = tuple(final_color[i] * (1 - fog_factor) + fog_color[i] * fog_factor for i in range(3)) + (1.0,)
    return out_Color
```

**Parameters:**
*   `u_inv_proj_view_rot`: Inverse projection-view rotation matrix [S1, unofficial].
*   `u_far`: Far plane distance [S1, unofficial].
*   `uViewport`: Viewport dimensions (width, height) [S1, unofficial].
*   `u_lod`: Level of detail or size parameter defining voxel bounds [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The ray direction calculation involves division by `transformed.w` [S1, unofficial]. If `w` is near zero, normalization fails [G].
*   **Boundaries:** The AABB intersection assumes the ray origin is at $(0,0,0)$ in local space [S1, unofficial]. If the voxel is not centered at the origin in local space, the intersection math must be adjusted [G].
*   **Degenerate Input:** If `u_lod` is zero or negative, the AABB bounds collapse or invert, leading to undefined behavior [G].
*   **Fog Start:** The fog start calculation `256.0 / u_far` is hardcoded [S1, unofficial]. If `u_far` is very large, `fog_start` becomes small, potentially causing fog to appear too early [G].
*   **Zero Direction Components:** Division by zero in `invR = 1.0/rd` occurs when the ray is parallel to an axis [G]. A standard guard replaces zero components with a large epsilon to prevent `inf` propagation that breaks `t0 <= t1` checks [G].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
Not given in the sources.

**When to Use:**
Not given in the sources.

**When Not to Use:**
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
