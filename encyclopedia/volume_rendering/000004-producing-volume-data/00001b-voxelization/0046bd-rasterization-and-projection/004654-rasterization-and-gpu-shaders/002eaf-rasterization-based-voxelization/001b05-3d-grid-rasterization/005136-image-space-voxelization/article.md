# Image-space Voxelization

Also called: FBO-based Voxelization, Legolizer Voxelization

## Overview
Image-space voxelization projects the vertices of a triangular mesh into a 2D image stored in a Frame Buffer Object (FBO) to determine voxel locations and colors [S1]. It solves the problem of converting boundary models into volumetric representations by leveraging GPU rasterization hardware, specifically using a single render pass and a single buffer to preserve original vertex colors, unlike methods requiring multiple passes or z-buffer stacks [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh $M$ already residing on the GPU, a grid size parameter $size$, and a cube size parameter $r$ [S1].
*   **Output:** A Vertex Buffer Object (VBO) containing the voxelized data, where each pixel in the FBO corresponds to a vertex in the VBO [S1].

**Data Structures:**
*   **FBO Layout:** The Frame Buffer Object stores vertex color in the RGB channels (representing $x, y, z$ coordinates) and the normalized voxel number in the Alpha channel [S1].
*   **VBO Layout:** Each pixel downloaded from the FBO becomes a vertex in the VBO [S1].

**Parameters:**
*   $size$: The dimension of the cubic voxel grid [S1].
*   $r$: The cube size, used to determine voxel indices and geometry generation [S1].

**Steps:**
1.  Create an FBO [S1].
2.  Set an orthogonal projection to the FBO [S1].
3.  Enable the mapping shader [S1].
4.  Draw the mesh $M$ (which is already on the GPU) [S1].
5.  Disable the mapping shader [S1].
6.  Download the FBO data [S1].
7.  Create a VBO from the downloaded FBO data [S1].
8.  Return the VBO pointer [S1].

**Vertex Shader Logic:**
The vertex shader computes the voxel index for each vertex $vp$ with color $vc$ [S1].
1.  Let $voxelxyz \leftarrow vc_{xyz}$ [S1].
2.  Compute the linearized 3D grid index:
    $$voxelw = \text{int}(vp_x) \times size^2 + \text{int}(vp_y) \times size + \text{int}(vp_z)$$
    where $vp_x, vp_y, vp_z$ are the projected vertex coordinates [S1].
3.  If $voxelw$ is outside the valid grid range, set $voxelw \leftarrow \text{NULL}$ [S1].
4.  Compute the normalized voxel number:
    $$voxelw_{normalized} = \frac{voxelw}{size^3}$$
    This value is stored in the Alpha channel of the FBO [S1].
5.  Emit the voxel as a pixel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(vp: tuple, vc: tuple, size: int) -> tuple:
    # vp: Vertex 3D Position (projected coordinates)
    # vc: Vertex Color (RGB, representing xyz)
    # size: Grid size
    # Returns: (color_rgb, alpha_normalized_voxel_index)
    voxelxyz = vc
    voxelw = int(vp[0]) * (size * size) + int(vp[1]) * size + int(vp[2])
    if voxelw < 0 or voxelw >= size ** 3:
        return None, None  # NULL value prevents invalid voxel generation
    voxelw_normalized = voxelw / (size * size * size)
    return vc, voxelw_normalized
```

**Edge Cases and Pitfalls:**
*   Vertices outside the defined grid receive a null value for $voxelw$ to prevent invalid voxel generation [S1].
*   Vertices written to the same position in the FBO are overwritten by the Z-buffer test, meaning only the front surface is completely displayed, while back surfaces may contain holes [S1].
*   Holes in back surfaces can be avoided by increasing the frame buffer resolution or mesh refinement [S1].

## Subtopics

### A-Buffer Solid Voxelization
Uses accumulation buffers for solid voxelization. Choose when anti-aliasing or transparency handling is required. [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md)

### Adaptive Sparse Voxelization with Bounding Box Thickness Cases
Adapts voxel density based on geometry complexity. Choose for memory efficiency in sparse scenes. [Adaptive Sparse Voxelization with Bounding Box Thickness Cases](1686-adaptive-sparse-voxelization-with-bounding-box-thickness-cas.md)

### Geometry Shader Voxelization
Generates voxels directly in the geometry shader stage. Choose for fine-grained control over voxel generation per primitive. [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md)

### GPU Mesh Voxelizer (Grid Generation)
General GPU-based grid generation techniques. Choose for broader grid generation needs beyond image-space projection. [GPU Mesh Voxelizer (Grid Generation)](11413-gpu-mesh-voxelizer-grid-generation.md)

### Derived Grid Rasterization
Rasterizes onto a derived grid structure. Choose when the grid structure is non-uniform or derived from scene data. [Derived Grid Rasterization](11477-derived-grid-rasterization.md)

### Unstructured Mesh Pixelization via Convex Element Containment
Handles unstructured meshes via containment tests. Choose for non-tetrahedral/unstructured input data. [Unstructured Mesh Pixelization via Convex Element Containment](11479-unstructured-mesh-pixelization-via-convex-element-containmen.md)

### Fast Solid Voxelization via Planar Raycasting
Uses raycasting for solid voxelization. Choose for high-accuracy solid interiors. [Fast Solid Voxelization via Planar Raycasting](13748-fast-solid-voxelization-via-planar-raycasting.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of vertices in the mesh, as it involves a single draw call and shader execution per vertex [S1].
*   **Memory Complexity:** Proportional to the FBO size ($size^2$ for the 2D image) and the resulting VBO size [S1].
*   **Performance:** Uses a single render pass and a single buffer, making it faster than methods requiring six passes and six z-buffers [S1].
*   **Strengths:** Preserves original voxel colors; efficient single-pass rasterization; leverages GPU hardware [S1].
*   **Limitations:** Back surfaces may contain holes due to Z-buffer occlusion; resolution limited by FBO size and mesh refinement [S1].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method is part of the Legolizer system [S1]. It was developed to create LEGO representations from boundary models in real-time [S1]. The approach differs from earlier methods by Karabassi et al. and Passalis et al., which used six z-buffers and six passes, by utilizing a single render pass and buffer [S1]. The method relies on mesh subdivision to ensure vertices sample the surface well, allowing the image-space projection to create a good volumetric approximation [S1].

## Sources
[S1] Legolizer: A Real-Time System for Modeling and Rendering LEGO Representations of Boundary Models, https://doi.org/10.1109/sibgrapi.2009.18, fetched 2026-11-02
