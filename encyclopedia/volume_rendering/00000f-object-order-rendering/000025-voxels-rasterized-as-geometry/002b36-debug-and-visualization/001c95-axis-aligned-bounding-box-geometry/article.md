# Axis-Aligned Bounding Box Geometry

Also called: AABB Geometry, RenderClippedBoundingBox, Volume Extent Bounding Box

## Overview
Renders the clipped bounding box geometry of a volume extent to the framebuffer, optionally including texture coordinates [S1, unofficial]. It defines the wireframe or solid outline of the volume using a fixed set of vertices representing the 6 faces of the box [S2, unofficial]. Unlike sibling methods that may draw arbitrary debug lines or dynamic bounds, this method specifically targets the static volume extent defined by minimum and maximum corners [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `currentBlock`: `size_t` index of the current rendering block [S1, unofficial].
- `numberOfBlocks`: `size_t` total number of blocks to process [S1, unofficial].
- `tcoordFlag`: Boolean/int flag indicating whether to render texture coordinates [S1, unofficial].
- `renWin`: Pointer to the render window context [S1, unofficial].
- `minB`: Minimum corner of the bounding box, typically `(-1.0, -1.0, -0.42578125)` [S2, unofficial].
- `maxB`: Maximum corner of the bounding box, typically `(1.0, 1.0, 0.42578125)` [S2, unofficial].

**Outputs:**
- Framebuffer with the bounding box geometry rendered [S1, unofficial].
- Boolean return value: `true` if rendering should abort, `false` otherwise [S1, unofficial].

**Data Structures:**
- **Vertices Array**: A flat array of 36 floats (12 triangles $\times$ 3 vertices $\times$ 3 components) defining the 6 faces of the box [S2, unofficial]. The vertices are hardcoded to represent the AABB defined by `minB` and `maxB`.
- **VAO (Vertex Array Object)**: Stores the vertex attribute configuration [S2, unofficial].
- **VBO (Vertex Buffer Object)**: Stores the vertex data in GPU memory [S2, unofficial].

**Steps:**
1. Validate that `currentBlock` is within the range `[0, numberOfBlocks)` [S1, unofficial]. If invalid, abort rendering [S1, unofficial].
2. Generate a Vertex Array Object (VAO) and a Vertex Buffer Object (VBO) using standard OpenGL calls [S2, unofficial].
3. Upload the hardcoded vertex data (36 floats) to the VBO using `GL_STATIC_DRAW` [S2, unofficial].
4. Set the vertex attribute pointer for position (attribute index 0) to read 3 floats per vertex, with no stride and no offset [S2, unofficial].
5. Enable the vertex attribute array [S2, unofficial].
6. Render the geometry (typically as lines or triangles) to the framebuffer [S1, unofficial].
7. Return `true` if rendering should abort, `false` otherwise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_aabb_geometry(current_block, number_of_blocks, tcoord_flag, ren_win, min_b, max_b):
    # Step 1: Validate block index
    if current_block >= number_of_blocks:
        return True  # Abort rendering

    # Step 2: Generate VAO and VBO
    vao = glGen_vertex_arrays(1)
    vbo = glGen_buffers(1)

    # Step 3: Define vertices for AABB
    # 6 faces, 2 triangles per face, 3 vertices per triangle = 36 vertices
    # Coordinates are derived from min_b and max_b
    # Example for min_b=(-1,-1,-0.42578125), max_b=(1,1,0.42578125)
    vertices = [
        # Front face (z = max_b.z)
        min_b.x, min_b.y, max_b.z,  max_b.x, min_b.y, max_b.z,  max_b.x, max_b.y, max_b.z,
        min_b.x, min_b.y, max_b.z,  max_b.x, max_b.y, max_b.z,  min_b.x, max_b.y, max_b.z,
        # Back face (z = min_b.z)
        min_b.x, min_b.y, min_b.z,  min_b.x, max_b.y, min_b.z,  max_b.x, max_b.y, min_b.z,
        min_b.x, min_b.y, min_b.z,  max_b.x, max_b.y, min_b.z,  max_b.x, min_b.y, min_b.z,
        # Left face (x = min_b.x)
        min_b.x, min_b.y, min_b.z,  min_b.x, max_b.y, min_b.z,  min_b.x, max_b.y, max_b.z,
        min_b.x, min_b.y, min_b.z,  min_b.x, max_b.y, max_b.z,  min_b.x, min_b.y, max_b.z,
        # Right face (x = max_b.x)
        max_b.x, min_b.y, min_b.z,  max_b.x, max_b.y, min_b.z,  max_b.x, max_b.y, max_b.z,
        max_b.x, min_b.y, min_b.z,  max_b.x, max_b.y, max_b.z,  max_b.x, min_b.y, max_b.z,
        # Bottom face (y = min_b.y)
        min_b.x, min_b.y, min_b.z,  max_b.x, min_b.y, min_b.z,  max_b.x, min_b.y, max_b.z,
        min_b.x, min_b.y, min_b.z,  max_b.x, min_b.y, max_b.z,  min_b.x, min_b.y, max_b.z,
        # Top face (y = max_b.y)
        min_b.x, max_b.y, min_b.z,  min_b.x, max_b.y, max_b.z,  max_b.x, max_b.y, max_b.z,
        min_b.x, max_b.y, min_b.z,  max_b.x, max_b.y, max_b.z,  max_b.x, max_b.y, min_b.z
    ]

    # Step 4: Upload vertex data
    glBind_buffer(GL_ARRAY_BUFFER, vbo)
    gl_buffer_data(GL_ARRAY_BUFFER, len(vertices) * 4, vertices, GL_STATIC_DRAW)

    # Step 5: Set vertex attribute pointer
    glBind_vertex_array(vao)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * 4, None)
    glEnableVertexAttribArray(0)

    # Step 6: Render
    # Note: Texture coordinates are not explicitly generated in the source notes,
    # but tcoordFlag suggests they might be used if available.
    # Assuming rendering as triangles or lines based on the 36-vertex structure.
    gl_draw_arrays(GL_TRIANGLES, 0, 36)

    # Step 7: Cleanup (optional, depending on context)
    # gl_delete_buffers(1, vbo)
    # gl_delete_vertex_arrays(1, vao)

    return False  # Continue rendering
```

**Edge Cases and Pitfalls:**
- **Precision**: The z-coordinate `0.42578125` is a specific value from the source [S2, unofficial]. Using floating-point comparisons for bounds validation should account for epsilon if dynamic bounds are used [G].
- **Boundaries**: The method assumes a fixed AABB. If the volume extent changes, the vertex data must be regenerated [S2, unofficial].
- **Degenerate Input**: If `min_b` equals `max_b`, the box has zero volume and will not render visibly [G].

## Subtopics

### [Bounding Box Drawing](2733-bounding-box-drawing.md)
Draws arbitrary bounding boxes for debugging. Choose when you need to visualize dynamic or non-axis-aligned bounds.

## Operating characteristics
- **Time Complexity**: $O(1)$ for geometry generation and upload, as the vertex count is fixed at 36 floats [S2, unofficial].
- **Memory Complexity**: $O(1)$, requiring storage for 36 floats and one VAO/VBO pair [S2, unofficial].
- **Strengths**: Simple, fast, and deterministic. Ideal for visualizing the static extent of a volume block [S1, unofficial][S2, unofficial].
- **Limitations**: Does not adapt to dynamic volume changes without regeneration. Only renders axis-aligned boxes [S2, unofficial].
- **When to use**: Debugging volume rendering by visualizing the extent of individual blocks or the entire volume [S1, unofficial].
- **When not to use**: When dynamic, non-axis-aligned, or complex bounding volumes are required [S2, unofficial].

## Background
This method is derived from implementations in VTK-based volume ray casters and custom ray tracing engines [S1, unofficial][S2, unofficial]. It is used in debug and visualization contexts to provide spatial reference for volume data [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
