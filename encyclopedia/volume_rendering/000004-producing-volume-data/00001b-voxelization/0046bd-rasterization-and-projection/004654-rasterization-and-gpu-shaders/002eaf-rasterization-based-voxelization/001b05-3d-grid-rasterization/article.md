# 3D Grid Rasterization

Also called: Grid-Snapped Voxelization

## Overview
3D Grid Rasterization converts a triangle mesh into a discrete 3D occupancy grid by projecting the mesh onto three orthogonal planes (XY, YZ, XZ) using hardware rasterization and combining the results [S4, unofficial]. It differs from CPU-based bounding-box intersection methods by leveraging GPU fragment shaders to determine voxel occupancy, and from full surface reconstruction methods (like o-voxel) by outputting only binary presence without solving for dual vertices or QEF [S1, unofficial]. It is a rasterization-based voxelization technique that produces a dense or sparse grid depending on the storage format used [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertex positions (and optionally normals/colors) in local or world space, and a target grid resolution [S4, unofficial].
*   **Output:** A 3D voxel grid representing occupancy. This can be a compact bit-packed array (1 bit per voxel) or a 3D texture storing RGBA data [S4, unofficial][S3, unofficial].

**Data Structures:**
*   **Voxel Grid Storage:** A `std::vector<uint32_t>` of size $(N_x \times N_y \times N_z) / 32$, where each `uint32_t` stores 32 voxels (1 bit per voxel) [S4, unofficial].
*   **3D Texture:** A `rgba16f` 3D texture where each voxel stores RGBA 16-bit floating point values [S3, unofficial].
*   **Hash Table (CPU variant):** A hash table mapping `VoxelCoord` (struct with int x, y, z) to occupancy, using a hash function $H = x \cdot 73856093 \oplus y \cdot 19349663 \oplus z \cdot 83492791$ [S1, unofficial].

**Parameters:**
*   `resolution`: Target grid resolution along the smallest bounding box dimension. Typical values: 64, 128, 256, 512 [S4, unofficial].
*   `voxel_size`: Calculated as $\min(\text{boundingBoxSize}) / \text{resolution}$ [S4, unofficial].
*   `surface_only`: Boolean flag. If true, only surface voxels are generated; if false, interior is filled [S10, unofficial].

**Steps:**

1.  **Compute Grid Dimensions:**
    Find the bounding box of the mesh vertices ($minCoords, maxCoords$) [S5, unofficial]. Calculate `voxelSize` as $\min(\text{boundingBoxSize}) / \text{resolution}$ [S4, unofficial]. Round the number of voxels in each dimension up to the nearest multiple of 32 using `makeMultipleOf32` [S4, unofficial]. Calculate `minCorner` and `maxCorner` based on the bounding box center and the rounded voxel count:
    $$ \text{minCorner} = \text{boundingBoxCenter} - 0.5 \times (\text{roundedNbVoxels} \times \text{voxelSize}) $$
    [S4, unofficial]

2.  **Initialize Storage:**
    Allocate the 3D texture or bit-packed array [S4, unofficial]. If using the GPU rasterization approach with 3D textures, allocate three temporary textures for projections: $xProjTextureId$, $yProjTextureId$, and $zProjTextureId$ [S4, unofficial].

3.  **X-Axis Projection (YZ Plane):**
    *   Allocate $xProjTextureId$ with dimensions $(N_y, N_z, N_x/32)$ [S4, unofficial].
    *   For each slice index `sliceX` from 0 to $N_x$ with step 32:
        *   Bind the slice `sliceX/32` of the texture to the framebuffer color attachment [S4, unofficial].
        *   Set viewport to $(N_y, N_z)$ [S4, unofficial].
        *   Clear buffer.
        *   Set up orthographic projection:
            $$ \text{viewProj} = \text{ortho}(minY, maxY, minZ, maxZ, x, x + 32 \times \text{voxelSize}) $$
            Apply rotation: $\text{rot} = \text{rotate}(-90^\circ, \text{x-axis}) \times \text{rotate}(90^\circ, \text{z-axis})$ [S4, unofficial].
        *   Render the mesh. The fragment shader writes to the texture [S4, unofficial]. For bit-packed storage, use additive blending or logic operations to set bits [S4, unofficial].

4.  **Y-Axis Projection (XZ Plane):**
    *   Allocate $yProjTextureId$ with dimensions $(N_x, N_z, N_y/32)$ [S4, unofficial].
    *   For each slice index `sliceY` from 0 to $N_y$ with step 32:
        *   Bind the slice `sliceY/32` of the texture to the framebuffer [S4, unofficial].
        *   Set viewport to $(N_x, N_z)$ [S4, unofficial].
        *   Clear buffer.
        *   Set up orthographic projection:
            $$ \text{viewProj} = \text{ortho}(minX, maxX, minZ, maxZ, y, y + 32 \times \text{voxelSize}) $$
            Apply rotation: $\text{rot} = \text{rotate}(-90^\circ, \text{x-axis})$ [S4, unofficial].
        *   Render the mesh.

5.  **Z-Axis Projection (XY Plane):**
    *   Allocate $zProjTextureId$ with dimensions $(N_x, N_y, N_z/32)$ [S4, unofficial].
    *   For each slice index `sliceZ` from 0 to $N_z$ with step 32:
        *   Bind the slice `sliceZ/32` of the texture to the framebuffer [S4, unofficial].
        *   Set viewport to $(N_x, N_y)$ [S4, unofficial].
        *   Clear buffer.
        *   Set up orthographic projection:
            $$ \text{viewProj} = \text{ortho}(minX, maxX, minY, maxY, z, z + 32 \times \text{voxelSize}) $$
            Apply rotation: $\text{rot} = \text{rotate}(180^\circ, \text{z-axis}) \times \text{rotate}(180^\circ, \text{y-axis})$ [S4, unofficial].
        *   Render the mesh.

6.  **Combine Projections (Optional/Alternative):**
    If using a single final texture (e.g., $zProjTextureId$ as the output), a "Compile Projections" pass may be used [S4, unofficial]. Bind $xProjTextureId$ and $yProjTextureId$ to texture units [S4, unofficial]. For each slice `sliceZ`, draw a triangle strip to combine X and Y projections into the Z texture using a fragment shader that checks occupancy from all three views [S4, unofficial].

7.  **Retrieve Result:**
    Bind the final texture (e.g., $zProjTextureId$) and read the texture data into a CPU buffer using `glGetTexImage` with format `GL_RGBA_INTEGER` and type `GL_UNSIGNED_INT_8_8_8_8` [S4, unofficial].

**Pseudocode (CPU Scanline Variant):**
The following pseudocode represents the CPU-based intersect scanline method, which is a core component of the occupancy determination in some implementations [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxelize_mesh_cpu(verts, faces, grid_size):
    # 1. Compute grid parameters
    min_coords = [min(v[i] for v in verts) for i in range(3)]
    max_coords = [max(v[i] for v in verts) for i in range(3)]
    bounding_box_size = [max_coords[i] - min_coords[i] for i in range(3)]
    voxel_size = min(bounding_box_size) / grid_size
    
    nb_voxels = [math.ceil(bounding_box_size[i] / voxel_size) for i in range(3)]
    nb_voxels_rounded = [math.ceil(n / 32) * 32 for n in nb_voxels]
    
    min_corner = [(min_coords[i] + max_coords[i]) / 2 - 0.5 * (nb_voxels_rounded[i] * voxel_size) for i in range(3)]
    
    occupied_voxels = set()
    
    for i in range(0, len(faces), 3):
        p0 = verts[faces[i]]
        p1 = verts[faces[i+1]]
        p2 = verts[faces[i+2]]
        
        for ax2 in range(3):
            ax0 = (ax2 + 1) % 3
            ax1 = (ax2 + 2) % 3
            
            t = [[p0[ax0], p0[ax2]], [p1[ax0], p1[ax2]], [p2[ax0], p2[ax2]]]
            t.sort(key=lambda v: v[1])
            
            start = int(math.floor((t[0][1] - min_corner[ax1]) / voxel_size))
            mid = int(math.floor((t[1][1] - min_corner[ax1]) / voxel_size))
            end = int(math.floor((t[2][1] - min_corner[ax1]) / voxel_size))
            
            start = max(0, min(start, nb_voxels_rounded[ax1] - 1))
            mid = max(0, min(mid, nb_voxels_rounded[ax1] - 1))
            end = max(0, min(end, nb_voxels_rounded[ax1] - 1))
            
            scan_line_half(start, mid, t, voxel_size, min_corner, ax0, ax1, ax2, occupied_voxels)
            t_upper = [t[2], t[1], t[0]]
            scan_line_half(mid, end, t_upper, voxel_size, min_corner, ax0, ax1, ax2, occupied_voxels)

    return occupied_voxels

def lerp(a, b, t, val_a, val_b):
    if a == b: return val_a
    alpha = (t - a) / (b - a)
    return [val_a[i] + alpha * (val_b[i] - val_a[i]) for i in range(len(val_a))]

def scan_line_half(row_start, row_end, t, voxel_size, min_corner, ax0, ax1, ax2, occupied_voxels):
    for y_idx in range(row_start, row_end):
        y = (y_idx + 1) * voxel_size + min_corner[ax1]
        
        t3 = lerp(t[0][1], t[1][1], y, [t[0][0], t[0][2]], [t[1][0], t[1][2]])
        t4 = lerp(t[0][1], t[2][1], y, [t[0][0], t[0][2]], [t[2][0], t[2][2]])
        
        if t3[0] > t4[0]:
            t3, t4 = t4, t3
            
        line_start = int(math.floor(t3[0] / voxel_size))
        line_end = int(math.floor(t4[0] / voxel_size))
        
        line_start = max(0, min(line_start, 1024))
        line_end = max(0, min(line_end, 1024))
        
        for x_idx in range(line_start, line_end):
            x = (x_idx + 1) * voxel_size + min_corner[ax0]
            z = lerp(t3[0], t4[0], x, [t3[1], 0.0], [t4[1], 0.0])
            z_idx = int(math.floor(z / voxel_size))
            
            occupied_voxels.add((x_idx, y_idx, z_idx))
```

**Edge Cases and Pitfalls:**
*   **Precision Glitches:** On some hardware (e.g., Intel HD 4400 on Linux), rasterization may suffer from glitches and lower maximum supported resolution compared to Windows [S4, unofficial].
*   **Triangles Outside Grid:** Triangles outside the `[GridMin, GridMax]` region are not voxelized because they map outside the valid NDC range for the grid [S3, unofficial].
*   **Division by Zero:** Handle cases where grid dimension is zero using ternary checks (e.g., `ax == 0.0 ? 0 : ...`) [S8, unofficial].
*   **Degenerate Triangles:** Filter out degenerate triangles (collinear or duplicate vertices) during mesh scaling [S11, unofficial].

## Subtopics

### [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md)
A-Buffer Solid Voxelization is a GPU-based technique that voxelize a watertight mesh by rasterizing it into a fixed-width, height, and depth 3D grid and resolving multiple depth intersections per pixel using an A-buffer linked list structure. It differs from standard depth-buffer rasterization by retaining all fragment depth information rather than discarding occluded fragments, allowing it to output a linked list of starting and ending voxels for each row and column in the grid.

### [Adaptive Sparse Voxelization with Bounding Box Thickness Cases](1686-adaptive-sparse-voxelization-with-bounding-box-thickness-cas.md)
This method rasterizes a triangle mesh into a sparse voxel data structure by classifying each triangle's axis-aligned bounding box (AABB) into nine cases based on its thickness in grid coordinates. It optimizes overlap testing by using simple iteration for 1D-thick boxes, 2D projection tests for 2D-thick boxes, and combined 3D plane and 2D projection tests for 3D-thick boxes.

### [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md)
Geometry Shader Voxelization converts polygonal meshes (`.obj`, `.ply`) into a discrete 3D occupancy grid by rasterizing triangles directly into a 3D texture using an OpenGL Geometry Shader. It determines occupied voxels by projecting triangles onto axis-aligned planes, computing edge distances, and iterating through the triangle's bounding box to check for overlaps in all three dimensions (XY, YZ, ZX). Unlike [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md) which uses a 2D accumulation buffer to resolve depth, this method explicitly tests 3D spatial overlap per voxel index within the shader loop, ensuring watertightness without traditional framebuffer depth buffers.

### [GPU Mesh Voxelizer (Grid Generation)](11413-gpu-mesh-voxelizer-grid-generation.md)
This method generates a dense 3D voxel grid by computing the center coordinates of each voxel cell directly on the GPU using a compute shader. It solves the problem of converting a mesh's spatial extent into a fixed-width, height, and depth 3D grid structure, returning a flat array of voxel centers. Unlike rasterization-based approaches that project geometry onto slices, this method calculates grid dimensions from the mesh's bounding box and fills the grid with uniform cells, making it suitable for preprocessing steps where a regular grid topology is required. It builds on [3D Grid Rasterization](903-3d-grid-rasterization.md) by providing the specific grid generation mechanism.

### [Derived Grid Rasterization](11477-derived-grid-rasterization.md)
Derived Grid Rasterization converts an unstructured 3D volume mesh (e.g., tetrahedra, hexahedra) into a dense, regular 3D voxel grid spanning the mesh's Axis-Aligned Bounding Box (AABB). This creates a structured data layout required for efficient volume ray-marching and axis-aligned slice planes on data that originally lacks a grid structure. Unlike sibling methods that may produce sparse structures or rely on different rasterization primitives, this method explicitly maps each voxel to its owning unstructured cell and local tetrahedron index, enabling direct lookup during rendering. It builds on [3D Grid Rasterization](903-3d-grid-rasterization.md) by handling unstructured cell topology rather than simple surface mesh voxelization.

### [Unstructured Mesh Pixelization via Convex Element Containment](11479-unstructured-mesh-pixelization-via-convex-element-containmen.md)
A pixelization routine that transforms variable-resolution unstructured mesh values into an image buffer by depositing values through nearest-neighbor interpolation with anti-aliasing, decoupling the pixelization step from the underlying data model. It differs from standard rasterization by using a geometric containment test based on dot product signs relative to element centroids, which eliminates the requirement for consistent vertex winding order (CW/CCW).

### Projection Voxelization Shader
#### Definition
A vertex shader that generates a full-screen quad to process each voxel slice, outputting non-normalized voxel coordinates for the fragment shader [S4, unofficial].

#### Steps
1.  Define quad corners: `const vec2 corners[4] = vec2[](vec2(0,0), vec2(1,0), vec2(0,1), vec2(1,1));` [S4, unofficial].
2.  Calculate voxel coordinate: `texelCoord = vec3(corner * gridSize.xy, slice);` where `corner` is selected by `gl_VertexID` [S4, unofficial].
3.  Set clip space position: `gl_Position = vec4(corner*2-1, 0, 1);` [S4, unofficial].

#### Layout
*   Input: `uniform uint slice` defining the current Z-slice index being processed.
*   Input: `uniform vec3 gridSize` defining the voxel grid dimensions.

## Operating characteristics
*   **Time Complexity:** $O(N \times V)$ where $N$ is the number of triangles in the mesh and $V$ is the number of slices (proportional to grid resolution) [S4, unofficial].
*   **Measured Performance:** On an Intel HD 4400 with a Suzanne model (31,000 vertices): $64^3$ grid takes 2 ms; $128^3$ takes 9 ms; $256^3$ takes 10 ms; $512^3$ takes 120 ms [S4, unofficial].
*   **Scaling:** Performance scales linearly with the number of triangles and the number of slices (resolution) [S4, unofficial].
*   **Strengths:** Leverages GPU hardware rasterization for speed; compact 1-bit-per-voxel storage format reduces memory usage [S4, unofficial].
*   **Limitations:** Precision glitches on some hardware; triangles outside the grid bounds are ignored; requires multiple passes for 3D coverage [S3, unofficial][S4, unofficial].
*   **When to use:** When fast occupancy grid generation is needed for global illumination or collision detection, and surface reconstruction is not required [S1, unofficial].
*   **When not to use:** When high-precision surface reconstruction (dual vertices) is needed, or when the mesh is highly sparse and CPU-based bounding box checks are more efficient [S1, unofficial].

## Background
3D Grid Rasterization is a technique used in voxel global illumination (VXGI) and real-time voxelization [S3, unofficial][S4, unofficial]. It was popularized by NVIDIA's VXGI research, which uses voxelization as a preprocessing step for cone tracing [S3, unofficial]. The specific GPU-based rasterization approach using multiple orthogonal projections and 3D textures is implemented in various open-source projects and engines [S4, unofficial]. The CPU-based scanline variant is derived from the o-voxel algorithm by Jianfeng Xiang, which focuses on efficient occupancy detection [S1, unofficial].

## Sources
[S1] yuyujunjun/mesh-voxelize, https://github.com/yuyujunjun/mesh-voxelize, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S3] BoyBaykiller/IDKEngine, https://github.com/BoyBaykiller/IDKEngine, fetched 2026-10-26 (unofficial)
[S4] piellardj/voxelizer-gpu, https://github.com/piellardj/voxelizer-gpu, fetched 2026-10-26 (unofficial)
[S5] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S6] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S7] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S8] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S9] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
[S10] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S11] GitHub - Septaris/mesh_vox, https://github.com/Septaris/mesh_vox, fetched 2026-10-30 (unofficial)
