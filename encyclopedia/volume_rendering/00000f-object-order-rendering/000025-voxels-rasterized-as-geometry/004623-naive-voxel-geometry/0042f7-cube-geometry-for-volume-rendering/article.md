# Cube Geometry for Volume Rendering
Also called: Axis-Aligned Cube, Voxel Cube Geometry, vzBlock Geometry

## Overview
Cube Geometry for Volume Rendering represents a simple axis-aligned cube used as a geometric container for volume data [S2]. It provides a fixed bounding primitive for the entire volume, contrasting with chunk-based or isometric approaches that subdivide or project the geometry [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data texture and camera parameters [S1, unofficial][S2].
*   **Output:** A rendered image of the volume contained within the cube's extents [S2].

**Data Structures:**
The geometry is defined by a Vertex Buffer Object (VBO) and an Element Buffer Object (EBO) [S1, unofficial].

1.  **Vertex Layout:**
    *   Total vertices: 8 (corners of the cube) [S1, unofficial].
    *   Data per vertex: 6 floats (3 for position, 3 for texture coordinates) [S1, unofficial].
    *   Stride: 24 bytes ($6 \times 4$ bytes) [S1, unofficial].
    *   Attribute 0 (Position): Offset 0, Size 3, Type `GL_FLOAT`, Normalized `GL_FALSE` [S1, unofficial].
    *   Attribute 1 (TexCoords): Offset 12 bytes ($3 \times 4$ bytes), Size 3, Type `GL_FLOAT`, Normalized `GL_FALSE` [S1, unofficial].

2.  **Index Layout:**
    *   Total indices: 36 (defining 6 faces, 2 triangles per face, 3 vertices per triangle) [S1, unofficial].
    *   Data type: `uint32` [S1, unofficial].
    *   Geometry definition: The indices define 6 faces of a cube centered at the origin with size $2 \times 2 \times 2$ (corners at $\pm 1$) [S1, unofficial].

**Parameters:**
*   **Extents:** By default, the logical extents of the cube are set to $(0, 0, 0)$ and $(1, 1, 1)$ [S2]. The geometric vertex data spans $[-1, 1]$ in each axis, requiring a model matrix or scaling transformation to map the geometry to the logical extents [G].

**Steps:**
1.  **Generate Buffers:** Generate a Vertex Array Object (VAO) and a VBO. Bind the VBO and upload the vertex data with `GL_STATIC_DRAW`. Generate an Element Buffer Object (EBO) and upload the index data with `GL_STATIC_DRAW` [S1, unofficial].
2.  **Configure Attributes:** Enable vertex attribute arrays. Attribute 0: size 3, type `GL_FLOAT`, normalized `GL_FALSE`, stride 24 bytes, offset 0. Attribute 1: size 3, type `GL_FLOAT`, normalized `GL_FALSE`, stride 24 bytes, offset 12 bytes [S1, unofficial].
3.  **Create Geometry Instance:** Instantiate the geometry object via `vzGeometry *geometry = new vzBlock()` [S2].
4.  **Link to Appearance:** Create a shape node that associates the geometry with an appearance (shader and volume texture) [S2].
5.  **Render:** Submit the shape to the render action. The shader receives uniform matrices (model, view, projection) and clipping boundaries (e.g., `clipMin`, `clipMax`) [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_cube_geometry():
    # 1. Define vertex data: 8 vertices, 6 floats each (pos + tex)
    vertices = [
        -1.0, -1.0, -1.0, 0.0, 0.0, 0.0,
         1.0, -1.0, -1.0, 1.0, 0.0, 0.0,
         1.0,  1.0, -1.0, 1.0, 1.0, 0.0,
        -1.0,  1.0, -1.0, 0.0, 1.0, 0.0,
        -1.0, -1.0,  1.0, 0.0, 0.0, 1.0,
         1.0, -1.0,  1.0, 1.0, 0.0, 1.0,
         1.0,  1.0,  1.0, 1.0, 1.0, 1.0,
        -1.0,  1.0,  1.0, 0.0, 1.0, 1.0
    ]
    # 2. Define index data: 36 indices for 6 faces
    indices = [
        0, 1, 2, 2, 3, 0,  # Back face (z=-1)
        4, 6, 5, 5, 7, 4,  # Front face (z=1)
        0, 5, 1, 1, 5, 4,  # Bottom face (y=-1)
        2, 7, 6, 6, 3, 2,  # Top face (y=1)
        3, 7, 4, 4, 0, 3,  # Left face (x=-1)
        1, 6, 7, 7, 2, 1   # Right face (x=1)
    ]
    # 3. Allocate and bind vertex buffer
    vbo_id = create_buffer(size=len(vertices) * 4, usage="STATIC_DRAW")
    bind_buffer(target="ARRAY_BUFFER", id=vbo_id)
    upload_data(buffer=vbo_id, data=vertices)
    # 4. Allocate and bind element buffer
    ebo_id = create_buffer(size=len(indices) * 4, usage="STATIC_DRAW")
    bind_buffer(target="ELEMENT_ARRAY_BUFFER", id=ebo_id)
    upload_data(buffer=ebo_id, data=indices)
    # 5. Configure vertex attributes
    enable_attribute_array(index=0)
    set_attribute_pointer(index=0, size=3, type="FLOAT", normalized=False, stride=24, offset=0)
    enable_attribute_array(index=1)
    set_attribute_pointer(index=1, size=3, type="FLOAT", normalized=False, stride=24, offset=12)
    # 6. Unbind buffers
    unbind_buffer(target="ARRAY_BUFFER")
    unbind_buffer(target="ELEMENT_ARRAY_BUFFER")
    return vbo_id, ebo_id

# Standard graphics state operations
def create_buffer(size, usage):
    return {"id": 1, "size": size, "usage": usage}
def bind_buffer(target, id):
    pass
def upload_data(buffer, data):
    pass
def enable_attribute_array(index):
    pass
def set_attribute_pointer(index, size, type, normalized, stride, offset):
    pass
def unbind_buffer(target):
    pass
```

## Operating characteristics
Time and memory complexity are constant with respect to volume resolution, as the geometry consists of a fixed 8-vertex, 36-index cube [S1, unofficial]. The method requires the shader to handle the entire volume sampling [S1, unofficial].

## Background
The method originates from the SGI OpenGL Volumizer (vz) library, specifically the `vzBlock` class, which provides a simple axis-aligned cube geometry for volume rendering applications [S2].

## Sources
[S1] GitHub - priscilla100/medical_volume_renderer, https://github.com/priscilla100/medical_volume_renderer/, fetched 2026-10-29 (unofficial)
[S2] Chapter 2. Getting Started, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch02.html, fetched 2026-10-30
