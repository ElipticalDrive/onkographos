# ClearView (Unified) Renderer

Also called: ClearView, ClearView Unified Buffer, ClearView WebGL2 Spike

## Overview
ClearView (Unified) is a 3-pass WebGL2 renderer designed for unstructured hexahedral mesh visualization, specifically targeting wireframes and scalar fields [S1, unofficial]. It solves the problem of rendering complex volumetric meshes in the browser by replacing traditional CPU-side sorting or SSBO-based GPU buffers with a Parallel Prefix List (PPLL) structure stored in image2D textures [S1, unofficial]. Unlike [WebSplatter](1741-websplatter.md), which uses splatting techniques, ClearView explicitly rasterizes per-face triangles and manages fragment ordering via atomic operations and a linked-list-like buffer, enabling correct alpha blending without depth-sorting artifacts [S1, unofficial].

## How it works
The renderer operates in three distinct passes: Clear, Gather, and Resolve [S1, unofficial]. It relies on WebGL2 features (`image2D`, `atomic_uint`) which are unavailable in software renderers like SwiftShader [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Mesh Data:** Hexahedral mesh defined by vertices, cells, and connectivity [S1, unofficial].
    *   **Vertices:** 8 per cell, stored in Standard (STD) layout [S1, unofficial].
    *   **Faces:** 6 per cell [S1, unofficial].
    *   **Edges:** 12 per cell [S1, unofficial].
    *   **Scalars:** Per-vertex or per-cell scalar values [S1, unofficial].
*   **Camera:** View/Projection matrices, camera position, and field of view [S1, unofficial].
*   **Sphere Parameters:** Center and radius for ray-sphere intersection tests (used for LOD/culling) [S1, unofficial].

**Outputs:**
*   **Final Image:** Alpha-blended 2D projection of the 3D mesh [S1, unofficial].

### Data Structures and Layouts
Due to WebGL2 lacking Shader Storage Buffer Objects (SSBOs), all mesh data is encoded into 2D textures with a fixed row width of `TEX_WIDTH = 4096` [S1, unofficial].

1.  **Face Texture (`faceTex`):**
    *   Format: `RG32UI` (2 unsigned 32-bit integers per texel).
    *   Layout: 2 texels per face. Stores face connectivity indices.
    *   Dimensions: `width = 4096`, `height = ceil(faceCount * 2 / 4096)` [S1, unofficial].
2.  **Vertex Texture (`vertexTex`):**
    *   Format: `RGBA32F` (4 floats per texel).
    *   Layout: 1 texel per vertex slot. Stores `(x, y, z, scalar_attribute)`.
    *   Mapping: VTK corner indices are mapped to STD indices via `VTK_TO_STD = [0, 1, 3, 2, 4, 5, 7, 6]` [S1, unofficial].
    *   Dimensions: `width = 4096`, `height = ceil(vertexCount / 4096)` [S1, unofficial].
3.  **Edge Texture (`edgeTex`):**
    *   Format: `RG32F` (2 floats per texel).
    *   Layout: 1 texel per edge. Stores edge indices or geometric data.
    *   Dimensions: `width = 4096`, `height = ceil(edgeCount / 4096)` [S1, unofficial].
4.  **Cell Scalar Texture (`cellScalarTex`):**
    *   Format: `R32F` (1 float per texel).
    *   Layout: 1 texel per face/cell.
    *   Dimensions: `width = 4096`, `height = ceil(faceCount / 4096)` [S1, unofficial].
5.  **Start Offset Texture (`startOffsetTex`):**
    *   Format: `R32UI` (unsigned 32-bit integer).
    *   Layout: 1 texel per pixel in the viewport. Stores the starting index in the fragment buffer for each pixel's linked list [S1, unofficial].
    *   Dimensions: Matches viewport width and height [S1, unofficial].
6.  **Fragment Buffer Texture (`fragmentBufferTex`):**
    *   Format: `RGBA32UI` (4 unsigned 32-bit integers per texel).
    *   Layout: 1 texel per fragment entry. Stores fragment data (e.g., depth, color, next pointer) [S1, unofficial].
    *   Dimensions: Fixed at `2048 x 2048` (4 million entries) [S1, unofficial].
7.  **Atomic Counter Buffer:**
    *   Format: `Uint32`.
    *   Usage: Tracks the total number of fragments written to `fragmentBufferTex`. Bound to binding point 0 [S1, unofficial].

### Hexahedral Topology
The renderer uses a specific Standard (STD) corner indexing for hexahedra:
*   0=c000, 1=c100, 2=c010, 3=c110, 4=c001, 5=c101, 6=c011, 7=c111 [S1, unofficial].
*   **Edges:** 12 per cell, defined as pairs of STD indices (e.g., `[0,1]`, `[1,3]`, etc.) [S1, unofficial].
*   **Faces:** 6 per cell. Each face lists 4 STD corner indices in face-local counter-clockwise (CCW) order (viewed from outside) and 4 edge indices. The ordering ensures outward-facing normals via cross product [S1, unofficial].

### Parameters
*   `FRAG_BUFFER_W`: 2048 [S1, unofficial].
*   `FRAG_BUFFER_H`: 2048 [S1, unofficial].
*   `LINKED_LIST_SIZE`: `FRAG_BUFFER_W * FRAG_BUFFER_H` (4,194,304) [S1, unofficial].
*   `RESOLVE_CUTOFF`: 32 fragments per pixel. If a pixel has fewer than 32 fragments, an insertion sort is used during resolution; otherwise, a more complex sort or walk is performed [S1, unofficial].
*   `TEX_WIDTH`: 4096 [S1, unofficial].
*   **LOD/Coloring Uniforms:**
    *   `uMaxLodValue`: 1.0
    *   `uSelectedLodValueFocus`: 0.4
    *   `uSelectedLodValueContext`: 0.7
    *   `uImportantLineBoostFactor`: 0.5 [S1, unofficial].

### Steps

#### Pass 1: Clear
1.  Bind `startOffsetTex` (unit 0, `R32UI`, `READ_WRITE`) and `fragmentBufferTex` (unit 1, `RGBA32UI`, `READ_WRITE`) as image2D textures [S1, unofficial].
2.  Bind the atomic counter buffer to binding point 0 [S1, unofficial].
3.  Set viewport to current canvas size [S1, unofficial].
4.  Disable depth testing and face culling [S1, unofficial].
5.  Draw a full-screen triangle (3 vertices) using the Clear shader [S1, unofficial].
6.  The shader resets all values in `startOffsetTex` to `-1`, initializing the linked lists for each pixel [S1, unofficial].

#### Pass 2: Gather
1.  Bind data textures: `faceTex` (unit 0), `vertexTex` (unit 1), `edgeTex` (unit 2), `cellScalarTex` (unit 3) [S1, unofficial].
2.  Bind `startOffsetTex` (unit 0, `R32UI`, `READ_WRITE`) and `fragmentBufferTex` (unit 1, `RGBA32UI`, `READ_WRITE`) as image2D textures [S1, unofficial].
3.  Set camera uniforms: MVP matrix, ModelViewInverse, camera position, look direction, sphere center/radius, LOD values, colors, scalar range, and FOV [S1, unofficial].
4.  Draw `faceCount * 6` triangles (representing the faces of the hexahedral mesh) [S1, unofficial].
5.  For each fragment generated:
    *   Perform ray-sphere intersection to determine visibility/LOD [S1, unofficial].
    *   Calculate fragment attributes (depth, color, etc.) [S1, unofficial].
    *   Use an atomic counter to obtain a unique index in `fragmentBufferTex` [S1, unofficial].
    *   Write fragment data to `fragmentBufferTex` at the obtained index [S1, unofficial].
    *   Atomically update the `startOffsetTex` for the corresponding pixel to point to this new fragment, forming a linked list [S1, unofficial].

#### Pass 3: Resolve
1.  Bind `startOffsetTex` (unit 0, `R32UI`, `READ_ONLY`) and `fragmentBufferTex` (unit 1, `RGBA32UI`, `READ_ONLY`) as image2D textures [S1, unofficial].
2.  Set viewport and `LINKED_LIST_SIZE` [S1, unofficial].
3.  Draw a full-screen triangle (3 vertices) using the Resolve shader [S1, unofficial].
4.  For each pixel:
    *   Read the starting index from `startOffsetTex` [S1, unofficial].
    *   Walk the linked list in `fragmentBufferTex` using the next-pointer stored in each fragment [S1, unofficial].
    *   Sort fragments by depth (if necessary, though the gather order often approximates back-to-front) [S1, unofficial].
    *   Perform alpha blending of the sorted fragments to produce the final color [S1, unofficial].
    *   Write the result to the framebuffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import re

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def ray_sphere_intersection(ray_origin, ray_direction, sphere_center, sphere_radius):
    A = dot(ray_direction, ray_direction)
    B = 2.0 * dot(ray_direction, [ray_origin[i] - sphere_center[i] for i in range(3)])
    C = dot([ray_origin[i] - sphere_center[i] for i in range(3)], [ray_origin[i] - sphere_center[i] for i in range(3)]) - sphere_radius ** 2
    discriminant = B ** 2 - 4.0 * A * C
    if discriminant < 0.0:
        return False
    sqrt_disc = math.sqrt(discriminant)
    t0 = (-B - sqrt_disc) / (2.0 * A)
    t1 = (-B + sqrt_disc) / (2.0 * A)
    intersection_position = [ray_origin[i] + t0 * ray_direction[i] for i in range(3)]
    return (t0, t1, intersection_position)

def ray_plane_intersection(ray_origin, ray_direction, plane_point, plane_normal):
    ln = dot(plane_normal, ray_direction)
    if abs(ln) < 1e-4:
        return False
    pos = dot(plane_normal, ray_origin) - dot(plane_normal, plane_point)
    t = -pos / ln
    intersection_position = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
    return intersection_position

def resolve_shader_source(src, sources, depth=0):
    if depth > 10:
        raise ValueError("Include depth exceeded")
    lines = src.split('\n')
    changed = False
    for i, line in enumerate(lines):
        match = re.match(r'^\s*#include\s+"([^"]+)"\s*$', line)
        if match:
            name = match.group(1)
            if name not in sources:
                raise ValueError(f"Unknown shader include: {name}")
            lines[i] = resolve_shader_source(sources[name], sources, depth + 1)
            changed = True
    return '\n'.join(lines) if changed else src

def build_clear_view_buffers(mesh):
    # Returns structured data for faces, vertices, edges, cell_scalar textures
    pass

def create_image_texture(width, height, format):
    # Returns a 2D texture object
    pass

def load_shaders(vs, fs):
    # Returns a shader program object
    pass

def bind_image_texture(unit, tex, format, access):
    pass

def bind_buffer_base(target, binding, buf):
    pass

def set_uniform1i(prog, name, val):
    pass

def set_uniform_matrix4fv(prog, name, mat):
    pass

def set_uniform3fv(prog, name, vec):
    pass

def set_uniform1f(prog, name, val):
    pass

def disable(cap):
    pass

def draw_arrays(mode, first, count):
    pass

class ClearViewRenderer:
    def __init__(self, mesh):
        self.mesh = mesh
        self.FRAG_BUFFER_W = 2048
        self.FRAG_BUFFER_H = 2048
        self.TEX_WIDTH = 4096
        self.buffers = build_clear_view_buffers(mesh)
        self.start_offset_tex = create_image_texture(self.viewport_w, self.viewport_h, format="R32UI")
        self.fragment_buffer_tex = create_image_texture(self.FRAG_BUFFER_W, self.FRAG_BUFFER_H, format="RGBA32UI")
        self.atomic_counter_buf = create_buffer(data=[0], usage="DYNAMIC_COPY")
        self.clear_prog = load_shaders('clear.vert', 'clear.frag')
        self.gather_prog = load_shaders('gather.vert', 'gather.frag')
        self.resolve_prog = load_shaders('resolve.vert', 'resolve.frag')

    def render(self, camera):
        w, h = self.viewport_w, self.viewport_h
        
        # Pass 1: Clear
        bind_image_texture(0, self.start_offset_tex, format="R32UI", access="READ_WRITE")
        bind_image_texture(1, self.fragment_buffer_tex, format="RGBA32UI", access="READ_WRITE")
        bind_buffer_base("ATOMIC_COUNTER_BUFFER", 0, self.atomic_counter_buf)
        set_uniform1i(self.clear_prog, 'uStartOffsetTex', 0)
        set_uniform1i(self.clear_prog, 'uFragmentBufferTex', 1)
        set_uniform1i(self.clear_prog, 'uViewportW', w)
        set_uniform1i(self.clear_prog, 'uViewportH', h)
        disable(DEPTH_TEST)
        disable(CULL_FACE)
        draw_arrays(TRIANGLES, 0, 3)

        # Pass 2: Gather
        bind_image_texture(0, self.start_offset_tex, format="R32UI", access="READ_WRITE")
        bind_image_texture(1, self.fragment_buffer_tex, format="RGBA32UI", access="READ_WRITE")
        bind_buffer_base("ATOMIC_COUNTER_BUFFER", 0, self.atomic_counter_buf)
        
        active_texture(0); bind_texture(2D, self.buffers.faces.tex)
        active_texture(1); bind_texture(2D, self.buffers.vertices.tex)
        active_texture(2); bind_texture(2D, self.buffers.edges.tex)
        active_texture(3); bind_texture(2D, self.buffers.cell_scalar.tex)
        
        set_uniform1i(self.gather_prog, 'uFaceTex', 0)
        set_uniform1i(self.gather_prog, 'uVertexTex', 1)
        set_uniform1i(self.gather_prog, 'uEdgeTex', 2)
        set_uniform1i(self.gather_prog, 'uCellScalarTex', 3)
        set_uniform1i(self.gather_prog, 'uStartOffsetTex', 0)
        set_uniform1i(self.gather_prog, 'uFragmentBufferTex', 1)
        set_uniform1i(self.gather_prog, 'uViewportW', w)
        set_uniform1i(self.gather_prog, 'uViewportH', h)
        set_uniform1i(self.gather_prog, 'uLinkedListSize', self.FRAG_BUFFER_W * self.FRAG_BUFFER_H)
        
        set_uniform_matrix4fv(self.gather_prog, 'uMVP', camera.mvp)
        set_uniform_matrix4fv(self.gather_prog, 'uModelViewInverse', camera.model_view_inverse)
        set_uniform3fv(self.gather_prog, 'uCameraPos', camera.position)
        set_uniform1f(self.gather_prog, 'uSphereRadius', self.sphere_radius)
        set_uniform3fv(self.gather_prog, 'uSphereCenter', self.sphere_center)
        
        draw_arrays(TRIANGLES, 0, self.buffers.face_count * 6)

        # Pass 3: Resolve
        bind_image_texture(0, self.start_offset_tex, format="R32UI", access="READ_ONLY")
        bind_image_texture(1, self.fragment_buffer_tex, format="RGBA32UI", access="READ_ONLY")
        
        set_uniform1i(self.resolve_prog, 'uStartOffsetTex', 0)
        set_uniform1i(self.resolve_prog, 'uFragmentBufferTex', 1)
        set_uniform1i(self.resolve_prog, 'uLinkedListSize', self.FRAG_BUFFER_W * self.FRAG_BUFFER_H)
        
        draw_arrays(TRIANGLES, 0, 3)
```

## Operating characteristics
*   **Time Complexity:** The Gather pass is $O(F)$ where $F$ is the number of faces, as each face is rasterized once. The Resolve pass is $O(P \cdot D)$ where $P$ is the number of pixels and $D$ is the average depth (number of fragments per pixel). Sorting is performed per-pixel, with a cutoff of 32 fragments using insertion sort [S1, unofficial].
*   **Memory Complexity:** The fragment buffer is fixed at $2048 \times 2048$ entries (4 million), consuming approximately 64 MB at `RGBA32UI` [S1, unofficial]. Data textures scale with mesh size but are bounded by the `TEX_WIDTH = 4096` constraint.
*   **Performance:** Tuned for scenes like SPE-9 (54,000 faces) with typical depths of 30-60 fragments per pixel [S1, unofficial].
*   **Strengths:** Fully client-side, no server-side sorting required. Works within WebGL2 constraints by avoiding SSBOs.
*   **Limitations:** Requires hardware support for `image2D` and `atomic_uint`. Does not work on SwiftShader or other software WebGL2 implementations that lack these features [S1, unofficial]. Fixed fragment buffer size may limit very deep scenes.

## Background
ClearView (Unified) is a WebGL2 adaptation of a GPU-accelerated visualization method for hexahedral meshes [S1, unofficial]. It addresses the limitations of WebGL2 by mapping SSBO-based data structures to image2D textures, enabling parallel fragment collection and resolution without CPU intervention [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
