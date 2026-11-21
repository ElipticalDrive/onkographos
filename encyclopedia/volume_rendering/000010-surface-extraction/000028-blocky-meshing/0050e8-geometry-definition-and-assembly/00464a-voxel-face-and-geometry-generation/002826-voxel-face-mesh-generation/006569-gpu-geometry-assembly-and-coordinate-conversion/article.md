# GPU Geometry Assembly and Coordinate Conversion

Also called: GPURenderer, GPURenderer Face Triangulation and Coordinate Conversion

## Overview
This method converts raw voxel face data into typed arrays (position, normal, UV, index, color) suitable for direct upload to a GPU buffer (e.g., WebGL or WebGPU `BufferGeometry`). It solves the problem of transforming engine-internal voxel coordinates into a target graphics API coordinate system while ensuring correct triangle winding order. It differs from [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) by handling the coordinate system conversion and winding correction logic explicitly, whereas the parent method focuses on populating a generic `MeshData` buffer based on face direction.

## How it works
**Inputs and Outputs:**
- **Input:** A list of raw voxel faces (obtained via `engine.getFaces({ raw: true })`), where each face contains vertex coordinates (`v`), a type (`face.type`), and optional style data.
- **Output:** A `GPUGeometry` object containing typed arrays: `position` (Float32Array, XYZ), `normal` (Float32Array, XYZ), `uv` (Float32Array, UV), `index` (Uint32Array, triangle indices), and optionally `color` (Float32Array, RGB).

**Data Structures:**
The `GPUGeometry` structure consists of:
- `position`: Float32Array, 3 floats per vertex.
- `normal`: Float32Array, 3 floats per vertex (flat-shaded).
- `uv`: Float32Array, 2 floats per vertex.
- `index`: Uint32Array, 3 indices per triangle.
- `color`: Float32Array, 3 floats per vertex (present only if `options.color` is true).
- `faceCount`: Number of quad faces encoded.
- `vertexCount`: Total vertices (`faceCount * 4`).

**Parameters:**
- `yUp` (boolean, default `false`): If true, converts from Heerich's Y-down/Z-into-screen space to Three.js Y-up/Z-toward-viewer space by negating Y and Z coordinates. This preserves winding order (determinant +1). [S1, unofficial]
- `scale` (number, default `1`): Uniform scale applied to vertex positions. [S1, unofficial]
- `color` (boolean, default `false`): If true, parses `face.style.fill` and includes a color array. [S1, unofficial]
- `defaultColor` (array, default `[1,1,1]`): Fallback color when parsing fails. [S1, unofficial]

**Steps:**
1. Initialize empty typed arrays for `position`, `normal`, `uv`, `index`, and optionally `color`. Determine array sizes based on the number of renderable faces.
2. Iterate through each face in the input list.
3. Skip faces where `face.type` is `'content'` or if vertex data is absent.
4. Determine the face normal `n`:
   - If `face.n` is present, use it.
   - Otherwise, look up `FACE_NORMALS[face.type]`.
   - If neither exists, default to `[0, 0, 0]`.
5. Determine the fill color:
   - If `options.color` is true, parse `face.style.fill` using `parseCSSColor`.
   - If parsing fails or `color` is disabled, use `defaultColor`.
6. For each of the 4 vertices `v` (indices 0–3) in the face quad:
   - Compute `ySign = yUp ? -1 : 1` and `zSign = yUp ? -1 : 1`.
   - Compute position:
     - `x = v[0] * scale`
     - `y = v[1] * scale * ySign`
     - `z = v[2] * scale * zSign`
   - Compute normal:
     - `nx = n[0]`
     - `ny = n[1] * ySign`
     - `nz = n[2] * zSign`
   - Compute UV: Look up `QUAD_UVS[i]` based on vertex index `i`.
   - Compute color: If enabled, use the parsed fill color.
   - Store these values in the respective arrays.
7. Determine winding order for the quad to ensure Counter-Clockwise (CCW) orientation:
   - Let `v0, v1, v2, v3` be the 4 vertices of the quad in order.
   - Compute edge vectors: `e1 = v1 - v0`, `e2 = v2 - v0`.
   - Compute cross product `cross = e1 × e2`.
   - Compute dot product `crossDotN = cross · n`.
   - Formula for `crossDotN`:
     $$
     \text{crossDotN} = (e_{1y} e_{2z} - e_{1z} e_{2y}) n_x + (e_{1z} e_{2x} - e_{1x} e_{2z}) n_y + (e_{1x} e_{2y} - e_{1y} e_{2x}) n_z
     $$
   - If `crossDotN >= 0`, the vertices are already CCW relative to the normal. Indices: `[base, base+1, base+2, base, base+2, base+3]`.
   - If `crossDotN < 0`, flip the winding. Indices: `[base, base+2, base+1, base, base+3, base+2]`.
   - Append these 6 indices to the `index` array.
8. Return the `GPUGeometry` object.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_geometry_assembly(faces, options):
    y_up = options.get('yUp', False)
    scale = options.get('scale', 1.0)
    enable_color = options.get('color', False)
    default_color = options.get('defaultColor', [1.0, 1.0, 1.0])
    
    y_sign = -1 if y_up else 1
    z_sign = -1 if y_up else 1
    
    # Pre-calculate UVs for a quad (example mapping)
    quad_uvs = [[0, 0], [1, 0], [1, 1], [0, 1]]
    
    # Estimate sizes
    face_count = 0
    for face in faces:
        if face.type != 'content' and face.vertices is not None:
            face_count += 1
            
    vertex_count = face_count * 4
    index_count = face_count * 6
    
    position = [0.0] * (vertex_count * 3)
    normal = [0.0] * (vertex_count * 3)
    uv = [0.0] * (vertex_count * 2)
    indices = [0] * index_count
    color = [0.0] * (vertex_count * 3) if enable_color else None
    
    v_idx = 0
    i_idx = 0
    
    for face in faces:
        if face.type == 'content' or face.vertices is None:
            continue
            
        # Determine normal
        if face.n is not None:
            n = face.n
        else:
            n = FACE_NORMALS.get(face.type, [0, 0, 0])
            
        # Determine color
        if enable_color:
            c = parse_css_color(face.style.fill)
            if c is None:
                c = default_color
        else:
            c = None
            
        verts = face.vertices # List of [x, y, z]
        
        # Process vertices
        for i in range(4):
            v = verts[i]
            # Position
            position[v_idx * 3] = v[0] * scale
            position[v_idx * 3 + 1] = v[1] * scale * y_sign
            position[v_idx * 3 + 2] = v[2] * scale * z_sign
            
            # Normal
            normal[v_idx * 3] = n[0]
            normal[v_idx * 3 + 1] = n[1] * y_sign
            normal[v_idx * 3 + 2] = n[2] * z_sign
            
            # UV
            uv_uv = quad_uvs[i]
            uv[v_idx * 2] = uv_uv[0]
            uv[v_idx * 2 + 1] = uv_uv[1]
            
            # Color
            if enable_color:
                color[v_idx * 3] = c[0]
                color[v_idx * 3 + 1] = c[1]
                color[v_idx * 3 + 2] = c[2]
                
            v_idx += 1
            
        # Winding check
        v0 = [position[v_idx-4*3], position[v_idx-4*3+1], position[v_idx-4*3+2]]
        v1 = [position[v_idx-3*3], position[v_idx-3*3+1], position[v_idx-3*3+2]]
        v2 = [position[v_idx-2*3], position[v_idx-2*3+1], position[v_idx-2*3+2]]
        
        e1 = [v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2]]
        e2 = [v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2]]
        
        cross_x = e1[1]*e2[2] - e1[2]*e2[1]
        cross_y = e1[2]*e2[0] - e1[0]*e2[2]
        cross_z = e1[0]*e2[1] - e1[1]*e2[0]
        
        cross_dot_n = cross_x*n[0] + cross_y*n[1] + cross_z*n[2]
        
        base = i_idx
        if cross_dot_n >= 0:
            indices[i_idx] = base
            indices[i_idx+1] = base + 1
            indices[i_idx+2] = base + 2
            indices[i_idx+3] = base
            indices[i_idx+4] = base + 2
            indices[i_idx+5] = base + 3
        else:
            indices[i_idx] = base
            indices[i_idx+1] = base + 2
            indices[i_idx+2] = base + 1
            indices[i_idx+3] = base
            indices[i_idx+4] = base + 3
            indices[i_idx+5] = base + 2
            
        i_idx += 6
        
    return GPUGeometry(position, normal, uv, indices, color, face_count, vertex_count)
```

**Edge Cases and Pitfalls:**
- **Coordinate System Conversion:** When converting from Heerich (X-right, Y-down, Z-into-screen) to Three.js (X-right, Y-up, Z-toward-viewer), both Y and Z must be negated (`ySign=-1, zSign=-1`) to preserve winding order (determinant +1). Using `scale.y = -1` alone changes handedness and breaks winding. [S1, unofficial]
- **Winding Order:** The winding check is critical for back-face culling. If the normal is zero or vertices are degenerate, the cross product may be zero, leading to undefined behavior. [S1, unofficial]
- **Color Parsing:** `parseCSSColor` must handle various formats (`#rgb`, `#rrggbb`, `rgb()`, `rgba()`). If it fails, the default color must be used to avoid undefined values. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of exposed faces $O(F)$, where $F$ is the number of faces. Each face is processed once, with constant-time operations per vertex.
- **Memory Complexity:** Linear with respect to the number of vertices $O(V)$, where $V = 4F$. The output arrays scale directly with the geometry size.
- **Performance:** Efficient for bulk geometry generation. The method avoids dynamic resizing by pre-calculating array sizes. [S1, unofficial]
- **Strengths:** Handles coordinate system conversion and winding correction in a single pass. Produces ready-to-upload typed arrays. [S1, unofficial]
- **Limitations:** Does not perform occlusion culling (this is handled by the engine's `getFaces` call). [S1, unofficial]

## Background
This method is part of the Heerich voxel engine's rendering pipeline, specifically the `GPURenderer` component. It bridges the gap between the engine's internal voxel representation and standard GPU graphics APIs. The method was introduced to support WebGL and WebGPU rendering, allowing for efficient batched geometry uploads. [S1, unofficial]

## Sources
[S1] meodai/heerich, https://github.com/meodai/heerich, fetched 2026-10-26 (unofficial)
