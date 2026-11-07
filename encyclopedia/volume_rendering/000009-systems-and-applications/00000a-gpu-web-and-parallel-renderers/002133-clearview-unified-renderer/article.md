# ClearView (Unified) Renderer

Also called: ClearView, ClearView Unified Buffer, ClearView WebGL2 Spike

## Overview
ClearView (Unified) is a 3-pass WebGL2 renderer designed for the real-time visualization of unstructured hexahedral meshes, specifically targeting wireframe and scalar field rendering in web environments [S1, unofficial]. It solves the problem of rendering complex volumetric data structures without server-side support for Shader Storage Buffer Objects (SSBOs) by encoding mesh geometry into `image2D` textures and using a Parallel Prefix List (PPLL) for fragment management [S1, unofficial]. Unlike sibling methods that may rely on CPU-side sorting or different GPU architectures, ClearView performs all sorting and compositing on the GPU via atomic operations and a final resolve pass, enabling interactive frame rates for large datasets like the SPE-9 benchmark [S1, unofficial].

## How it works
The method consists of three distinct rendering passes: Clear, Gather, and Resolve. It operates on hexahedral meshes where each cell has 8 vertices, 12 edges, and 6 faces.

### Inputs and Outputs
*   **Input:** A hexahedral mesh defined by vertex positions, cell connectivity, and per-cell scalar values. The mesh is converted into four texture buffers: `faceTex`, `vertexTex`, `edgeTex`, and `cellScalarTex` [S1, unofficial].
*   **Output:** A 2D image buffer containing the alpha-blended visualization of the mesh faces and edges [S1, unofficial].

### Data Structures and Layouts
The renderer avoids SSBOs by mapping data to 2D textures with a fixed row width of `TEX_WIDTH = 4096` [S1, unofficial].

1.  **Vertex Buffer (`vertexTex`):**
    *   Format: `RGBA32F` (Float32Array).
    *   Layout: 1 texel per vertex slot. Each texel stores $(x, y, z, \text{scalar})$.
    *   Mapping: Vertices are stored in a Standard (STD) corner order. VTK connectivity indices are mapped to STD using `VTK_TO_STD = [0, 1, 3, 2, 4, 5, 7, 6]` [S1, unofficial].
    *   Dimensions: `width = 4096`, `height = ceil(vertexCount / 4096)` [S1, unofficial].

2.  **Face Buffer (`faceTex`):**
    *   Format: `RG32UI` (Uint32Array).
    *   Layout: 2 texels per face. Stores indices for the 4 corners of the face in CCW order (viewed from outside) and 4 edge indices [S1, unofficial].
    *   Dimensions: `width = 4096`, `height = ceil(faceCount / 2048)` (since 2 texels/face) [S1, unofficial].

3.  **Edge Buffer (`edgeTex`):**
    *   Format: `RG32F` (Float32Array).
    *   Layout: 1 texel per edge. Stores edge geometry data [S1, unofficial].
    *   Dimensions: `width = 4096`, `height = ceil(edgeCount / 4096)` [S1, unofficial].

4.  **Cell Scalar Buffer (`cellScalarTex`):**
    *   Format: `R32F` (Float32Array).
    *   Layout: 1 texel per face, storing the scalar value associated with the cell [S1, unofficial].
    *   Dimensions: `width = 4096`, `height = ceil(faceCount / 4096)` [S1, unofficial].

5.  **PPLL Structures (Runtime):**
    *   **Start Offset Texture (`startOffsetTex`):** `uimage2D` (R32UI). Sized to the viewport. Stores the starting index in the fragment buffer for each pixel [S1, unofficial].
    *   **Fragment Buffer Texture (`fragmentBufferTex`):** `uimage2D` (RGBA32UI). Fixed size `2048 \times 2048` (4 million entries). Stores fragment data (position, color, depth, etc.) [S1, unofficial].
    *   **Atomic Counter Buffer:** Single `Uint32` value, bound to binding point 0, tracking the total number of fragments inserted [S1, unofficial].

### Hexahedral Topology
The renderer uses a specific enumeration for hexahedral cells to ensure outward-facing normals [S1, unofficial]:
*   **STD Corner Indices:** $0=c_{000}, 1=c_{100}, 2=c_{010}, 3=c_{110}, 4=c_{001}, 5=c_{101}, 6=c_{011}, 7=c_{111}$.
*   **Edges:** 12 per cell, defined as pairs of STD indices:
    ```python
    HEX_EDGES = [
      [0, 1], [1, 3], [2, 3], [0, 2],  # bottom (z=0)
      [4, 5], [5, 7], [6, 7], [4, 6],  # top (z=1)
      [0, 4], [1, 5], [3, 7], [2, 6],  # vertical struts
    ]
    ```
*   **Faces:** 6 per cell. Each face lists 4 STD corner indices in CCW order (viewed from outside) and 4 edge indices. Example for the -z (bottom) face: `corners: [0, 2, 3, 1]`, `edges: [3, 2, 1, 0]` [S1, unofficial].

### Parameters
*   `FRAG_BUFFER_W = 2048`
*   `FRAG_BUFFER_H = 2048`
*   `LINKED_LIST_SIZE = FRAG_BUFFER_W \times FRAG_BUFFER_H = 4,194,304`
*   `TEX_WIDTH = 4096`
*   `RESOLVE_CUTOFF = 32` (fragments per pixel for insertion sort threshold) [S1, unofficial].
*   **LOD/Coloring Uniforms:**
    *   `uMaxLodValue = 1.0`
    *   `uSelectedLodValueFocus = 0.4`
    *   `uSelectedLodValueContext = 0.7`
    *   `uImportantLineBoostFactor = 0.5` [S1, unofficial].

### Numbered Steps

**Preprocessing: Buffer Construction**
1.  Calculate `faceCount = 6 * nCells`, `vertexCount = 8 * nCells`, `edgeCount = 12 * nCells` [S1, unofficial].
2.  Allocate `vertexTex`, `faceTex`, `edgeTex`, `cellScalarTex` with dimensions derived from `TEX_WIDTH` [S1, unofficial].
3.  Populate `vertexTex`: For each cell $c$ and STD corner $k$, map to VTK index `vtkCorner = VTK_TO_STD[k]`. Get global point index `pi = mesh.cellConn[c * 8 + vtkCorner]`. Store `mesh.positions[pi*3...pi*3+2]` and `mesh.scalars[pi]` into the texel at linear index `c * 8 + k` [S1, unofficial].
4.  Populate `faceTex`: For each cell, store face corner and edge indices according to `HEX_FACES` [S1, unofficial].
5.  Initialize `startOffsetTex` with -1 and `fragmentBufferTex` with zeros [S1, unofficial].
6.  Initialize `atomicCounterBuf` to 0 [S1, unofficial].

**Pass 1: Clear**
1.  Bind `startOffsetTex` (binding 0, R32UI) and `fragmentBufferTex` (binding 1, RGBA32UI) as `image2D` [S1, unofficial].
2.  Bind `atomicCounterBuf` to binding 0 [S1, unofficial].
3.  Set uniforms: `uStartOffsetTex`, `uFragmentBufferTex`, `uViewportW`, `uViewportH` [S1, unofficial].
4.  Disable depth test and culling [S1, unofficial].
5.  Draw 3 triangles (fullscreen quad) to reset all values in `startOffsetTex` to -1 [S1, unofficial].

**Pass 2: Gather**
1.  Bind `startOffsetTex` (binding 0, R32UI) and `fragmentBufferTex` (binding 1, RGBA32UI) as `image2D` [S1, unofficial].
2.  Bind `atomicCounterBuf` to binding 0 [S1, unofficial].
3.  Bind data textures: `faceTex` (unit 0), `vertexTex` (unit 1), `edgeTex` (unit 2), `cellScalarTex` (unit 3) [S1, unofficial].
4.  Set uniforms: `uFaceTex`, `uVertexTex`, `uEdgeTex`, `uCellScalarTex`, `uStartOffsetTex`, `uFragmentBufferTex`, `uViewportW`, `uViewportH`, `uLinkedListSize`, camera matrices (`MVP`, `ModelViewInverse`), camera position, look direction, sphere parameters, LOD values, colors, scalar range, and FOV [S1, unofficial].
5.  Disable depth test and culling [S1, unofficial].
6.  Draw `faceCount * 6` triangles (representing the faces of the hexahedral mesh) [S1, unofficial].
    *   *Fragment Shader Logic:* For each fragment, compute ray-sphere and ray-plane intersections to determine visibility and depth. Use atomic operations to insert the fragment into the PPLL structure in `fragmentBufferTex` and update `startOffsetTex` [S1, unofficial].

**Pass 3: Resolve**
1.  Bind `startOffsetTex` and `fragmentBufferTex` as `READ_ONLY` image2D textures [S1, unofficial].
2.  Set uniforms: `uStartOffsetTex`, `uFragmentBufferTex`, `uViewportW`, `uViewportH`, `uLinkedListSize` [S1, unofficial].
3.  Disable depth test and culling [S1, unofficial].
4.  Draw 3 triangles (fullscreen quad) [S1, unofficial].
    *   *Fragment Shader Logic:* Walk the PPLL for the current pixel's start offset. Sort fragments using insertion sort if the count is $\le 32$. Perform alpha blending to produce the final color [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper: Ray-Sphere Intersection
def ray_sphere_intersection(ray_origin, ray_direction, sphere_center, sphere_radius):
    # Returns (t0, t1, intersection_position) or None
    A = dot(ray_direction, ray_direction)
    B = 2.0 * dot(ray_direction, ray_origin - sphere_center)
    C = dot(ray_origin - sphere_center, ray_origin - sphere_center) - sphere_radius ** 2
    discriminant = B ** 2 - 4.0 * A * C
    
    if discriminant < 0:
        return None
        
    sqrt_disc = sqrt(discriminant)
    t0 = (-B - sqrt_disc) / (2.0 * A)
    t1 = (-B + sqrt_disc) / (2.0 * A)
    intersection_pos = ray_origin + t0 * ray_direction
    return (t0, t1, intersection_pos)

# Helper: Ray-Plane Intersection
def ray_plane_intersection(ray_origin, ray_direction, plane_point, plane_normal):
    # Returns intersection_position or None
    ln = dot(plane_normal, ray_direction)
    if abs(ln) < 1e-4:
        return None
    pos = dot(plane_normal, ray_origin) - dot(plane_normal, plane_point)
    t = -pos / ln
    return ray_origin + t * ray_direction

# Shader Loader: Resolve Includes
def resolve_shader_source(src, sources, depth=0):
    if depth > 10:
        raise Error("Include depth exceeded")
    lines = src.split('\n')
    changed = False
    for i, line in enumerate(lines):
        match = re.match(r'^\s*#include\s+"([^"]+)"\s*$', line)
        if match:
            name = match.group(1)
            if name not in sources:
                raise Error(f"Unknown shader include: {name}")
            lines[i] = resolve_shader_source(sources[name], sources, depth + 1)
            changed = True
    return '\n'.join(lines) if changed else src

# Main Render Pipeline
def clearview_render(mesh, camera, canvas):
    # 1. Build Buffers
    buffers = build_clearview_buffers(mesh) # Returns faceTex, vertexTex, edgeTex, cellScalarTex
    
    # 2. Initialize PPLL Structures
    start_offset_tex = create_image2D(canvas.width, canvas.height, format='R32UI')
    fragment_buffer_tex = create_image2D(2048, 2048, format='RGBA32UI')
    atomic_counter_buf = create_buffer([0], type='ATOMIC_COUNTER_BUFFER')
    
    # 3. Pass 1: Clear
    gl.use_program(clear_program)
    gl.bind_image_texture(0, start_offset_tex, 0, gl.READ_WRITE, gl.R32UI)
    gl.bind_image_texture(1, fragment_buffer_tex, 0, gl.READ_WRITE, gl.RGBA32UI)
    gl.bind_buffer_base(gl.ATOMIC_COUNTER_BUFFER, 0, atomic_counter_buf)
    gl.set_uniform1i(clear_program, 'uStartOffsetTex', 0)
    gl.set_uniform1i(clear_program, 'uFragmentBufferTex', 1)
    gl.set_uniform1i(clear_program, 'uViewportW', canvas.width)
    gl.set_uniform1i(clear_program, 'uViewportH', canvas.height)
    gl.disable(gl.DEPTH_TEST)
    gl.disable(gl.CULL_FACE)
    gl.draw_arrays(gl.TRIANGLES, 0, 3)
    
    # 4. Pass 2: Gather
    gl.use_program(gather_program)
    gl.bind_image_texture(0, start_offset_tex, 0, gl.READ_WRITE, gl.R32UI)
    gl.bind_image_texture(1, fragment_buffer_tex, 0, gl.READ_WRITE, gl.RGBA32UI)
    gl.bind_buffer_base(gl.ATOMIC_COUNTER_BUFFER, 0, atomic_counter_buf)
    
    gl.active_texture(gl.TEXTURE0)
    gl.bind_texture(gl.TEXTURE_2D, buffers.faceTex)
    gl.active_texture(gl.TEXTURE1)
    gl.bind_texture(gl.TEXTURE_2D, buffers.vertexTex)
    gl.active_texture(gl.TEXTURE2)
    gl.bind_texture(gl.TEXTURE_2D, buffers.edgeTex)
    gl.active_texture(gl.TEXTURE3)
    gl.bind_texture(gl.TEXTURE_2D, buffers.cellScalarTex)
    
    gl.set_uniform1i(gather_program, 'uFaceTex', 0)
    gl.set_uniform1i(gather_program, 'uVertexTex', 1)
    gl.set_uniform1i(gather_program, 'uEdgeTex', 2)
    gl.set_uniform1i(gather_program, 'uCellScalarTex', 3)
    gl.set_uniform1i(gather_program, 'uStartOffsetTex', 0)
    gl.set_uniform1i(gather_program, 'uFragmentBufferTex', 1)
    gl.set_uniform1i(gather_program, 'uViewportW', canvas.width)
    gl.set_uniform1i(gather_program, 'uViewportH', canvas.height)
    gl.set_uniform1i(gather_program, 'uLinkedListSize', 4194304)
    # Set camera uniforms...
    
    gl.draw_arrays(gl.TRIANGLES, 0, buffers.faceCount * 6)
    
    # 5. Pass 3: Resolve
    gl.use_program(resolve_program)
    gl.bind_image_texture(0, start_offset_tex, 0, gl.READ_ONLY, gl.R32UI)
    gl.bind_image_texture(1, fragment_buffer_tex, 0, gl.READ_ONLY, gl.RGBA32UI)
    
    gl.set_uniform1i(resolve_program, 'uStartOffsetTex', 0)
    gl.set_uniform1i(resolve_program, 'uFragmentBufferTex', 1)
    gl.set_uniform1i(resolve_program, 'uViewportW', canvas.width)
    gl.set_uniform1i(resolve_program, 'uViewportH', canvas.height)
    gl.set_uniform1i(resolve_program, 'uLinkedListSize', 4194304)
    
    gl.draw_arrays(gl.TRIANGLES, 0, 3)
```

### Edge Cases and Pitfalls
*   **SwiftShader Compatibility:** SwiftShader's WebGL2 implementation does not support `image2D` or `atomic_uint`, returning `null`/`undefined` for `MAX_IMAGE_UNITS`. This method will fail on such drivers [S1, unofficial].
*   **SSBO Absence:** WebGL2 lacks SSBOs. The method handles this by mapping SSBO layouts to `image2D` textures with explicit format bindings (`gl.bindImageTexture`) and a fixed row width of 4096 to ensure data fits on any driver [S1, unofficial].
*   **Shader Recursion:** Shader include resolution must prevent infinite recursion. The loader enforces a maximum depth of 10 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The Gather pass is $O(F)$ where $F$ is the number of faces, dominated by atomic operations. The Resolve pass is $O(P \cdot K \log K)$ or $O(P \cdot K)$ where $P$ is the number of pixels and $K$ is the number of fragments per pixel (capped at 32 for insertion sort) [S1, unofficial].
*   **Memory Complexity:** $O(V + F + E)$ for mesh data textures, plus $O(W \times H)$ for `startOffsetTex` and $O(4 \times 10^6)$ entries for `fragmentBufferTex` (64 MB at RGBA32UI) [S1, unofficial].
*   **Performance:** Tuned for SPE-9 (54,000 faces, ~30-60 depth). The 2048x2048 fragment buffer provides sufficient capacity for typical view depths [S1, unofficial].
*   **Strengths:** Enables real-time web rendering of unstructured meshes without server-side SSBO support. Efficient use of GPU atomics for sorting.
*   **Limitations:** Dependent on WebGL2 `image2D` and atomic counter support. Fixed fragment buffer size may limit very deep scenes.

## Background
ClearView (Unified) is a WebGL2 renderer implementation for hexahedral mesh visualization, adapted from a C++ reference implementation (`ClearViewRenderer_FacesUnified.cpp`) [S1, unofficial]. It was developed as part of the `geresim-spike` project to provide a unified buffer approach for web-based scientific visualization, addressing the limitations of WebGL2 regarding buffer objects [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
