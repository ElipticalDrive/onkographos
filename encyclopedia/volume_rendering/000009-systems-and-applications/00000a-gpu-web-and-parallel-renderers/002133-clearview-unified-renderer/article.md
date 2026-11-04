# ClearView (Unified) Renderer

Also called: ClearView, ClearView Unified Buffer, ClearView WebGL2 Spike

## Overview
ClearView (Unified) is a 3-pass WebGL2 renderer for unstructured hexahedral meshes that uses a Parallel Prefix List (PPLL) to render wireframes and scalar fields on the GPU [S1, unofficial]. It solves the problem of rendering complex volumetric meshes in web browsers by adapting a desktop GPU technique (which typically relies on Shader Storage Buffer Objects) to WebGL2's texture-only data storage model [S1, unofficial]. It is distinguished from its siblings by its explicit 3-pass architecture (Clear, Gather, Resolve) and its use of atomic operations on image textures to build a linked list of fragments for depth-sorted alpha blending [S1, unofficial].

## How it works
The method consists of three sequential GPU passes. It operates on hexahedral meshes where each cell has 8 corners, 12 edges, and 6 faces.

### Inputs and Outputs
*   **Input:** A hexahedral mesh defined by vertex positions, cell connectivity (mapping 8 VTK-style corners to 8 STD corners), and per-vertex scalar attributes.
*   **Output:** A final 2D image with depth-sorted, alpha-blended wireframes and scalar field visualizations.

### Data Structures and Layouts
WebGL2 lacks Shader Storage Buffer Objects (SSBOs), so all mesh data is encoded into `image2D` textures with a fixed row width of `TEX_WIDTH = 4096` [S1, unofficial].

1.  **Vertex Texture (`vertexTex`)**:
    *   Format: `RGBA32F` (Float32Array).
    *   Layout: 1 texel per vertex slot. Each texel stores `(x, y, z, scalar_attribute)`.
    *   Size: `height = ceil(8 * nCells / 4096)`.
    *   Corner Mapping: VTK corner indices `[0, 1, 3, 2, 4, 5, 7, 6]` are mapped to STD indices `[0..7]` where `0=c000, 1=c100, ..., 7=c111` [S1, unofficial].

2.  **Face Texture (`faceTex`)**:
    *   Format: `RG32UI` (Uint32Array).
    *   Layout: 2 texels per face. Stores face-local corner indices and edge indices.
    *   Size: `height = ceil(6 * nCells / 4096)`.
    *   Enumeration: 6 faces per cell, defined in `HEX_FACES` array with corners in face-local CCW order (viewed from outside) to ensure outward-facing normals [S1, unofficial].

3.  **Edge Texture (`edgeTex`)**:
    *   Format: `RG32F` (Float32Array).
    *   Layout: 1 texel per edge. Stores edge indices.
    *   Size: `height = ceil(12 * nCells / 4096)`.
    *   Definition: 12 edges per cell, defined as pairs of STD corner indices in `HEX_EDGES` [S1, unofficial].

4.  **Cell Scalar Texture (`cellScalarTex`)**:
    *   Format: `R32F` (Float32Array).
    *   Layout: 1 texel per face. Stores scalar values associated with the face/cell.
    *   Size: `height = ceil(6 * nCells / 4096)`.

5.  **PPLL Structures**:
    *   **Start Offset Texture (`startOffsetTex`)**: `R32UI` image texture. Size matches the viewport. Stores the starting index in the fragment buffer for each pixel.
    *   **Fragment Buffer Texture (`fragmentBufferTex`)**: `RGBA32UI` image texture. Fixed size `2048x2048` (4 million entries). Stores fragment data (position, color, depth, etc.).
    *   **Atomic Counter Buffer**: Single `Uint32` value. Tracks the total number of fragments inserted into the buffer.

### Numbered Steps

1.  **Initialization**:
    *   Create `startOffsetTex` (viewport size) and `fragmentBufferTex` (2048x2048).
    *   Create and bind an atomic counter buffer (initial value 0).
    *   Upload mesh data to `faceTex`, `vertexTex`, `edgeTex`, and `cellScalarTex` using the `TEX_WIDTH = 4096` stride.

2.  **Pass 1: Clear**:
    *   Bind `startOffsetTex` and `fragmentBufferTex` as `image2D` (read/write).
    *   Bind the atomic counter buffer.
    *   Render a full-screen triangle (or quad) to reset all values in `startOffsetTex` to `-1`.
    *   Disable depth testing and culling.

3.  **Pass 2: Gather**:
    *   Bind `startOffsetTex` and `fragmentBufferTex` as `image2D` (read/write).
    *   Bind `faceTex`, `vertexTex`, `edgeTex`, and `cellScalarTex` as standard textures (units 0-3).
    *   Set uniforms: MVP matrix, ModelViewInverse, camera position, sphere parameters, LOD values, scalar range, and FOV.
    *   Render `faceCount * 6` triangles (representing the hexahedral faces).
    *   In the fragment shader:
        *   Compute ray-sphere and ray-plane intersections to determine visibility.
        *   Use `atomicAdd` on the atomic counter to get a unique index for the new fragment.
        *   Write fragment data to `fragmentBufferTex` at the obtained index.
        *   Update `startOffsetTex` for the corresponding pixel to point to this new fragment (building the linked list).

4.  **Pass 3: Resolve**:
    *   Bind `startOffsetTex` and `fragmentBufferTex` as `image2D` (read-only).
    *   Render a full-screen triangle.
    *   In the fragment shader:
        *   Read the linked list of fragments for the current pixel from `fragmentBufferTex` using the start offset.
        *   Sort the fragments by depth (insertion sort if count $\le$ 32, otherwise other method).
        *   Perform alpha blending of the sorted fragments to produce the final color.

### Formulas

**Ray-Sphere Intersection** [S1, unofficial]:
Given ray origin $O$, direction $D$, sphere center $C$, and radius $R$:
$$ A = D \cdot D $$
$$ B = 2.0 \cdot D \cdot (O - C) $$
$$ C_{val} = (O - C) \cdot (O - C) - R^2 $$
$$ \Delta = B^2 - 4AC_{val} $$
If $\Delta < 0$, no intersection. Otherwise:
$$ t_0 = \frac{-B - \sqrt{\Delta}}{2A}, \quad t_1 = \frac{-B + \sqrt{\Delta}}{2A} $$
Intersection position: $P = O + t_0 D$.

**Ray-Plane Intersection** [S1, unofficial]:
Given ray origin $O$, direction $D$, plane point $P_{plane}$, and normal $N$:
$$ ln = N \cdot D $$
If $|ln| < 10^{-4}$, no intersection (parallel).
$$ t = \frac{N \cdot P_{plane} - N \cdot O}{ln} $$
Intersection position: $P = O + t D$.

### Parameters
*   `FRAG_BUFFER_W`: 2048
*   `FRAG_BUFFER_H`: 2048
*   `LINKED_LIST_SIZE`: 4,194,304 ($2048 \times 2048$)
*   `RESOLVE_CUTOFF`: 32 fragments per pixel (threshold for insertion sort)
*   `TEX_WIDTH`: 4096
*   `uMaxLodValue`: 1.0
*   `uSelectedLodValueFocus`: 0.4
*   `uSelectedLodValueContext`: 0.7
*   `uImportantLineBoostFactor`: 0.5

### Edge Cases and Pitfalls
*   **SwiftShader Compatibility**: SwiftShader's WebGL2 implementation does not support `image2D` or `atomic_uint`, returning `null`/`undefined` for `MAX_IMAGE_UNITS` [S1, unofficial].
*   **Infinite Recursion**: Shader include resolution must enforce a maximum depth (e.g., 10) to prevent infinite loops [S1, unofficial].
*   **Precision**: Plane intersection uses a threshold of $10^{-4}$ for parallel detection [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Shader Loader
def resolve_shader_includes(src, sources, depth=0):
    if depth > 10:
        raise Error("Include depth exceeded")
    lines = src.split('\n')
    for i, line in enumerate(lines):
        match = re.match(r'^\s*#include\s+"([^"]+)"\s*$', line)
        if match:
            name = match.group(1)
            if name not in sources:
                raise Error(f"Unknown shader include: {name}")
            lines[i] = resolve_shader_includes(sources[name], sources, depth + 1)
    return '\n'.join(lines)

# Buffer Building
def build_clearview_buffers(mesh):
    n_cells = mesh.n_cells
    face_count = 6 * n_cells
    vertex_count = 8 * n_cells
    edge_count = 12 * n_cells
    TEX_WIDTH = 4096
    
    # VTK to STD mapping
    VTK_TO_STD = [0, 1, 3, 2, 4, 5, 7, 6]
    
    # Vertex Buffer (RGBA32F)
    vertex_height = math.ceil(vertex_count / TEX_WIDTH)
    vertices_tex = Float32Array(vertex_height * TEX_WIDTH * 4)
    for c in range(n_cells):
        for k in range(8):
            std_corner = k
            vtk_corner = VTK_TO_STD[std_corner]
            pi = mesh.cell_conn[c * 8 + vtk_corner]
            linear_idx = c * 8 + std_corner
            col = linear_idx % TEX_WIDTH
            row = linear_idx // TEX_WIDTH
            ti = (row * TEX_WIDTH + col) * 4
            vertices_tex[ti] = mesh.positions[pi * 3]
            vertices_tex[ti+1] = mesh.positions[pi * 3 + 1]
            vertices_tex[ti+2] = mesh.positions[pi * 3 + 2]
            vertices_tex[ti+3] = mesh.scalars[pi]
            
    # Face, Edge, Cell Scalar buffers built similarly using HEX_FACES and HEX_EDGES
    # ... (omitted for brevity, follows same pattern)
    
    return {
        'faces': {'texels': face_tex, 'width': TEX_WIDTH, 'height': math.ceil(face_count / TEX_WIDTH)},
        'vertices': {'texels': vertices_tex, 'width': TEX_WIDTH, 'height': vertex_height},
        'edges': {'texels': edge_tex, 'width': TEX_WIDTH, 'height': math.ceil(edge_count / TEX_WIDTH)},
        'cell_scalar': {'texels': cell_scalar_tex, 'width': TEX_WIDTH, 'height': math.ceil(face_count / TEX_WIDTH)},
        'face_count': face_count,
        'vertex_count': vertex_count,
        'edge_count': edge_count
    }

# Ray Intersection Shaders (GLSL-like)
def ray_sphere_intersection(ray_origin, ray_direction, sphere_center, sphere_radius):
    A = dot(ray_direction, ray_direction)
    B = 2.0 * dot(ray_direction, ray_origin - sphere_center)
    C = dot(ray_origin - sphere_center, ray_origin - sphere_center) - sphere_radius**2
    discriminant = B**2 - 4.0 * A * C
    if discriminant < 0:
        return False, None, None
    sqrt_disc = sqrt(discriminant)
    t0 = (-B - sqrt_disc) / (2.0 * A)
    t1 = (-B + sqrt_disc) / (2.0 * A)
    intersection_pos = ray_origin + t0 * ray_direction
    return True, t0, intersection_pos

def ray_plane_intersection(ray_origin, ray_direction, plane_point, plane_normal):
    ln = dot(plane_normal, ray_direction)
    if abs(ln) < 1e-4:
        return False, None
    pos = dot(plane_normal, ray_origin) - dot(plane_normal, plane_point)
    t = -pos / ln
    intersection_pos = ray_origin + t * ray_direction
    return True, intersection_pos
```

## Subtopics

### WebSplatter
A splatting-based renderer for point clouds and volumetric data in WebGL. It differs from ClearView by using additive blending of splats rather than explicit geometry rendering and linked lists. Choose WebSplatter for point-based data; choose ClearView for explicit mesh wireframes and scalar fields [1741-websplatter.md].

### CPU-Driven Task Spawning
A method where the CPU manages rendering tasks and dispatches them to the GPU. It differs from ClearView's fully GPU-driven atomic PPLL approach. Choose CPU-Driven for complex culling logic or when GPU atomics are unsupported; choose ClearView for high-performance, self-contained GPU rendering [4397-cpu-driven-task-spawning.md].

### Pure client-side renderers
Renderers that operate entirely in the browser without server-side processing. ClearView is a specific implementation within this category, optimized for WebGL2 mesh rendering. Choose Pure client-side for general web visualization; choose ClearView specifically for hexahedral mesh wireframes/scalars [5040-pure-client-side-renderers.md].

## Operating characteristics
*   **Time Complexity**: The Gather pass is $O(F)$ where $F$ is the number of faces, dominated by atomic operations. The Resolve pass is $O(P \cdot K \log K)$ or $O(P \cdot K)$ where $P$ is the number of pixels and $K$ is the average number of fragments per pixel (capped at 32 for insertion sort).
*   **Memory Complexity**: $O(F + P \cdot K_{max})$. The fragment buffer is fixed at 4 million entries (64 MB for `RGBA32UI`). Mesh data scales with cell count.
*   **Measured Performance**: Tuned for SPE-9 mesh (54,000 faces). The 2048x2048 buffer supports ~30-60 depth complexity per pixel efficiently [S1, unofficial].
*   **Strengths**: Fully GPU-accelerated, no CPU-GPU synchronization during rendering, works in WebGL2 without extensions.
*   **Limitations**: Fixed fragment buffer size limits maximum depth complexity. Not compatible with SwiftShader. Requires careful texture size management due to WebGL2 limitations.

## Background
ClearView (Unified) is an implementation of a 3-pass WebGL2 renderer for hexahedral meshes, adapted from a C++ desktop renderer (ClearViewRenderer_FacesUnified). It was developed as part of the "geresim-spike" project to enable high-performance volumetric mesh visualization in web browsers, overcoming WebGL2's lack of SSBOs by mapping data to image textures [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
