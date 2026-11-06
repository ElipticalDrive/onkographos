# GPU Voxelization
Also called: Basic GPU Voxelization, GPU Rasterization-based Voxelization, GPU Voxelization via GPU Rasterization, GPU Voxelization via Orthogonal Rasterization, GPU-based Voxelization, Voxelization using 3D Texture UAV, Voxelization via GPU Rasterization

## Overview
GPU Voxelization converts a triangle mesh into a regular voxel grid by leveraging the GPU's hardware rasterization pipeline to map fragments directly to 3D grid cells [S1, S3, S5, unofficial]. It solves the performance bottleneck of CPU-based ray-tracing voxelization by parallelizing the intersection test across the rasterizer [S5, unofficial]. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using a 3D texture UAV or SSBO as the direct write target rather than a 2D accumulation buffer, and from [GPU Volume Kernel Voxelization](1496-gpu-volume-kernel-voxelization.md) by relying on standard triangle rasterization rather than volume kernels.

## How it works
**Inputs and Outputs:**
- Input: Triangle mesh vertices, voxel grid resolution ($W, H, D$), voxel step size (edge length), and projection parameters.
- Output: A voxel grid (3D texture or SSBO) marking occupied cells, or a list of voxel coordinates.

**Data Structures:**
- **Voxel Grid:** A 3D texture bound as an Unordered Access View (UAV) [S3, unofficial] or a 1D Shader Storage Buffer Object (SSBO) `int cnts[]` of size $W \times H \times D$ [S5, unofficial].
- **Projection Matrices:** Orthographic projection matrices mapping world space to clip space $[-1, 1]^2 \times [0, 1]$ and then to voxel space $[0, W] \times [0, H] \times [0, D]$ [S1, unofficial].

**Steps:**
1. **Compute AABB:** Iterate over mesh vertices to find min/max $x, y, z$ to determine the Axis-Aligned Bounding Box (AABB) [S5, unofficial].
2. **Calculate Resolution:** Compute grid dimensions: $W = \lfloor (max_x - min_x) / step \rfloor + 1$, similarly for $H, D$ [S5, unofficial].
3. **Setup Render State:** Disable depth testing and back-face culling to ensure all triangles are processed regardless of orientation [S3, S5, unofficial].
4. **Rasterize:** Render the mesh using an orthographic camera aligned with a voxel grid axis. The viewport dimensions match the 2D projection of the voxel grid [S3, unofficial].
5. **Fragment Processing:** In the Pixel/Fragment Shader, map the fragment's position to voxel coordinates and write data to the grid [S3, unofficial].

**Formulas:**
- **Voxel Index (SSBO):** $index = y \times (D \times W) + z \times W + x$ [S5, unofficial].
- **Voxel Coordinates:** $x = \lfloor (FragPos.x - min_x) / step \rfloor$, $y = \lfloor (FragPos.y - min_y) / step \rfloor$, $z = \lfloor (FragPos.z - min_z) / step \rfloor$ [S5, unofficial].
- **Homogeneous Mapping:** After applying the voxel-space projection matrix, divide $x, y, z$ by $w$ to get cell coordinates [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position):
    # Transform to clip space using orthographic projection
    clip_pos = projection_matrix * view_matrix * vec4(position, 1.0)
    return clip_pos

# Geometry Shader (Optional, for Multi-View or Conservative Rasterization)
def geometry_shader(triangle_vertices):
    # Calculate face normal to select projection axis (Multi-View)
    v1 = triangle_vertices[1] - triangle_vertices[0]
    v2 = triangle_vertices[2] - triangle_vertices[0]
    normal = cross(v1, v2)
    
    # Select view based on max absolute normal component
    if abs(normal.x) > abs(normal.y) and abs(normal.x) > abs(normal.z):
        view_idx = 0 # X-axis
    elif abs(normal.y) > abs(normal.z):
        view_idx = 1 # Y-axis
    else:
        view_idx = 2 # Z-axis
        
    # Transform vertices using selected view-project matrix
    selected_vp = view_project_matrices[view_idx]
    for v in triangle_vertices:
        emit(selected_vp * vec4(v, 1.0))

# Fragment Shader
def fragment_shader(frag_pos_world):
    # Calculate voxel indices
    x = floor((frag_pos_world.x - box_min.x) / step)
    y = floor((frag_pos_world.y - box_min.y) / step)
    z = floor((frag_pos_world.z - box_min.z) / step)
    
    # Calculate linear index
    index = y * (resolution.z * resolution.x) + z * resolution.x + x
    
    # Atomic write to SSBO or UAV
    atomic_add(ssbo[index], 1)
```

**Edge Cases and Pitfalls:**
- **Cracks:** Occur when triangle slope is steep relative to view direction (depth gradient > 1.0) [S3, unofficial].
- **Holes:** Occur if primitives are projected in a direction where no pixel centers are covered [S3, unofficial].
- **Extra Voxels:** Manual conservative rasterization (edge expansion) may shade extra pixels [S3, unofficial].

## Subtopics

### [VoxelPipe](2797-voxelpipe.md)
VoxelPipe is a highly flexible, programmable software pipeline for triangle voxelization implemented in CUDA. It solves the problem of converting triangle meshes into voxel grids with support for fully conservative and thin voxelizations, arbitrary render targets (boolean, floating-point, vector), and user-defined shaders. It distinguishes itself from prior binary voxelizers by supporting arbitrary framebuffer types and programmability via vertex and fragment shaders, tackling load balancing via a tile-based algorithm instead of assigning individual triangles to single threads with global memory atomics, and optimizing the triangle/box overlap test. [VoxelPipe](2797-voxelpipe.md)

### [Conservative Rasterization](2936-conservative-rasterization.md)
This method prevents holes in voxelized geometry caused by triangles smaller than the voxel grid resolution by rasterizing all fragments that "touch" the triangle. It achieves this by analytically expanding the triangle in clip space before rasterization, ensuring that any voxel potentially intersected by the triangle is covered. Unlike hardware conservative rasterization (limited to NVIDIA) or Multi-Sample Anti-Aliasing (MSAA) activation, this geometric approach is hardware-agnostic but computationally more intensive, requiring a geometry shader for the expansion logic. [Conservative Rasterization](2936-conservative-rasterization.md)

## Operating characteristics
- **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of triangles and $R$ is the rasterization cost (pixels covered).
- **Memory Complexity:** $O(W \cdot H \cdot D)$ for the voxel grid [S5, unofficial].
- **Performance:** Extremely fast due to GPU parallelism, significantly faster than CPU ray-tracing [S5, unofficial].
- **Strengths:** High throughput, leverages existing graphics hardware [S5, unofficial].
- **Limitations:** Memory intensive for high resolutions; prone to cracks/holes without conservative rasterization or multi-view [S3, S5, unofficial].
- **When to Use:** Real-time applications requiring fast mesh-to-voxel conversion [S5, unofficial].
- **When Not to Use:** When exact geometric intersection is required without approximation, or when memory is severely constrained.

## Background
Originated from the need for efficient voxelization for global illumination and physics simulations [S1, S5, unofficial]. Based on approaches by Michael Schwarz [S1, unofficial] and NVIDIA [S3, S5, unofficial]. Used in real-time ray tracing, position-based dynamics, and voxel cone tracing [S1, S4, S5, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S3] 【转】The Basics of GPU Voxelization - 竹庭, https://www.cnblogs.com/hustztz/p/7575729.html, fetched 2026-10-27 (unofficial)
[S4] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
[S5] 体素化Voxelization：基于GPU的三维体素化 - YangWC's Blog, https://yangwc.com/2019/06/11/Voxelization/, fetched 2026-10-30 (unofficial)
