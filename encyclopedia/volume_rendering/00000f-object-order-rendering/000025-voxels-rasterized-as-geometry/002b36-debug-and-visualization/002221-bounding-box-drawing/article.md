# Bounding Box Drawing

Also called: Bounding box and mesh grid overlay, Box wireframe, BoxOverlay, Wireframe Bounding Box Visualization, debug box

## Overview
Bounding Box Drawing renders the geometric boundaries of a volume or selected voxel as wireframe lines to provide spatial context, define regions of interest (ROI), or highlight specific elements during volume rendering [S1, S3, S5, S6, unofficial]. It solves the problem of visual ambiguity in 3D scalar field data by explicitly marking extents and selection targets [S1, S5, unofficial]. Unlike [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md), which may generate static mesh primitives for collision or culling, this method focuses on real-time visualization overlays, often using line primitives (`GL_LINES`) and depth offsets to ensure visibility over the volume surface [S1, S5, S6, unofficial]. It differs from [Debug Voxel Visualization](7095-debug-voxel-visualization.md) by rendering the container boundaries or selection frames rather than individual voxel cells [S1, S5, S6, unofficial].

## How it works
**Inputs:**
- Volume dimensions or bounding box coordinates: $\mathbf{min} = (x_{min}, y_{min}, z_{min})$ and $\mathbf{max} = (x_{max}, y_{max}, z_{max})$ [S2, S4, S6, unofficial].
- Optional: ROI texture-space coordinates $t \in [0, 1]$ [S1, unofficial].
- Optional: Mesh grid division count `divs` [S1, unofficial].
- Optional: Selection voxel position `pos` and size [S5, S6, unofficial].
- Rendering parameters: Line color, thickness, depth offset [S1, S2, S5, S6, unofficial].

**Outputs:**
- A set of line segments submitted to the graphics pipeline for rasterization [S1, S3, S4, S5, S6, unofficial].

**Data Structures:**
- **Vertex Buffer (VBO):** A flat array of `float32` values representing vertex positions. For a standard AABB wireframe, this contains 24 vertices (8 corners, each repeated to form 12 line segments, i.e., 24 vertices $\times$ 3 coordinates = 72 floats) [S4, S6, unofficial]. For mesh grids, additional vertices are added for interior grid lines [S1, unofficial].
- **Vertex Array Object (VAO):** Layout specifies location 0, `vec3` position, stride 12 bytes, offset 0, type `GL_FLOAT` [S1, unofficial].
- **Transformation Matrices:** Model-view-projection (MVP) matrices for positioning the box in world space [S1, S4, S6, unofficial].

**Steps:**

1. **Compute Corner Coordinates:**
   Determine the 8 axis-aligned corners of the box.
   - If using texture-space coordinates $t \in [0, 1]$ and aspect ratio `aspect`:
     $$ \mathbf{half\_extent} = \text{aspect} \times 0.5 $$
     $$ \mathbf{corner} = (2\mathbf{t} - 1) \times \mathbf{half\_extent} $$
     where $\mathbf{t}$ maps to min/max bounds [S1, unofficial].
   - If using explicit min/max vectors $\mathbf{min}, \mathbf{max}$:
     Corners are combinations of $x \in \{x_{min}, x_{max}\}$, $y \in \{y_{min}, y_{max}\}$, $z \in \{z_{min}, z_{max}\}$ [S2, S4, S6, unofficial].
   - For a selected voxel at `pos` with size 1.0, apply a scale factor (e.g., 1.002) and translation to create a slightly larger box [S6, unofficial].

2. **Generate Edge Vertices:**
   Construct the 12 edges connecting the 8 corners.
   - **Bitwise Enumeration Method [S2, unofficial]:**
     Iterate `combinedCoord1` from 0 to 7 (binary 000 to 111).
     For each bit `shift` in $\{1, 2, 4\}$ (representing x, y, z axes):
     1. If `(combinedCoord1 & shift) != 0`, skip (this ensures each edge is drawn exactly once, transitioning from 0 to 1 in that bit position) [S2, unofficial].
     2. Compute `combinedCoord2 = combinedCoord1 + shift`.
     3. Extract coordinates for `p1` (from `combinedCoord1`) and `p2` (from `combinedCoord2`):
        For each axis $a \in \{x, y, z\}$:
        $$ p.a = \begin{cases} \mathbf{min}.a & \text{if } (\text{coord} \gg a) \& 1 == 0 \\ \mathbf{max}.a & \text{otherwise} \end{cases} $$
     4. Store pair $(\mathbf{p1}, \mathbf{p2})$ as a line segment [S2, unofficial].
   - **Explicit Loop Method [S3, unofficial]:**
     Define 6 line loops (top, bottom, and 4 vertical connectors) using vertices defined by $\pm \text{dim}/2$ [S3, unofficial].
   - **Unit Cube Method [S6, unofficial]:**
     Define 24 vertices representing 12 edges of a unit cube $(0,0,0)$ to $(1,1,1)$, which are then transformed by the model matrix [S6, unofficial].

3. **Generate Mesh Grid Lines (Optional) [S1, unofficial]:**
   If a mesh grid is requested:
   - Iterate over the 6 faces of the ROI box.
   - For each face, generate $(\text{divs}-1)$ interior lines along each face axis [S1, unofficial].

4. **Apply Transformations:**
   - **Model Space:** Translate to target position `pos` and scale if highlighting a specific voxel [S6, unofficial].
   - **View/Projection:** Multiply by camera view and projection matrices [S1, S4, unofficial].
   - **Lattice/Persp Integration [S7]:**
     Compute scaling and shifting matrices $P$ based on lattice coordinate transformations:
     $$ P = \begin{pmatrix} \text{scale} & \text{shift} \\ 0 & 1 \end{pmatrix} $$
     Update rotation matrix: $\text{rot.mat}_{new} = \text{rot.mat}_{old} \times P$ [S7].
     For perspective integration, use `trans3dz` to compute transformed coordinates including z:
     $$ \mathbf{tr} = \begin{pmatrix} x & y & z & 1 \end{pmatrix} \times \mathbf{pmat} $$
     $$ x_{out} = tr_x / tr_w, \quad y_{out} = tr_y / tr_w, \quad z_{out} = tr_z / tr_w $$
     Identify the front-most corner to redraw the bounding box correctly [S7].

5. **Set Rendering State:**
   - Disable lighting, enable blending (`GL_SRC_ALPHA`, `GL_ONE_MINUS_SRC_ALPHA`) [S1, S3, unofficial].
   - Set line width (e.g., 1.5 for box, 3.0 for selection) [S1, S3, S6, unofficial].
   - Enable line smooth if supported [S3, unofficial].
   - Set depth test to hide edges behind volume voxels, but apply a depth offset (or z-buffer bias) to prevent z-fighting/occlusion of the box lines by the volume surface [S5, S6, unofficial].

6. **Draw:**
   - Bind VAO/VBO and shader program [S4, S6, unofficial].
   - Set uniforms: color, projection, view rotation, far plane, view translation [S4, unofficial].
   - Call `glDrawArrays(GL_LINES, 0, vertex_count)` [S1, S4, S6, unofficial].
   - For vertex shader generation (no VBO), generate vertices in the vertex shader based on input parameters [S5, unofficial].

7. **Cleanup:**
   - Restore previous GL state (lighting, blend, attributes) [S3, unofficial].
   - Reset buffer lengths if using dynamic accumulation [S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def draw_bounding_box(min_v, max_v, thickness=1.0, color=(1.0, 1.0, 1.0, 0.7)):
    """
    Draws the wireframe bounding box defined by min_v and max_v.
    Uses bitwise enumeration to generate 12 edges.
    """
    # 1. Generate vertices
    vertices = []
    for coord1 in range(8):  # 0 to 7
        for shift in [1, 2, 4]:  # x, y, z bits
            # Skip if bit is already set to avoid duplicates/backwards edges
            if (coord1 & shift) != 0:
                continue
            
            coord2 = coord1 + shift
            
            # Extract p1 coordinates
            p1 = [0.0, 0.0, 0.0]
            p2 = [0.0, 0.0, 0.0]
            
            for axis_idx, axis_val in enumerate([1, 2, 4]):
                # Determine if bit is set in coord1
                bit_set = (coord1 & axis_val) != 0
                # Determine if bit is set in coord2 (it must be set if we didn't skip)
                # Actually, coord2 = coord1 + shift, so the bit corresponding to shift is 1 in coord2
                # and 0 in coord1. Other bits are same.
                
                # For p1: use min if bit is 0, max if bit is 1
                if (coord1 & axis_val):
                    p1[axis_idx] = max_v[axis_idx]
                else:
                    p1[axis_idx] = min_v[axis_idx]
                    
                # For p2: use min if bit is 0, max if bit is 1
                if (coord2 & axis_val):
                    p2[axis_idx] = max_v[axis_idx]
                else:
                    p2[axis_idx] = min_v[axis_idx]
            
            vertices.append(p1)
            vertices.append(p2)
            
    # 2. Upload to VBO (if not using vertex shader generation)
    vbo = create_vbo(vertices)
    
    # 3. Set GL State
    gl_disable(GL_LIGHTING)
    gl_enable(GL_BLEND)
    gl_blend_func(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
    gl_line_width(thickness)
    gl_color(color)
    
    # 4. Draw
    gl_bind_vao(vao)
    gl_draw_arrays(GL_LINES, 0, len(vertices))
    
    # 5. Cleanup
    gl_enable(GL_LIGHTING)
    gl_disable(GL_BLEND)
```

## Subtopics

### [Tree Topology Rendering](13035-tree-topology-rendering.md)
Tree Topology Rendering visualizes the internal hierarchical structure of an OpenVDB grid by rendering wireframe bounding boxes for every node (both leaf and non-leaf) in the tree. It serves as a debugging and visualization tool to inspect the spatial distribution and granularity of the voxel data, distinguishing itself from standard volume rendering by exposing the sparse data structure's geometry rather than the scalar field values. It builds on [Bounding Box Drawing](2733-bounding-box-drawing.md) by automating the generation of these boxes for the entire tree hierarchy.

## Operating characteristics
- **Time Complexity:** $O(1)$ for a single static bounding box, as it involves a fixed number of vertices (24) and draw calls. For mesh grids, complexity is proportional to the number of divisions $O(\text{divs}^2)$. For tree topology, it is $O(N)$ where $N$ is the number of nodes [S1, S4, unofficial].
- **Memory Complexity:** $O(1)$ for static box (72 floats). Dynamic accumulation requires temporary storage for vertices before upload [S4, unofficial].
- **Performance:** Very high, as it uses simple line primitives and minimal vertex data. Can be drawn every frame with negligible overhead [S1, S5, unofficial].
- **Strengths:** Provides immediate spatial context; lightweight; easy to implement; effective for debugging selection and ROI [S1, S5, S6, unofficial].
- **Limitations:** Does not convey internal data structure details (unless using Tree Topology variant); can be occluded by volume data without depth offset [S5, unofficial]; line width rendering quality varies across hardware [S3, unofficial].
- **When to use:** Debugging, ROI definition, selection highlighting, spatial orientation [S1, S5, S6, unofficial].
- **When not to use:** When detailed internal structure visualization is required (use Tree Topology or Debug Voxel Visualization) [S1, S5, S6, unofficial].

## Background
Bounding box drawing is a fundamental technique in computer graphics for visualizing object extents. In the context of volume rendering, it has been widely adopted in scientific visualization tools (e.g., ClearVolume, ClearVolume-plus) and voxel engines (e.g., Elysian Outpost, voxel-engine) to aid user interaction and debugging [S1, S5, S6, unofficial]. The method relies on standard OpenGL line primitives and has been adapted for various integration contexts, including lattice and perspective plotting environments in R [S7].

## Sources
[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
[S3] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S4] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
[S5] feat(#40): Find the voxel at the screen center with a voxel ray, https://github.com/nicolas-goyon/Elysian-Outpost/pull/113, fetched 2026-10-27 (unofficial)
[S6] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
[S7] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
