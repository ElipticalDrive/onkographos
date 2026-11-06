# Interactive Slice and Point-Based Volume Rendering

Also called: VREMS slice/voxel pipeline

## Overview
This method renders a 3D segmentation volume by dynamically extracting a 2D slice from the volume data, uploading it to a 2D texture, and drawing a slice plane that samples it, while overlaying segmentation labels as 3D points at voxel coordinates [S1, unofficial]. It differs from standard texture slicing by decoupling the volumetric intensity visualization (the slice) from the discrete label visualization (the points), allowing for distinct rendering pipelines and per-label coloring [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: A 3D intensity image array (`image`), a list of 3D label arrays (`labels`), and corresponding per-label colors (`labels_colors`).
- **Outputs**: A rendered video or sequence of images showing the rotating slice and point cloud.

**Data Structures**
- **Slice Texture**: A 2D OpenGL texture of format `GL_RED` and type `GL_FLOAT`. The data is a flattened 1D array of 32-bit floats representing a single 2D slice of the volume.
- **Voxel VBO**: A 1D array of interleaved `float32` triples $(x, y, z)$ representing voxel coordinates. The stride is `voxels.itemsize * 3` (12 bytes). Offset is 0 [S1, unofficial].
- **Slice Index Buffer**: A 6-element `uint32` array `[0, 1, 2, 0, 2, 3]` defining two triangles for a quad [S1, unofficial].
- **VAOs**: One Vertex Array Object (VAO) per label, containing the VBO for that label's voxel coordinates.

**Parameters**
- `oversampling`: Multiplier for the off-screen framebuffer resolution relative to the window size [S1, unofficial].
- `alpha_image`: Hardcoded alpha blending factor for boundary texture rendering (default 0.8) [S1, unofficial].
- `background_color`: Default clear color for the off-screen framebuffer (default `(0.15, 0.15, 0.15, 1.0)`) [S1, unofficial].
- `view_distance`: Default camera distance from scene center (default 1.5) [S1, unofficial].
- `rotation_speed`: Default angular velocity for the Y-axis rotation per frame (default 1.0) [S1, unofficial].
- `camera_height`: Default vertical offset for the camera placement function (default 0.65) [S1, unofficial].
- `only_image`: Boolean flag. If true, renders four pre-computed boundary textures instead of the interactive slice plane [S1, unofficial].

**Algorithm Steps**
1. **Scene Setup**: Compute `scene_center` as the midpoint of the image dimensions: $[W/2, H/2, D/2]$. Initialize GLFW and create a window. Compile shaders for voxels, slices, texboxes, and screen display [S1, unofficial].
2. **Camera Setup**: Compute view and projection matrices using `place_camera` and `get_projection` functions, which take image dimensions, `view_distance`, and `camera_height` as inputs [S1, unofficial].
3. **Label Processing**: For each label in `labels`:
    - Find non-zero indices in the label array to identify voxel coordinates.
    - Reshape coordinates into $(N, 3)$ triples.
    - Sort the triples by the Y coordinate in descending order.
    - Flatten back to a 1D array.
    - Create a VAO and VBO. Bind the VBO, upload the sorted voxel data with `GL_STATIC_DRAW`, and set up a 3-component float vertex attribute at stride `voxels.itemsize * 3` [S1, unofficial].
4. **Render Loop**:
    - **Rotation**: Compute a Y-axis rotation matrix based on the frame count and `rotation_speed` [S1, unofficial].
    - **Slice Rendering**:
        - Extract a 2D slice from the 3D image array at index `count - countdown + 1` along the second axis [S1, unofficial].
        - Flatten the slice data and convert to 32-bit floating point.
        - Upload the flattened slice to the 2D texture using `GL_RED` format and `GL_FLOAT` type [S1, unofficial].
        - Bind the slice shader. Set view, projection, and model matrices. Set `max_height_loc_slice` to `count - countdown`. Draw the indexed triangle list using the slice index buffer [S1, unofficial].
    - **Voxel Rendering**:
        - Bind the voxel shader. Set view, projection, and model matrices. Set `max_height_loc_voxel` to `count`.
        - Iterate over each label's VAO. Set the `label_color` uniform to the corresponding color. Draw points using `GL_POINTS` [S1, unofficial].
    - **Boundary Mode**: If `only_image` is true, skip slice/voxel rendering and draw four pre-computed boundary textures using separate VAOs/VBOs [S1, unofficial].

**Pitfalls**
- The slice plane rendering uses `max_height_loc_slice` set to `count - countdown`, while voxel rendering uses `max_height_loc_voxel` set to `count`. This indicates different depth clipping or culling thresholds for slices vs points [S1, unofficial].
- When `only_image` is true, the render loop switches to drawing four pre-computed boundary textures instead of the interactive slice plane, requiring separate VAOs/VBOs for each boundary [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by CPU-side slice extraction, sorting, and GPU upload per frame. Sorting voxel coordinates is $O(N \log N)$ where $N$ is the number of non-zero voxels in the label.
- **Memory Complexity**: $O(W \times H)$ for the slice texture and $O(N)$ for voxel VBOs.
- **Strengths**: Real-time visualization of both continuous intensity data and discrete segmentation labels. Simple implementation using standard OpenGL primitives.
- **Limitations**: Not a true volume rendering technique; it relies on 2D slices and points. Performance degrades with high voxel counts due to sorting and VBO uploads. Limited to single-slice visualization unless `only_image` mode is used for boundaries.
- **When to use**: Interactive exploration of segmentation results overlaid on intensity data.
- **When not to use**: When full volumetric occlusion or lighting is required.

## Background
This method is part of the VREMS (Volume Rendering and Exploration of Medical Segmentation) project, developed by Cyril Meyer. It is designed for visualizing 3D segmentation volumes, particularly in biomedical imaging contexts like the Lucchi dataset [S1, unofficial].

## Sources
[S1] Cyril-Meyer/VREMS, https://github.com/Cyril-Meyer/VREMS, fetched 2026-10-27 (unofficial)
