# ClearView (Unified) Renderer

Also called: ClearView, ClearView Unified Buffer, ClearView WebGL2 Spike

## Overview
ClearView (Unified) is a 3-pass WebGL2 renderer designed for visualizing unstructured hexahedral meshes, specifically wireframes and scalar fields, by adapting GPU-accelerated parallel prefix list (PPLL) techniques to browser constraints [S1, unofficial]. It solves the problem of rendering complex volumetric meshes in WebGL2, which lacks Shader Storage Buffer Objects (SSBOs), by encoding mesh data into `image2D` textures and using atomic operations to manage fragment accumulation [S1, unofficial]. Unlike [WebSplatter](1741-websplatter.md) which focuses on splatting, ClearView explicitly handles hexahedral topology via a gather-resolve pipeline that supports depth-sorted alpha blending without CPU-side sorting [S1, unofficial].

## How it works
The renderer operates in three distinct passes: Clear, Gather, and Resolve. It maps hexahedral mesh data into fixed-width textures to bypass WebGL2 SSBO limitations, then uses atomic counters to build a parallel prefix list of fragments for sorting and blending.

### Inputs and Outputs
**Inputs:**
*   Hexahedral mesh defined by:
    *   `positions`: `Float32Array` of vertex coordinates $(x, y, z)$.
    *   `cellConn`: `Uint32Array` of vertex indices per cell.
    *   `scalars`: `Float32Array` of per-vertex scalar values.
    *   `nPoints`: Number of vertices.
    *   `nCells`: Number of hexahedral cells.
*   Camera parameters: Model-View-Projection (MVP) matrix, Model-View Inverse matrix, camera position, look direction, field of view (FOV).
*   Rendering parameters: Sphere center and radius (for ray intersection), LOD values, colors, scalar range.

**Outputs:**
*   Final color buffer (framebuffer attachment) containing the rendered wireframe and scalar field.

### Data Structures and Layouts
Due to WebGL2 limitations, all mesh data is stored in 2D `image2D` textures with a fixed row width `TEX_WIDTH = 4096` [S1, unofficial].

1.  **Vertex Texture (`vertexTex`)**:
    *   Format: `RGBA32F` (4 floats per texel).
    *   Layout: 1 texel per vertex slot. Each texel stores $(x, y, z, \text{scalar})$.
    *   Dimensions: Width = 4096, Height = $\lceil (8 \times n_{\text{cells}}) / 4096 \rceil$.
    *   Indexing: For cell $c$ and STD corner $k$ ($0..7$), the linear index is $c \times 8 + k$. The VTK corner index is $VTK\_TO\_STD[k]$. The vertex position is `mesh.positions[mesh.cellConn[c * 8 + VTK_TO_STD[k]] * 3 + xyz]`.

2.  **Face Texture (`faceTex`)**:
    *   Format: `RG32UI` (2 unsigned ints per texel).
    *   Layout: 2 texels per face. Stores face-local edge indices or corner references.
    *   Dimensions: Width = 4096, Height = $\lceil (6 \times n_{\text{cells}} \times 2) / 4096 \rceil$.

3.  **Edge Texture (`edgeTex`)**:
    *   Format: `RG32F` (2 floats per texel).
    *   Layout: 1 texel per edge.
    *   Dimensions: Width = 4096, Height = $\lceil (12 \times n_{\text{cells}}) / 4096 \rceil$.

4.  **Cell Scalar Texture (`cellScalarTex`)**:
    *   Format: `R32F` (1 float per texel).
    *   Layout: 1 texel per face (for face-based scalar interpolation).
    *   Dimensions: Width = 4096, Height = $\lceil (6 \times n_{\text{cells}}) / 4096 \rceil$.

5.  **Start Offset Texture (`startOffsetTex`)**:
    *   Format: `R32UI`.
    *   Layout: 1 texel per pixel in the viewport. Stores the starting index in the fragment buffer for each pixel's linked list.
    *   Dimensions: Matches viewport width and height.

6.  **Fragment Buffer Texture (`fragmentBufferTex`)**:
    *   Format: `RGBA32UI`.
    *   Layout: Stores fragment data (e.g., depth, color, normal).
    *   Dimensions: Fixed at `FRAG_BUFFER_W = 2048`, `FRAG_BUFFER_H = 2048` (4 million entries) [S1, unofficial].

7.  **Atomic Counter Buffer**:
    *   Format: `Uint32`.
    *   Usage: Tracks the total number of fragments written to `fragmentBufferTex`. Bound to binding point 0.

**Hexahedral Topology (STD Layout):**
*   **Corner Indices (STD):** 0=c000, 1=c100, 2=c010, 3=c110, 4=c001, 5=c101, 6=c011, 7=c111.
*   **VTK to STD Mapping:** `[0, 1, 3, 2, 4, 5, 7, 6]` [S1, unofficial].
*   **Edges:** 12 per cell, defined as pairs of STD indices:
    *   Bottom (z=0): [0,1], [1,3], [2,3], [0,2]
    *   Top (z=1): [4,5], [5,7], [6,7], [4,6]
    *   Vertical: [0,4], [1,5], [3,7], [2,6]
*   **Faces:** 6 per cell. Each face has 4 corners in face-local CCW order (viewed from outside) and 4 edge indices. Example: -z face corners `[0, 2, 3, 1]`, edges `[3, 2, 1, 0]` [S1, unofficial].

### Numbered Steps

**Preprocessing: Buffer Construction**
1.  Calculate `faceCount = 6 * nCells`, `vertexCount = 8 * nCells`, `edgeCount = 12 * nCells`.
2.  Allocate `vertexTex` texels. For each cell $c$ and STD corner $k$:
    *   Get VTK corner index: $v_{idx} = mesh.cellConn[c \times 8 + VTK\_TO\_STD[k]]$.
    *   Get position: $p = mesh.positions[v_{idx} \times 3]$.
    *   Get scalar: $s = mesh.scalars[v_{idx}]$.
    *   Write to `vertexTex` at linear index $c \times 8 + k$ as $(p_x, p_y, p_z, s)$.
3.  Allocate `faceTex`, `edgeTex`, `cellScalarTex` similarly based on topology definitions.

**Pass 1: Clear**
1.  Bind `startOffsetTex` as `image2D` unit 0 (READ_WRITE, `R32UI`) and `fragmentBufferTex` as unit 1 (READ_WRITE, `RGBA32UI`).
2.  Bind atomic counter buffer to binding 0.
3.  Set uniforms: `uStartOffsetTex` (0), `uFragmentBufferTex` (1), `uViewportW`, `uViewportH`.
4.  Draw a full-screen triangle (3 vertices) using the Clear shader.
5.  The shader resets every texel in `startOffsetTex` to -1.

**Pass 2: Gather**
1.  Bind `startOffsetTex` (unit 0, READ_WRITE, `R32UI`) and `fragmentBufferTex` (unit 1, READ_WRITE, `RGBA32UI`).
2.  Bind data textures: `faceTex` (unit 0), `vertexTex` (unit 1), `edgeTex` (unit 2), `cellScalarTex` (unit 3).
3.  Set uniforms: Camera matrices (MVP, ModelViewInverse), camera position, look direction, sphere parameters, LOD values, colors, scalar range, FOV, `uLinkedListSize` ($2048 \times 2048$).
4.  Draw `faceCount * 6` triangles (representing the hexahedral faces).
5.  For each fragment generated:
    *   Compute ray-sphere intersection to determine visibility/depth.
    *   Use `atomicAdd` on the atomic counter to get a unique index in `fragmentBufferTex`.
    *   Write fragment data (depth, color, etc.) to `fragmentBufferTex` at the obtained index.
    *   Update `startOffsetTex` for the corresponding pixel to maintain the linked list structure (PPLL).

**Pass 3: Resolve**
1.  Bind `startOffsetTex` (unit 0, READ_ONLY, `R32UI`) and `fragmentBufferTex` (unit 1, READ_ONLY, `RGBA32UI`).
2.  Set uniforms: `uStartOffsetTex` (0), `uFragmentBufferTex` (1), `uViewportW`, `uViewportH`, `uLinkedListSize`.
3.  Draw a full-screen triangle.
4.  For each pixel:
    *   Read the start offset from `startOffsetTex`.
    *   Walk the linked list in `fragmentBufferTex` using the stored next pointers.
    *   Sort fragments by depth (insertion sort if list size $\le 32$, otherwise external sort or optimized walk) [S1, unofficial].
    *   Perform alpha blending of sorted fragments to produce the final pixel color.

### Formulas

**Ray-Sphere Intersection:**
Given ray origin $O$, direction $D$, sphere center $C$, and radius $R$:
$$ A = D_x^2 + D_y^2 + D_z^2 $$
$$ B = 2.0 \cdot (D_x(O_x - C_x) + D_y(O_y - C_y) + D_z(O_z - C_z)) $$
$$ C_{val} = (O_x - C_x)^2 + (O_y - C_y)^2 + (O_z - C_z)^2 - R^2 $$
$$ \text{discriminant} = B^2 - 4AC_{val} $$
If discriminant $< 0$, no intersection. Otherwise:
$$ t_0 = \frac{-B - \sqrt{\text{discriminant}}}{2A}, \quad t_1 = \frac{-B + \sqrt{\text{discriminant}}}{2A} $$
Intersection position: $P = O + t_0 D$ [S1, unofficial].

**Ray-Plane Intersection:**
Given ray $O, D$, plane point $P_{plane}$, normal $N$:
$$ \ln = N \cdot D $$
If $|\ln| < 10^{-4}$, no intersection (parallel).
$$ t = \frac{N \cdot P_{plane} - N \cdot O}{\ln} $$
Intersection position: $P = O + t D$ [S1, unofficial].

### Parameters
*   `TEX_WIDTH`: 4096 (fixed texture row width) [S1, unofficial].
*   `FRAG_BUFFER_W`: 2048 [S1, unofficial].
*   `FRAG_BUFFER_H`: 2048 [S1, unofficial].
*   `LINKED_LIST_SIZE`: $2048 \times 2048 = 4,194,304$ [S1, unofficial].
*   `Resolve Cutoff`: 32 fragments per pixel for insertion sort optimization [S1, unofficial].
*   `uMaxLodValue`: 1.0 [S1, unofficial].
*   `uSelectedLodValueFocus`: 0.4 [S1, unofficial].
*   `uSelectedLodValueContext`: 0.7 [S1, unofficial].
*   `uImportantLineBoostFactor`: 0.5 [S1, unofficial].

### Edge Cases and Pitfalls
*   **SwiftShader Compatibility:** SwiftShader's WebGL2 implementation does not support `image2D` or `atomic_uint`, returning `null`/`undefined` for `MAX_IMAGE_UNITS` [S1, unofficial].
*   **WebGL2 SSBO Limitation:** WebGL2 lacks SSBOs, necessitating the texture-based data storage approach with explicit `gl.bindImageTexture` calls [S1, unofficial].
*   **Shader Include Recursion:** Shader loaders must prevent infinite recursion from `#include` directives. A maximum depth of 10 is enforced, throwing an error if exceeded [S1, unofficial].
*   **Precision:** Use `R32UI` for offsets and `RGBA32UI` for fragment data to ensure sufficient precision for large fragment counts.

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
    VTK_TO_STD = [0, 1, 3, 2, 4, 5, 7, 6]

    # Vertex Buffer (RGBA32F)
    vertex_height = math.ceil(vertex_count / TEX_WIDTH)
    vertices_tex = np.zeros((vertex_height * TEX_WIDTH * 4), dtype=np.float32)
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
            vertices_tex[ti + 1] = mesh.positions[pi * 3 + 1]
            vertices_tex[ti + 2] = mesh.positions[pi * 3 + 2]
            vertices_tex[ti + 3] = mesh.scalars[pi]

    # Face, Edge, CellScalar buffers similarly constructed...
    return {
        'faces': {'texels': face_tex, 'width': TEX_WIDTH, 'height': face_height},
        'vertices': {'texels': vertices_tex, 'width': TEX_WIDTH, 'height': vertex_height},
        'edges': {'texels': edge_tex, 'width': TEX_WIDTH, 'height': edge_height},
        'cell_scalar': {'texels': cell_scalar_tex, 'width': TEX_WIDTH, 'height': cell_scalar_height},
        'face_count': face_count,
        'vertex_count': vertex_count,
        'edge_count': edge_count
    }

# Ray Intersection Helpers
def ray_sphere_intersection(ray_origin, ray_direction, sphere_center, sphere_radius):
    A = dot(ray_direction, ray_direction)
    B = 2.0 * dot(ray_direction, ray_origin - sphere_center)
    C = dot(ray_origin - sphere_center, ray_origin - sphere_center) - sphere_radius**2
    discriminant = B**2 - 4 * A * C
    if discriminant < 0:
        return False, None, None
    sqrt_disc = math.sqrt(discriminant)
    t0 = (-B - sqrt_disc) / (2 * A)
    t1 = (-B + sqrt_disc) / (2 * A)
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
A splatting-based renderer for volumetric data, optimized for WebGL2. It differs from ClearView by using depth-sorted splats rather than explicit mesh face rendering. Choose when rendering dense scalar fields or point clouds rather than explicit wireframes. [WebSplatter](1741-websplatter.md)

### CPU-Driven Task Spawning
A method where the CPU manages rendering tasks and dispatches them to the GPU. It differs from ClearView's fully GPU-driven PPLL approach. Choose when CPU-GPU synchronization overhead is acceptable or when task dependencies are complex. [CPU-Driven Task Spawning](4397-cpu-driven-task-spawning.md)

### Pure client-side renderers
Renderers that operate entirely within the browser without server-side processing. ClearView is a specific implementation within this category. [Pure client-side renderers](5040-pure-client-side-renderers.md)

### Single Node Multi-GPU/Server Architecture
Architectures leveraging multiple GPUs or server-side rendering. ClearView is limited to single-GPU WebGL2 contexts. [Single Node Multi-GPU/Server Architecture](5165-single-node-multi-gpu-server-architecture.md)

## Operating characteristics
*   **Time Complexity**: The Gather pass is $O(F)$ where $F$ is the number of faces, dominated by atomic operations. The Resolve pass is $O(P \cdot L)$ where $P$ is the number of pixels and $L$ is the average number of fragments per pixel. With the cutoff of 32, sorting is $O(1)$ per pixel for small lists.
*   **Memory Complexity**: $O(V + F + E + P \cdot L_{max})$ where $V, F, E$ are vertex, face, edge counts, and $L_{max}$ is the max fragments per pixel (4M entries in `fragmentBufferTex`).
*   **Measured Performance**: Tuned for SPE-9 mesh (54,000 faces). Fragment buffer size is 64 MB (2048x2048 RGBA32UI) [S1, unofficial].
*   **Strengths**: Fully GPU-accelerated, no CPU-side sorting required, works within WebGL2 constraints.
*   **Limitations**: Limited by WebGL2 features (no SSBOs), performance degrades with very high fragment density per pixel, not compatible with SwiftShader.
*   **When to use**: WebGL2 applications requiring high-quality wireframe and scalar field visualization of hexahedral meshes.
*   **When not to use**: When SSBOs are available (use more efficient buffer formats), or when rendering non-hexahedral meshes without adaptation.

## Background
ClearView (Unified) was developed as part of the `geresim-spike` project, a WebGL2 spike for hexahedral mesh visualization [S1, unofficial]. It adapts techniques from desktop GPU renderers (like the C++ `ClearViewRenderer_FacesUnified`) to the browser environment, specifically addressing the lack of SSBOs in WebGL2 by using `image2D` textures and atomic counters [S1, unofficial]. The method is documented in the project's source code and associated plan files (PLAN.md, docs/project-recon.md) [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
