# Voxel Fragment Shader Rendering
## Overview
Voxel Fragment Shader Rendering is a fragment-shader-based technique that renders voxels as solid cubes by performing ray-AABB intersection in screen space [S1, unofficial]. It computes surface normals, samples colors, and applies fog effects entirely within the fragment shader [S1, unofficial]. It builds on the parent family's mechanism of converting scalar field data into geometric primitives submitted to the graphics pipeline [G]. See [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) for shared details.
## How it works
### Inputs and Outputs
**Inputs:**
- `vColor` (`vec3`): Vertex color [S1, unofficial].
- `vVert` (`vec3`): Voxel center coordinates in world space [S1, unofficial].
- `ndotl` (`float`): Dot product of the normal and light direction [S1, unofficial].
- `u_inv_proj_view_rot` (`mat4`): Inverse projection-view rotation matrix [S1, unofficial].
- `u_far` (`float`): Far plane distance of the camera [S1, unofficial].
- `uViewport` (`vec2`): Viewport dimensions (width, height) [S1, unofficial].
- `u_lod` (`float`): Level of detail or voxel size parameter [S1, unofficial].

**Outputs:**
- `out_Color` (`vec4`): Final fragment color [S1, unofficial].

### Data Structures
The voxel is defined implicitly by its center `vxl` (derived from `vVert`) and size `u_lod` [S1, unofficial]. The AABB bounds are:
$$
minV = vxl - 0.5 \times u\_lod
$$
$$
maxV = vxl + 0.5 \times u\_lod
$$
[S1, unofficial].

### Steps
1. **Ray Direction Calculation**: Compute the world-space ray direction `ray` from the fragment screen coordinates. Normalize screen coordinates to $[0,1]$ using `uViewport`. Convert to clip space $[-1,1]$ with $z=-1$. Transform by `u_inv_proj_view_rot` and normalize the resulting vector [S1, unofficial].
2. **Ray-AABB Intersection**: Perform ray-AABB intersection against the voxel defined by `minV` and `maxV`. The ray origin is $(0,0,0)$ [S1, unofficial].
3. **Hit Point Calculation**: Calculate the hit point `hit = vxl - ray * result.x`. This point is relative to the voxel center in the range $[-0.5 \times u\_lod, 0.5 \times u\_lod]$ [S1, unofficial].
4. **Normal Determination**: Determine the surface normal by finding the maximum absolute component of `hit`. The normal is a unit vector along the axis corresponding to the maximum absolute dimension, signed by the sign of the hit coordinate [S1, unofficial].
5. **Texture Coordinate Calculation**: Calculate texture coordinates `uv` based on the normal. Offset `uv` by 0.5 [S1, unofficial].
6. **Depth Setting**: Set the fragment depth to `result.x / u_far` [S1, unofficial].
7. **Fog Application**: Apply fog. Calculate `fog_start = 256.0 / u_far`. Calculate fog factor `fog = pow(max(depth - fog_start, 0.0001), 0.5)`. Mix the calculated color with fog color $(0.5, 0.5, 0.7)$ using the fog factor [S1, unofficial].

### Formulas
**Ray Direction:**
$$
ray = \text{normalize}\left( \frac{(u\_inv\_proj\_view\_rot \times vec4(x \times 2 - 1, y \times 2 - 1, -1, 1)).xyz}{(u\_inv\_proj\_view\_rot \times vec4(x \times 2 - 1, y \times 2 - 1, -1, 1)).w} \right)
$$
where $x$ and $y$ are normalized screen coordinates [S1, unofficial].

**AABB Intersection:**
$$
invR = 1.0 / ray
$$
$$
tbot = (minV - ro) \times invR
$$
$$
ttop = (maxV - ro) \times invR
$$
$$
tmin = \min(ttop, tbot)
$$
$$
tmax = \max(ttop, tbot)
$$
$$
t0 = \max(\max(tmin.x, tmin.y), tmin.z)
$$
$$
t1 = \min(\min(tmax.x, tmax.y), tmax.z)
$$
Hit if $t0 \le t1$ [S1, unofficial].

**Hit Point:**
$$
hit = vxl - ray \times t0
$$
where $t0$ is the entry intersection distance [S1, unofficial].

**UV Coordinates:**
- If $normal.y \neq 0$: $uv.y = hit.x$, $uv.x = hit.z \times normal.y$
- If $normal.x \neq 0$: $uv.y = hit.y$, $uv.x = hit.z \times normal.x$
- If $normal.z \neq 0$: $uv.y = hit.y$, $uv.x = hit.x \times -normal.z$
Offset $uv$ by 0.5 [S1, unofficial].

**Fog:**
$$
fog\_start = 256.0 / u\_far
$$
$$
fog = \text{pow}(\max(depth - fog\_start, 0.0001), 0.5)
$$
$$
out\_Color = \text{mix}(calculated\_color, vec4(0.5, 0.5, 0.7, 1.0), fog)
$$
[S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fragment_shader(v_color, v_vert, ndotl, u_inv_proj_view_rot, u_far, u_viewport, u_lod, frag_coord):
    """
    Renders a voxel as a solid cube using ray-AABB intersection in screen space.
    
    Args:
        v_color (vec3): Vertex color.
        v_vert (vec3): Voxel center in world space.
        ndotl (float): Dot product of normal and light.
        u_inv_proj_view_rot (mat4): Inverse projection-view rotation matrix.
        u_far (float): Far plane distance.
        u_viewport (vec2): Viewport dimensions (width, height).
        u_lod (float): Voxel size parameter.
        frag_coord (vec2): Fragment screen coordinates.
        
    Returns:
        vec4: Final fragment color.
        float: Fragment depth.
    """
    # 1. Compute screen coordinates normalized to [0, 1]
    x = frag_coord.x / u_viewport.x
    y = frag_coord.y / u_viewport.y
    
    # 2. Convert to clip space [-1, 1] with z = -1
    clip_pos = vec4(x * 2 - 1, y * 2 - 1, -1, 1)
    
    # 3. Transform by inverse projection-view rotation
    transformed = u_inv_proj_view_rot * clip_pos
    
    # 4. Calculate ray direction
    ray = normalize(transformed.xyz / transformed.w)
    
    # 5. Define voxel AABB bounds
    vxl = v_vert
    min_v = vxl - 0.5 * u_lod
    max_v = vxl + 0.5 * u_lod
    
    # 6. Ray-AABB Intersection
    ro = vec3(0, 0, 0)
    inv_r = 1.0 / ray
    
    tbot = (min_v - ro) * inv_r
    ttop = (max_v - ro) * inv_r
    
    tmin = min(ttop, tbot)
    tmax = max(ttop, tbot)
    
    t0 = max(max(tmin.x, tmin.y), tmin.z)
    t1 = min(min(tmax.x, tmax.y), tmax.z)
    
    # Check for hit
    if t0 > t1:
        return vec4(0.0, 0.0, 0.0, 0.0), t0 / u_far
    
    # 7. Calculate hit point relative to voxel center
    hit = vxl - ray * t0
    
    # 8. Determine surface normal
    abs_hit = abs(hit)
    if abs_hit.x >= abs_hit.y and abs_hit.x >= abs_hit.z:
        normal = vec3(sign(hit.x), 0.0, 0.0)
    elif abs_hit.y >= abs_hit.x and abs_hit.y >= abs_hit.z:
        normal = vec3(0.0, sign(hit.y), 0.0)
    else:
        normal = vec3(0.0, 0.0, sign(hit.z))
        
    # 9. Calculate texture coordinates (uv)
    uv = vec2(0.0, 0.0)
    if normal.y != 0.0:
        uv.y = hit.x
        uv.x = hit.z * normal.y
    elif normal.x != 0.0:
        uv.y = hit.y
        uv.x = hit.z * normal.x
    else: # normal.z != 0.0
        uv.y = hit.y
        uv.x = hit.x * -normal.z
        
    uv = uv + vec2(0.5, 0.5)
    
    # 10. Set fragment depth
    depth = t0 / u_far
    
    # 11. Calculate color
    # Texture sampling logic is not detailed in the sources [G].
    # Using vertex color and ndotl as a base approximation.
    color = v_color * (0.5 + 0.5 * ndotl)
    
    # 12. Apply fog
    fog_start = 256.0 / u_far
    fog_factor = pow(max(depth - fog_start, 0.0001), 0.5)
    fog_color = vec4(0.5, 0.5, 0.7, 1.0)
    
    out_color = mix(color, fog_color, fog_factor)
    
    return out_color, depth
```

### Parameters
- `u_inv_proj_view_rot`: Inverse projection-view rotation matrix [S1, unofficial].
- `u_far`: Far plane distance of the camera [S1, unofficial].
- `uViewport`: Viewport dimensions (width, height) [S1, unofficial].
- `uTex`: Texture sampler (declared but commented out in color calculation) [S1, unofficial].
- `u_lod`: Level of detail or size parameter used to define the voxel bounds relative to its center `vxl` [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: The ray direction calculation involves division by the w-component of the transformed clip coordinate. If this value is near zero, it may cause instability [G].
- **Boundaries**: The AABB intersection assumes the ray origin is at $(0,0,0)$ in the local voxel space. If the voxel is not centered at the origin in its local coordinate system, the intersection logic will fail [S1, unofficial].
- **Degenerate Input**: If `u_lod` is zero or negative, the voxel bounds will be invalid [G].
- **Fog Calculation**: The fog factor uses the fragment depth, which must be set before the fog calculation to ensure correct depth-based fogging [S1, unofficial].
## Operating characteristics
Not given in the sources.
## Background
Voxel Fragment Shader Rendering is an unofficial technique described in the `voxplat` repository by kosshi-net [S1, unofficial]. It leverages modern GPU capabilities to perform ray-casting in the fragment shader, providing a flexible approach to voxel rendering. The method is part of a broader family of shader-based rendering techniques for voxels.
## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
