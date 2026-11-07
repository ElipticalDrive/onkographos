# Axis-Aligned Bounding Box Geometry

Also called: AABB Geometry, RenderClippedBoundingBox, Volume Extent Bounding Box

## Overview
This method renders the clipped axis-aligned bounding box (AABB) geometry of a voxel volume to the framebuffer, providing visual feedback about the volume's spatial extent without contributing to the final volumetric appearance [S1, S3, unofficial]. It differs from [Bounding Box Drawing](2733-bounding-box-drawing.md) by explicitly generating the 36-vertex triangle mesh for the full box surface rather than just drawing wireframe lines, and from [Debug Voxel Visualization](7095-debug-voxel-visualization.md) by operating on the aggregate volume bounds rather than individual voxel cells [S2, S3, unofficial].

## How it works
**Inputs:**
- `box`: An AABB structure containing `minCorner()` and `maxCorner()` vectors (typically `vec3` or `fvec3`) [S3, unofficial].
- `img_size`: Optional 3D integer vector defining the voxel grid dimensions, used to compute normalized texture coordinates [S3, unofficial].
- `tcoordFlag`: Boolean flag indicating whether to generate and upload texture coordinates [S1, unofficial].
- `currentBlock`: Index of the current rendering block, used for validation [S1, unofficial].
- `numberOfBlocks`: Total number of blocks to process [S1, unofficial].

**Outputs:**
- A Vertex Array Object (VAO) and Vertex Buffer Object (VBO) containing the box geometry [S2, unofficial].
- Optional texture coordinate buffer [S3, unofficial].

**Data Structures:**
- `vertices`: Array of 36 floats (12 triangles × 3 vertices × 3 components) defining the 6 faces of the box [S2, unofficial].
- `texc`: Array of 8 `fvec3` elements (one per corner) for texture coordinates [S3, unofficial].

**Steps:**
1. Validate that `currentBlock` is within the range `[0, numberOfBlocks)` [S1, unofficial].
2. Extract the minimum and maximum corners of the AABB:
   $$
   \begin{aligned}
   x_0 &= \text{box.minCorner().x()}, & y_0 &= \text{box.minCorner().y()}, & z_0 &= \text{box.minCorner().z()} \\
   x_1 &= \text{box.maxCorner().x()}, & y_1 &= \text{box.maxCorner().y()}, & z_1 &= \text{box.maxCorner().z()}
   \end{aligned}
   $$
   [S3, unofficial]
3. Define the 8 corner positions in the following order:
   $$
   \begin{aligned}
   & (x_0, y_0, z_1), (x_1, y_0, z_1), (x_1, y_1, z_1), (x_0, y_1, z_1) \\
   & (x_0, y_0, z_0), (x_1, y_0, z_0), (x_1, y_1, z_0), (x_0, y_1, z_0)
   \end{aligned}
   $$
   [S3, unofficial]
4. If `tcoordFlag` is true and `img_size` is provided, compute normalized texture coordinates:
   $$
   \begin{aligned}
   dx &= 0.5 / \text{img\_size.x()}, & dy &= 0.5 / \text{img\_size.y()}, & dz &= 0.5 / \text{img\_size.z()} \\
   x_0 &= \text{min\_corner.x()} / \text{img\_size.x()} + dx, & x_1 &= \text{max\_corner.x()} / \text{img\_size.x()} - dx \\
   y_0 &= \text{min\_corner.y()} / \text{img\_size.y()} + dy, & y_1 &= \text{max\_corner.y()} / \text{img\_size.y()} - dy \\
   z_0 &= \text{min\_corner.z()} / \text{img\_size.z()} + dz, & z_1 &= \text{max\_corner.z()} / \text{img\_size.z()} - dz
   \end{aligned}
   $$
   [S3, unofficial]
5. Generate VAO and VBO:
   ```python
   glGenVertexArrays(1, &vao)
   glGenBuffers(1, &vbo)
   ```
   [S2, unofficial]
6. Upload vertex data to the GPU:
   ```python
   glBindBuffer(GL_ARRAY_BUFFER, vbo)
   glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW)
   ```
   [S2, unofficial]
7. Set vertex attribute pointer:
   ```python
   glBindVertexArray(vao)
   glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0)
   glEnableVertexAttribArray(0)
   ```
   [S2, unofficial]
8. If `tcoordFlag` is true, copy texture coordinates to the texture coordinate buffer and mark bounds as dirty:
   ```python
   memcpy(mTexCoord->ptr(), texc, sizeof(texc))
   mGeometry->setBoundsDirty(True)
   ```
   [S3, unofficial]
9. Return `True` if rendering should abort, `False` otherwise [S1, unofficial].

**Parameters:**
- `minB`: Minimum corner of the bounding box, typically set to `(-1.0, -1.0, -0.42578125)` [S2, unofficial].
- `maxB`: Maximum corner of the bounding box, typically set to `(1.0, 1.0, 0.42578125)` [S2, unofficial].
- `renWin`: Pointer to `vtkRenderWindow` for context and framebuffer access [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `img_size` contains zero components, texture coordinate generation fails and logs an error [S3, unofficial].
- Scalar types `VTK_CHAR`, `VTK_BIT`, `VTK_ID_TYPE`, and `VTK_STRING` are not supported in the broader rendering context, though this method focuses on geometry [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for geometry generation, as the box always has 8 corners and 12 triangles [S2, unofficial].
- **Memory Complexity:** $O(1)$, requiring fixed-size buffers for 36 floats and 8 texture coordinate vectors [S2, S3, unofficial].
- **Strengths:** Simple, fast, and effective for visualizing volume extent [S1, S3, unofficial].
- **Limitations:** Does not provide per-voxel detail or internal structure [S3, unofficial].

## Background
This method is used in volume rendering systems to visualize the spatial bounds of voxel data, often in conjunction with debug or visualization tools [S1, S3, unofficial]. It relies on standard OpenGL rasterization states and simple geometric primitives [G].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S3] RaycastVolume.cpp, https://visualizationlibrary.org/documentation/_raycast_volume_8cpp_source.html, fetched 2026-10-27 (unofficial)
