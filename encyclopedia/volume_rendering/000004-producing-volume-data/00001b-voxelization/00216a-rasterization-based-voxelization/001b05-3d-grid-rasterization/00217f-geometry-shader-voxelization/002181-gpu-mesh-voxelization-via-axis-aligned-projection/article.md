# GPU Mesh Voxelization via Axis-Aligned Projection

Also called: Axis-Aligned Projection, Axis-aligned voxelization, Dominant Axis Projection, Dominant Axis Selection, Dominant Axis Voxelization, Single-Pass Voxelization via Axis Rotation, Voxelization with Dominant Axis Projection

## Overview
GPU Mesh Voxelization via Axis-Aligned Projection is a rasterization-based technique that converts polygonal triangle meshes into a 3D voxel grid by projecting each triangle onto a 2D plane aligned with its dominant normal axis [S1, S4]. It solves the problem of efficiently filling a 3D occupancy or color grid using standard GPU hardware rasterizers, which are optimized for 2D screens [S4]. Unlike [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) which may use generic 3D rasterization logic, this method explicitly selects the optimal projection axis per triangle to maximize fragment coverage and minimize holes [S3]. It differs from [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) by relying on hardware rasterization rather than analytical intersection tests, and from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by typically using a single draw call with axis-specific view matrices rather than multiple passes or pure atomic accumulation without depth-based projection [S4].

## How it works
**Inputs and Outputs:**
*   **Input:** Triangle mesh (vertices, indices), voxel grid dimensions ($W \times H \times D$), world-to-voxel transformation parameters [S4, S5].
*   **Output:** 3D voxel grid (either `RWTexture3D` or `RWStructuredBuffer`) containing occupancy flags, positions, and colors [S1, S2, S5].

**Data Structures:**
1.  **Constant Buffer (`cbuffer`):** Matrices: `World`, `ViewX`, `ViewY`, `ViewZ`, `Projection` [S2]. Integers: `width`, `height`, `depth`, `bcount` (flag for depth/occlusion handling) [S2].
2.  **Voxel Buffer (`u1`):** `RWTexture3D<float4> voxels` [S2] or `RWStructuredBuffer<Voxel> voxels` where `struct Voxel { int3 pos; int4 color; }` [S2].
3.  **Pixel Shader Input (`PS_INPUT`):** `float4 pos : SV_POSITION` [S2], `unsigned int axis : COLOR1` (Dominant axis index: 0=X, 1=Y, 2=Z) [S2], `float4 color : COLOR0` (Optional vertex/texture color) [S2], `float2 uv : TEXCOORD0` (Optional) [S2].

**Steps:**

1.  **CPU Setup:** Compute the voxel grid position, extents, and three view-projection matrices: $VP_X$, $VP_Y$, and $VP_Z$. These matrices project 3D world coordinates onto the 2D planes perpendicular to the X, Y, and Z axes respectively, mapping them into the voxel grid space [S5]. Configure the GPU pipeline: disable depth testing and face culling to ensure all triangles are processed regardless of occlusion or orientation [S4, S5]. Disable color writes to the default render target if using a structured buffer output [S5].

2.  **Geometry Shader (Dominant Axis Selection):** For each input triangle, calculate the face normal $\mathbf{n}$ using the cross product of edge vectors: $\mathbf{n} = (\mathbf{v}_2 - \mathbf{v}_1) \times (\mathbf{v}_3 - \mathbf{v}_1)$ [S1]. Determine the dominant axis index $ind$ by finding the component with the maximum absolute value: $\text{maxComponent} = \max(|n_x|, |n_y|, |n_z|)$ and $ind = 0$ if $|n_x| = \text{maxComponent}$, $1$ if $|n_y| = \text{maxComponent}$, else $2$ [S1, S3]. Select the corresponding view matrix ($View_X$, $View_Y$, or $View_Z$) [S2]. Transform the triangle vertices using the World matrix and the selected View matrix [S2, S5].

3.  **Reprojection:** Reorder the components of the clip-space position to project along the dominant axis. This ensures the rasterizer generates fragments for the most visible projection of the triangle, maximizing coverage [S1, S3]:
    *   If $ind == 0$ (X-axis dominant): `gl_Position = vec4(gl_in[i].gl_Position.zyx, 1.0)`
    *   If $ind == 1$ (Y-axis dominant): `gl_Position = vec4(gl_in[i].gl_Position.xzy, 1.0)`
    *   If $ind == 2$ (Z-axis dominant): `gl_Position = vec4(gl_in[i].gl_Position.xyz, 1.0)` [S3].
    Emit the vertex using `EmitVertex()` [S3].

4.  **Pixel Shader (Voxel Writing):** Receive the projected fragment, the dominant axis index, and color data [S2]. Apply conservative rasterization (e.g., "bloated" vertices or `NV_conservative_raster` extension) to guarantee all partially covered voxels are filled and avoid holes [S4, S5]. If `bcount` is non-zero, increment an atomic counter and discard the fragment to handle depth buffering or occlusion culling logic [S2]. Calculate the 3D voxel coordinates $(x, y, z)$ from the 2D rasterized position and the dominant axis. Using viewport dimensions $W, H$ and depth $D$:
    *   If axis is 0 (X): $x = \text{depth} - \text{input.pos.x}$, $y = \text{input.pos.z} \times W$, $z = H - \text{input.pos.y}$ [S2].
    *   If axis is 1 (Y): $y = \text{input.pos.x} \times H$, $z = \text{input.pos.z} \times D$, $x = W - \text{input.pos.y}$ [S2].
    *   If axis is 2 (Z): $z = \text{input.pos.x} \times D$, $x = \text{input.pos.z} \times W$, $y = H - \text{input.pos.y}$ [S2].
    Write the voxel data (position and color) to the output buffer. If using a `RWStructuredBuffer`, use an atomic counter (`IncrementCounter()`) to append voxels, as the order is not guaranteed [S2]. If using `RWTexture3D`, write directly to the calculated 3D index [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_shader(triangle_vertices):
    # 1. Calculate Normal
    v0, v1, v2 = triangle_vertices
    edge1 = v1 - v0
    edge2 = v2 - v0
    normal = cross(edge1, edge2)
    
    # 2. Find Dominant Axis
    abs_normal = abs(normal)
    max_comp = max(abs_normal.x, abs_normal.y, abs_normal.z)
    axis = 0
    if abs_normal.y == max_comp:
        axis = 1
    elif abs_normal.z == max_comp:
        axis = 2
        
    # 3. Select View Matrix
    view_mat = ViewX if axis == 0 else (ViewY if axis == 1 else ViewZ)
    
    # 4. Process Vertices
    output_vertices = []
    for i in range(3):
        world_pos = World * vec4(triangle_vertices[i], 1.0)
        view_pos = view_mat * world_pos
        
        # 5. Reproject for Dominant Axis
        if axis == 0:
            reprojected = vec4(view_pos.z, view_pos.y, view_pos.x, 1.0)
        elif axis == 1:
            reprojected = vec4(view_pos.x, view_pos.z, view_pos.y, 1.0)
        else:
            reprojected = vec4(view_pos.x, view_pos.y, view_pos.z, 1.0)
            
        output_vertices.append({
            'pos': reprojected,
            'axis': axis,
            'color': vertex_colors[i]
        })
    return output_vertices

def pixel_shader(fragment):
    # 1. Calculate Voxel Coordinates
    x_ndc = fragment.pos.x
    y_ndc = fragment.pos.y
    z_ndc = fragment.pos.z
    
    voxel_pos = [0, 0, 0]
    if fragment.axis == 0:
        voxel_pos[0] = int((depth - z_ndc) * width)
        voxel_pos[1] = int(y_ndc * width)
        voxel_pos[2] = int((height - x_ndc) * height)
    elif fragment.axis == 1:
        voxel_pos[1] = int((height - z_ndc) * height)
        voxel_pos[0] = int(x_ndc * width)
        voxel_pos[2] = int((depth - y_ndc) * depth)
    else:
        voxel_pos[2] = int((depth - z_ndc) * depth)
        voxel_pos[0] = int(y_ndc * width)
        voxel_pos[1] = int((height - x_ndc) * height)
        
    # 2. Write to Buffer
    if use_structured_buffer:
        index = voxels.IncrementCounter()
        voxels[index] = Voxel(pos=voxel_pos, color=fragment.color)
    else:
        voxels[voxel_pos] = fragment.color
```

**Parameters:**
*   **Grid Resolution:** Commonly tested at $32^3, 128^3, 256^3, 512^3$ [S5].
*   **bcount:** Integer flag. If non-zero, triggers atomic counter increment and fragment discard for depth/occlusion management [S2].
*   **viewport:** Integer flag (0, 1, or 2) to select projection plane in some implementations [S2].

**Edge Cases and Pitfalls:**
*   **Holes:** Standard rasterization only covers pixels where the triangle center lies. This creates holes in the voxel grid. Mitigation requires "bloated" vertices or the `NV_conservative_raster` extension [S4, S5].
*   **Overwriting:** Using atomic counters for structured buffers means write order is undefined. Without proper depth handling (e.g., `bcount` logic), voxels may be overwritten incorrectly [S2].
*   **Lighting Normal:** The Phong lighting calculation must use the surface normal *before* the dominant axis rotation, because the rotation changes the normal's orientation relative to the light source, leading to incorrect lighting if the rotated normal is used [S1].
*   **Precision:** Floating-point atomic operations are not universally supported; manual averaging or integer formats may be required for color accumulation [S3].

## Subtopics

### [GPU-based Conservative Voxelization](2580-gpu-based-conservative-voxelization.md)
A two-pass GPU algorithm that converts 3D triangle meshes into a list of voxels using conservative rasterization, followed by an octree construction step. It uses atomic counters to determine voxel count for buffer allocation. Choose this when memory efficiency (sparse storage) is critical and you need guaranteed hole-free voxelization.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of triangles, as each triangle is processed once in a single draw call [S5].
*   **Memory Complexity:** $O(W \times H \times D)$ for the voxel grid storage [S5].
*   **Performance:** Voxelizing a whole scene at $512 \times 512 \times 512$ resolution took 11.70 ms on an NVIDIA GTX 770 without LOD or optimizations [S5]. The $128^3$ grid is significantly faster [S5].
*   **Strengths:** Single-pass rendering (efficient), leverages highly optimized hardware rasterizer, simple implementation [S5].
*   **Limitations:** Requires conservative rasterization to avoid holes, fixed-resolution grid can be memory-intensive for high resolutions, order-independent writing requires careful handling of overwrites [S1, S5].
*   **When to use:** Real-time global illumination, voxel cone tracing, volumetric effects where speed is prioritized over perfect geometric accuracy [S5].
*   **When not to use:** When sparse storage is strictly required (use octree variants), or when analytical precision is needed without rasterization artifacts.

## Background
This method originated as a practical implementation of the first step of Voxel Cone Tracing, a technique for real-time global illumination [S3, S5]. It was popularized by blog posts and source code from developers like Alexandre Pestana, Simon Yeung, and Alex Nankervis, who adapted hardware rasterization for 3D grids [S5]. The core insight is that the GPU rasterizer can be "tricked" into filling a 3D volume by projecting triangles along their dominant axis, effectively treating the 3D grid as a series of 2D layers [S5]. It is widely used in modern real-time rendering pipelines for GI and volumetric effects [S5].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26
[S2] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S3] Voxel Cone Tracing Renderer - Graphics Programming Portfolio, https://otaviopeixoto1.github.io/portfolio/vctgi/, fetched 2026-10-27 (unofficial)
[S4] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27
[S5] Voxelization using the GPU hardware rasterizer | Alexandre Pestana, https://www.alexandre-pestana.com/voxelization-using-gpu-hardware-rasterizer/, fetched 2026-10-29 (unofficial)
