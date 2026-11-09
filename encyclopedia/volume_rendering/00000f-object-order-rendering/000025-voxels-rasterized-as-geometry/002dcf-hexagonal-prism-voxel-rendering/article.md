# Hexagonal Prism Voxel Rendering

Also called: HEXAGON, HexRender

## Overview
Hexagonal Prism Voxel Rendering is a geometry-based volume rendering technique that represents occupied voxels as hexagonal prisms to reduce aliasing and improve visual quality compared to cubic voxels [S1, unofficial]. It differs from cubic voxel methods by utilizing a specific hexagonal vertex layout and instanced rendering to optimize draw calls [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A voxel grid defined by integer coordinates $(x, y, z)$ within the bounds $[0, 255]$ inclusive [S1, unofficial].
*   A palette ID or texture index associated with each voxel [S1, unofficial].
*   Scene parameters including camera matrix, world transform, scale, and rotation [S1, unofficial].

**Outputs:**
*   Rasterized hexagonal prisms submitted to the graphics pipeline via an instanced draw call [S1, unofficial].

### Data Structures

**Hexagonal Prism Geometry (Static)**
The geometry consists of a single hexagonal prism unit [S1, unofficial].
*   **Vertices:** 36 vertices total. Each vertex is 6 floats: 3 for position $(x, y, z)$ and 3 for normal $(n_x, n_y, n_z)$ [S1, unofficial].
*   **Faces:** The prism comprises a top face, a bottom face, and 6 side faces [S1, unofficial].
*   **Indices:** 36 indices total [S1, unofficial].
    *   Top face indices: 0–5 [S1, unofficial].
    *   Bottom face indices: 6–11 [S1, unofficial].
    *   Side face indices: 12–35 [S1, unofficial].
*   **Local Space Coordinates:**
    *   The prism is defined in a local space where $x \in [-0.5, 1.5]$ and $y \in [0, \sqrt{3}]$ [S1, unofficial].
    *   Top face $z = 1$, Bottom face $z = 0$ [S1, unofficial].
    *   Top face normals: $(0, 0, 1)$ [S1, unofficial].
    *   Bottom face normals: $(0, 0, -1)$ [S1, unofficial].
    *   Side face normals point outward in the $xy$-plane and are calculated based on hexagon geometry involving $\sqrt{3}$ [S1, unofficial].
    *   Grid placement offset is calculated as: $\text{offset} = (1.5 \times x, \sqrt{3} \times y + (x \bmod 2) \times 0.5 \times \sqrt{3}, z)$ [S1, unofficial].

**Instance Data (Per Voxel)**
*   **Structure `MV_Voxel`:**
    *   `uint8_t x`: X coordinate (offset 0) [S1, unofficial].
    *   `uint8_t y`: Y coordinate (offset 1) [S1, unofficial].
    *   `uint8_t z`: Z coordinate (offset 2) [S1, unofficial].
    *   `uint8_t index`: Palette index (offset 3) [S1, unofficial].
    *   **Stride:** `sizeof(MV_Voxel)` (4 bytes) [S1, unofficial].
*   **Buffer:** Instance VBO (`instaceVBO`) containing one `MV_Voxel` per exposed voxel [S1, unofficial].

**Vertex Attributes (Shader Input)**
1.  Attribute 0: Position (from VBO) [S1, unofficial].
2.  Attribute 1: Normal (from VBO) [S1, unofficial].
3.  Attribute 2: Palette Index (from Instance VBO, divisor 1) [S1, unofficial].
4.  Attribute 3: Voxel Position (from Instance VBO, divisor 1) [S1, unofficial].

### Steps

1.  **Initialization:**
    *   Load voxel shape with method identifier `'hex'` [S1, unofficial].
    *   Instantiate `HexRender` object with shape and palette ID [S1, unofficial].
    *   Set texture, transform, world transform, and scale [S1, unofficial].

2.  **Voxel Trimming (`TrimVoxels`):**
    *   **Goal:** Remove internal voxels, keeping only exposed ones [S1, unofficial].
    *   **Pass 1:** Insert all non-hole voxel positions into an `unordered_set` [S1, unofficial].
    *   **Pass 2:** For each non-hole voxel, check its 6 neighbors using displacement arrays:
        *   $dx = \{-1, 1, 0, 0, 0, 0\}$
        *   $dy = \{0, 0, -1, 1, 0, 0\}$
        *   $dz = \{0, 0, 0, 0, -1, 1\}$ [S1, unofficial].
    *   **Exposure Check:** A voxel is marked as exposed if any neighbor is out of bounds ($<0$ or $>255$) or not present in the set [S1, unofficial].
    *   **Result:** Add exposed voxels to the trimmed list [S1, unofficial].

3.  **Geometry Generation:**
    *   Create Vertex Buffer Object (VBO) and Element Buffer Object (EBO) for the hexagonal prism geometry [S1, unofficial].
    *   Create Instance VBO (`instaceVBO`) populated with `MV_Voxel` data for all trimmed voxels [S1, unofficial].
    *   Link vertex attributes as defined in the Data Structures section [S1, unofficial].

4.  **Rendering (`HexRender::draw`):**
    *   Push camera matrix, scale, palette ID, size (`vec3(0,0,0)` as SM flag), color texture, position matrix, rotation matrix, world position matrix, and world rotation matrix to the shader [S1, unofficial].
    *   Bind Vertex Array Object (VAO) [S1, unofficial].
    *   Call `glDrawElementsInstanced` with:
        *   Primitive type: `GL_TRIANGLES` [S1, unofficial].
        *   Index count: 36 [S1, unofficial].
        *   Index type: `GL_UNSIGNED_INT` [S1, unofficial].
        *   Offset: 0 [S1, unofficial].
        *   Instance count: Equal to the number of trimmed voxels [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
BOUND_MIN = 0
BOUND_MAX = 255
SQRT_3 = math.sqrt(3)
GL_TRIANGLES = 4
GL_UNSIGNED_INT = 5

# Helper definitions to ensure self-contained valid Python
def create_vbo(data, stride):
    """Simulates uploading vertex/instance data to GPU memory."""
    return {"data": data, "stride": stride}

def set_uniform(name, value):
    """Simulates setting a shader uniform variable."""
    pass

def bind_vao():
    """Simulates binding the Vertex Array Object."""
    pass

def bind_vbo(vbo):
    """Simulates binding a Vertex Buffer Object."""
    pass

def gl_draw_elements_instanced(primitive_type, index_count, index_type, offset, instance_count):
    """Simulates the instanced draw call."""
    pass

def identity():
    """Returns a 4x4 identity matrix."""
    return [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]

def generate_hex_prism_geometry():
    """
    Generates the static vertex and index data for a single hexagonal prism.
    Returns vertices (list of 6-float tuples) and indices (list of ints).
    Matches source layout: 36 vertices, 36 indices.
    """
    vertices = []
    indices = []
    
    def add_vertex(x, y, z, nx, ny, nz):
        vertices.append((x, y, z, nx, ny, nz))
        
    # Top Face (z=1, normal 0,0,1) - 6 vertices, indices 0-5
    top_y_base = SQRT_3
    for i in range(6):
        angle = i * math.pi / 3
        # Standard hexagon vertices mapped to local space bounds
        x = 0.5 + 0.5 * math.cos(angle)
        y = 0.5 * SQRT_3 + 0.5 * SQRT_3 * math.sin(angle)
        add_vertex(x, y, 1, 0, 0, 1)
        
    # Bottom Face (z=0, normal 0,0,-1) - 6 vertices, indices 6-11
    for i in range(6):
        angle = i * math.pi / 3
        x = 0.5 + 0.5 * math.cos(angle)
        y = 0.5 * SQRT_3 + 0.5 * SQRT_3 * math.sin(angle)
        add_vertex(x, y, 0, 0, 0, -1)
        
    # Side Faces - 24 vertices (4 per face * 6 faces), indices 12-35
    # Each side face connects a top edge to a bottom edge
    for i in range(6):
        angle_curr = i * math.pi / 3
        angle_next = ((i + 1) % 6) * math.pi / 3
        
        x1 = 0.5 + 0.5 * math.cos(angle_curr)
        y1 = 0.5 * SQRT_3 + 0.5 * SQRT_3 * math.sin(angle_curr)
        x2 = 0.5 + 0.5 * math.cos(angle_next)
        y2 = 0.5 * SQRT_3 + 0.5 * SQRT_3 * math.sin(angle_next)
        
        # Calculate outward normal for the side face
        nx = math.cos((angle_curr + angle_next) / 2)
        ny = math.sin((angle_curr + angle_next) / 2)
        
        # Quad split into 2 triangles (6 indices per face)
        base_idx = 12 + i * 6
        # Triangle 1
        indices.extend([base_idx, base_idx+1, base_idx+2])
        indices.extend([base_idx, base_idx+2, base_idx+3])
        # Triangle 2
        indices.extend([base_idx, base_idx+3, base_idx+4])
        indices.extend([base_idx, base_idx+4, base_idx+5])
        
        # Add 6 vertices for this side face (top 3, bottom 3)
        add_vertex(x1, y1, 1, nx, ny, 0)
        add_vertex(x2, y2, 1, nx, ny, 0)
        add_vertex(x1, y1, 0, nx, ny, 0)
        add_vertex(x2, y2, 0, nx, ny, 0)
        add_vertex(x1, y1, 1, nx, ny, 0)
        add_vertex(x2, y2, 1, nx, ny, 0)
        
    return vertices, indices

def trim_voxels(voxel_set):
    """
    Removes internal voxels by keeping only exposed ones.
    voxel_set: set of (x,y,z) tuples representing non-hole voxels.
    Returns: list of exposed (x,y,z) tuples.
    """
    exposed = []
    dx = [-1, 1, 0, 0, 0, 0]
    dy = [0, 0, -1, 1, 0, 0]
    dz = [0, 0, 0, 0, -1, 1]
    
    for x, y, z in voxel_set:
        is_exposed = False
        for i in range(6):
            nx, ny, nz = x + dx[i], y + dy[i], z + dz[i]
            # Check bounds
            if nx < BOUND_MIN or nx > BOUND_MAX or \
               ny < BOUND_MIN or ny > BOUND_MAX or \
               nz < BOUND_MIN or nz > BOUND_MAX:
                is_exposed = True
                break
            # Check existence
            if (nx, ny, nz) not in voxel_set:
                is_exposed = True
                break
        if is_exposed:
            exposed.append((x, y, z))
            
    return exposed

def render_hex_voxels(trimmed_voxels, camera_matrix, world_transform, scale, palette_id, texture):
    """
    Renders the trimmed voxels as hexagonal prisms.
    """
    # 1. Prepare Instance Data
    instance_data = []
    for x, y, z in trimmed_voxels:
        # Clamp coordinates to 0-255
        x = max(BOUND_MIN, min(BOUND_MAX, x))
        y = max(BOUND_MIN, min(BOUND_MAX, y))
        z = max(BOUND_MIN, min(BOUND_MAX, z))
        instance_data.append({
            'x': x,
            'y': y,
            'z': z,
            'index': palette_id
        })
        
    # 2. Upload Instance VBO
    instaceVBO = create_vbo(instance_data, stride=4)
    
    # 3. Setup Shader Uniforms
    set_uniform('MV_Voxel', camera_matrix)
    set_uniform('scale', scale)
    set_uniform('paletteID', palette_id)
    set_uniform('size', (0, 0, 0))
    set_uniform('colorTexture', texture)
    set_uniform('positionMatrix', identity())
    set_uniform('rotationMatrix', identity())
    set_uniform('worldPositionMatrix', world_transform)
    set_uniform('worldRotationMatrix', identity())
    
    # 4. Draw
    bind_vao()
    bind_vbo(instaceVBO)
    gl_draw_elements_instanced(
        primitive_type=GL_TRIANGLES,
        index_count=36,
        index_type=GL_UNSIGNED_INT,
        offset=0,
        instance_count=len(trimmed_voxels)
    )
```

### Parameters
*   **Method Identifier:** `'hex'` (string) in XML scene file attribute `name` [S1, unofficial].
*   **Voxel Bounds:** $x, y, z \in [0, 255]$ [S1, unofficial].
*   **Scale:** `vec3(0,0,0)` used as a flag in the shader [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Clamping:** Voxel coordinates must be clamped to the $[0, 255]$ range [S1, unofficial]. Neighbors outside this range are considered exposed, causing the boundary voxel to be rendered [S1, unofficial].
*   **Index Count Discrepancy:** The source specifies 36 indices, which form 12 triangles (the source notes '18 triangles', which is mathematically inconsistent with 36 indices) [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
*   **Origin:** This method is described in the unofficial repository `TTFH/Voxel-Render` [S1, unofficial].
*   **Usage:** Used in the context of the `Voxel-Render` project for rendering voxel shapes with improved visual quality [S1, unofficial].

## Sources

[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
