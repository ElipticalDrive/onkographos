# Block face vertex layout

Also called: Block.h block_faces, Cube Geometry Definition, Cube face vertices

## Overview
Block face vertex layout defines the static geometric primitives for a unit cube, serving as the fundamental mesh template for blocky meshing algorithms. It specifies the positions, colors, and texture coordinates for the six faces of a cube, allowing surface extraction methods to generate meshes by instantiating these faces for visible voxel boundaries. It differs from sibling methods like [Greedy Meshing](1776-greedy-meshing.md) or [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by providing only the raw vertex data for individual faces without performing topology optimization, face culling, or merging of adjacent coplanar faces.

## How it works
The method constructs a unit cube mesh centered at the origin or defined in local space, consisting of 6 faces. Each face is composed of 2 triangles, resulting in 6 vertices per face and 36 vertices total for the complete cube. The layout supports instanced rendering where the base geometry is constant, and instance attributes (position, color, texture index) vary per voxel.

### Inputs and Outputs
- **Input**: None (static definition).
- **Output**: Vertex buffer data containing positions, colors, and UV coordinates for 36 vertices.

### Data Structures
Two primary data structures define the geometry. The first, used in [S1, unofficial], defines a unit cube with corners at $(0,0,0)$ to $(1,1,1)$ and uses an index-based quad-to-triangle mapping. The second, used in [S2, unofficial], defines a cube centered at $(0,0,0)$ with corners at $\pm 0.5$ and uses a flat array of explicit vertex attributes.

#### Structure A: Index-Based Unit Cube (Source [S1, unofficial])
This structure defines 8 unique vertices and maps them to 12 triangles (6 quads) via indices.

1.  **Vertices Array (`Cube.vertices`)**: 8 `Vector3` elements.
    -   Index 0: $(0, 0, 0)$
    -   Index 1: $(1, 0, 0)$
    -   Index 2: $(1, 1, 0)$
    -   Index 3: $(0, 1, 0)$
    -   Index 4: $(0, 0, 1)$
    -   Index 5: $(1, 0, 1)$
    -   Index 6: $(1, 1, 1)$
    -   Index 7: $(0, 1, 1)$

2.  **UV Offsets Array (`Cube.uvOffsets`)**: 4 `Vector2` elements.
    -   Index 0: $(0, 0)$
    -   Index 1: $(0, 1)$
    -   Index 2: $(1, 0)$
    -   Index 3: $(1, 1)$

3.  **Quad Index Mapping (`Cube.quads`)**: Flattened array of 24 integers, representing 6 quads (4 indices each). The order corresponds to faces: Front (Z+), Back (Z-), Right (X+), Left (X-), Top (Y+), Bottom (Y-).
    -   Values: `[0, 3, 1, 2, 5, 6, 4, 7, 3, 7, 2, 6, 1, 5, 0, 4, 4, 7, 0, 3, 1, 2, 5, 6]`
    -   Face 0 (Front, Z+): Indices `0, 3, 1, 2`
    -   Face 1 (Back, Z-): Indices `5, 6, 4, 7`
    -   Face 2 (Right, X+): Indices `3, 7, 2, 6`
    -   Face 3 (Left, X-): Indices `1, 5, 0, 4`
    -   Face 4 (Top, Y+): Indices `4, 7, 0, 3`
    -   Face 5 (Bottom, Y-): Indices `1, 2, 5, 6`

4.  **Triangle Offset Mapping (`Cube.quadsOffsetForTris`)**: Array of 6 integers mapping quad indices to triangle vertices.
    -   Values: `[0, 1, 2, 2, 1, 3]`
    -   This allows iterating $t$ from 0 to 5 to get the two triangles for each quad. For a given face $f$, the triangle vertices are derived from `quads[4*f + quadsOffsetForTris[t]]`.

#### Structure B: Explicit Vertex Attribute Cube (Source [S2, unofficial])
This structure defines 36 explicit vertices (6 per face), each containing Position (`vec3`), Color (`vec3`), and Texture Coordinate (`vec2`). The cube is centered at $(0,0,0)$ with side length 1 (range $[-0.5, 0.5]$).

The array `block_faces[6][6]` stores the vertices. The faces are ordered: Back (Z-), Front (Z+), Left (X-), Right (X+), Bottom (Y-), Top (Y+).

1.  **Back Face (Z-)**:
    -   Vertices:
        1.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 0)$
        2.  Pos: $(0.5, 0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 1)$
        3.  Pos: $(0.5, -0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 0)$
        4.  Pos: $(0.5, 0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 1)$
        5.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 0)$
        6.  Pos: $(-0.5, 0.5, -0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 1)$

2.  **Front Face (Z+)**:
    -   Vertices:
        1.  Pos: $(-0.5, -0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 0)$
        2.  Pos: $(0.5, -0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 0)$
        3.  Pos: $(0.5, 0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 1)$
        4.  Pos: $(0.5, 0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(1, 1)$
        5.  Pos: $(-0.5, 0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 1)$
        6.  Pos: $(-0.5, -0.5, 0.5)$, Color: $(0.9, 0.9, 0.9)$, UV: $(0, 0)$

3.  **Left Face (X-)**:
    -   Vertices:
        1.  Pos: $(-0.5, 0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 0)$
        2.  Pos: $(-0.5, 0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 1)$
        3.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 1)$
        4.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 1)$
        5.  Pos: $(-0.5, -0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 0)$
        6.  Pos: $(-0.5, 0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 0)$

4.  **Right Face (X+)**:
    -   Vertices:
        1.  Pos: $(0.5, 0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 0)$
        2.  Pos: $(0.5, -0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 1)$
        3.  Pos: $(0.5, 0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 1)$
        4.  Pos: $(0.5, -0.5, -0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 1)$
        5.  Pos: $(0.5, 0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(0, 0)$
        6.  Pos: $(0.5, -0.5, 0.5)$, Color: $(0.8, 0.8, 0.8)$, UV: $(1, 0)$

5.  **Bottom Face (Y-)**:
    -   Vertices:
        1.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 1)$
        2.  Pos: $(0.5, -0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 1)$
        3.  Pos: $(0.5, -0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 0)$
        4.  Pos: $(0.5, -0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 0)$
        5.  Pos: $(-0.5, -0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 0)$
        6.  Pos: $(-0.5, -0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 1)$

6.  **Top Face (Y+)**:
    -   Vertices:
        1.  Pos: $(-0.5, 0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 1)$
        2.  Pos: $(0.5, 0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 0)$
        3.  Pos: $(0.5, 0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 1)$
        4.  Pos: $(0.5, 0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(1, 0)$
        5.  Pos: $(-0.5, 0.5, -0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 1)$
        6.  Pos: $(-0.5, 0.5, 0.5)$, Color: $(1.0, 1.0, 1.0)$, UV: $(0, 0)$

### Pseudocode
The following Python-like pseudocode implements the generation of the explicit vertex layout (Structure B) as described in [S2, unofficial]. It creates a list of 36 vertices, each with position, color, and UV.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_block_face_vertices():
    """
    Generates the 36 vertices for a unit cube centered at (0,0,0).
    Returns a list of 36 vertex dictionaries, each containing 'pos', 'color', and 'uv'.
    Order: Back (Z-), Front (Z+), Left (X-), Right (X+), Bottom (Y-), Top (Y+).
    """
    vertices = []

    # Helper to add a vertex
    def add_vertex(x, y, z, r, g, b, u, v):
        vertices.append({
            'pos': (x, y, z),
            'color': (r, g, b),
            'uv': (u, v)
        })

    # Back Face (Z-)
    # Vertices: (-0.5,-0.5,-0.5), (0.5,0.5,-0.5), (0.5,-0.5,-0.5), (0.5,0.5,-0.5), (-0.5,-0.5,-0.5), (-0.5,0.5,-0.5)
    # UVs: (0,0), (1,1), (1,0), (1,1), (0,0), (0,1)
    # Color: (0.9, 0.9, 0.9)
    add_vertex(-0.5, -0.5, -0.5, 0.9, 0.9, 0.9, 0.0, 0.0)
    add_vertex(0.5, 0.5, -0.5, 0.9, 0.9, 0.9, 1.0, 1.0)
    add_vertex(0.5, -0.5, -0.5, 0.9, 0.9, 0.9, 1.0, 0.0)
    add_vertex(0.5, 0.5, -0.5, 0.9, 0.9, 0.9, 1.0, 1.0)
    add_vertex(-0.5, -0.5, -0.5, 0.9, 0.9, 0.9, 0.0, 0.0)
    add_vertex(-0.5, 0.5, -0.5, 0.9, 0.9, 0.9, 0.0, 1.0)

    # Front Face (Z+)
    # Vertices: (-0.5,-0.5,0.5), (0.5,-0.5,0.5), (0.5,0.5,0.5), (0.5,0.5,0.5), (-0.5,0.5,0.5), (-0.5,-0.5,0.5)
    # UVs: (0,0), (1,0), (1,1), (1,1), (0,1), (0,0)
    # Color: (0.9, 0.9, 0.9)
    add_vertex(-0.5, -0.5, 0.5, 0.9, 0.9, 0.9, 0.0, 0.0)
    add_vertex(0.5, -0.5, 0.5, 0.9, 0.9, 0.9, 1.0, 0.0)
    add_vertex(0.5, 0.5, 0.5, 0.9, 0.9, 0.9, 1.0, 1.0)
    add_vertex(0.5, 0.5, 0.5, 0.9, 0.9, 0.9, 1.0, 1.0)
    add_vertex(-0.5, 0.5, 0.5, 0.9, 0.9, 0.9, 0.0, 1.0)
    add_vertex(-0.5, -0.5, 0.5, 0.9, 0.9, 0.9, 0.0, 0.0)

    # Left Face (X-)
    # Vertices: (-0.5,0.5,0.5), (-0.5,0.5,-0.5), (-0.5,-0.5,-0.5), (-0.5,-0.5,-0.5), (-0.5,-0.5,0.5), (-0.5,0.5,0.5)
    # UVs: (1,0), (1,1), (0,1), (0,1), (0,0), (1,0)
    # Color: (0.8, 0.8, 0.8)
    add_vertex(-0.5, 0.5, 0.5, 0.8, 0.8, 0.8, 1.0, 0.0)
    add_vertex(-0.5, 0.5, -0.5, 0.8, 0.8, 0.8, 1.0, 1.0)
    add_vertex(-0.5, -0.5, -0.5, 0.8, 0.8, 0.8, 0.0, 1.0)
    add_vertex(-0.5, -0.5, -0.5, 0.8, 0.8, 0.8, 0.0, 1.0)
    add_vertex(-0.5, -0.5, 0.5, 0.8, 0.8, 0.8, 0.0, 0.0)
    add_vertex(-0.5, 0.5, 0.5, 0.8, 0.8, 0.8, 1.0, 0.0)

    # Right Face (X+)
    # Vertices: (0.5,0.5,0.5), (0.5,-0.5,-0.5), (0.5,0.5,-0.5), (0.5,-0.5,-0.5), (0.5,0.5,0.5), (0.5,-0.5,0.5)
    # UVs: (1,0), (0,1), (1,1), (0,1), (0,0), (1,0)
    # Color: (0.8, 0.8, 0.8)
    add_vertex(0.5, 0.5, 0.5, 0.8, 0.8, 0.8, 1.0, 0.0)
    add_vertex(0.5, -0.5, -0.5, 0.8, 0.8, 0.8, 0.0, 1.0)
    add_vertex(0.5, 0.5, -0.5, 0.8, 0.8, 0.8, 1.0, 1.0)
    add_vertex(0.5, -0.5, -0.5, 0.8, 0.8, 0.8, 0.0, 1.0)
    add_vertex(0.5, 0.5, 0.5, 0.8, 0.8, 0.8, 0.0, 0.0)
    add_vertex(0.5, -0.5, 0.5, 0.8, 0.8, 0.8, 1.0, 0.0)

    # Bottom Face (Y-)
    # Vertices: (-0.5,-0.5,-0.5), (0.5,-0.5,-0.5), (0.5,-0.5,0.5), (0.5,-0.5,0.5), (-0.5,-0.5,0.5), (-0.5,-0.5,-0.5)
    # UVs: (0,1), (1,1), (1,0), (1,0), (0,0), (0,1)
    # Color: (1.0, 1.0, 1.0)
    add_vertex(-0.5, -0.5, -0.5, 1.0, 1.0, 1.0, 0.0, 1.0)
    add_vertex(0.5, -0.5, -0.5, 1.0, 1.0, 1.0, 1.0, 1.0)
    add_vertex(0.5, -0.5, 0.5, 1.0, 1.0, 1.0, 1.0, 0.0)
    add_vertex(0.5, -0.5, 0.5, 1.0, 1.0, 1.0, 1.0, 0.0)
    add_vertex(-0.5, -0.5, 0.5, 1.0, 1.0, 1.0, 0.0, 0.0)
    add_vertex(-0.5, -0.5, -0.5, 1.0, 1.0, 1.0, 0.0, 1.0)

    # Top Face (Y+)
    # Vertices: (-0.5,0.5,-0.5), (0.5,0.5,0.5), (0.5,0.5,-0.5), (0.5,0.5,0.5), (-0.5,0.5,-0.5), (-0.5,0.5,0.5)
    # UVs: (0,1), (1,0), (1,1), (1,0), (0,1), (0,0)
    # Color: (1.0, 1.0, 1.0)
    add_vertex(-0.5, 0.5, -0.5, 1.0, 1.0, 1.0, 0.0, 1.0)
    add_vertex(0.5, 0.5, 0.5, 1.0, 1.0, 1.0, 1.0, 0.0)
    add_vertex(0.5, 0.5, -0.5, 1.0, 1.0, 1.0, 1.0, 1.0)
    add_vertex(0.5, 0.5, 0.5, 1.0, 1.0, 1.0, 1.0, 0.0)
    add_vertex(-0.5, 0.5, -0.5, 1.0, 1.0, 1.0, 0.0, 1.0)
    add_vertex(-0.5, 0.5, 0.5, 1.0, 1.0, 1.0, 0.0, 0.0)

    return vertices
```

### Parameters and Edge Cases
- **Cube Size**: The sources define a unit cube (side length 1). Scaling is applied via instance transformation matrices or vertex shader uniforms.
- **Centering**: Source [S1, unofficial] uses $[0,1]^3$, while [S2, unofficial] uses $[-0.5, 0.5]^3$. Implementers must ensure consistency with their coordinate system.
- **Winding Order**: The vertex order in [S2, unofficial] ensures consistent front-face winding (counter-clockwise) for all faces when viewed from the outside.
- **Color Variation**: [S2, unofficial] assigns different base colors to different faces (Z-faces: 0.9, X-faces: 0.8, Y-faces: 1.0). This is likely for debugging or simple shading. In production, colors are typically passed via instance attributes or texture sampling.
- **UV Mapping**: The UVs in [S2, unofficial] are not standard planar projections for all faces; they are specific to the face's orientation to ensure texture alignment. For example, the Left and Right faces have UVs that may require rotation or flipping depending on the texture atlas layout.

## Operating characteristics
- **Time Complexity**: $O(1)$ to generate the static vertex buffer.
- **Memory Complexity**: $O(1)$ for the static mesh data (36 vertices).
- **Performance**: Rendering requires one draw call per visible face if not instanced, or one instanced draw call per chunk/region if instancing is used. The lack of face culling in the layout itself means the mesh generation step (handled by parent methods like [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md)) determines the actual vertex count sent to the GPU.
- **Strengths**: Simple to implement, minimal memory overhead for the mesh definition, supports instancing efficiently.
- **Limitations**: High vertex count for large scenes if not culled, no topology optimization (no greedy meshing), potential for overdraw if backfaces are not culled.

## Background
The block face vertex layout is a standard technique in voxel rendering, particularly in games like Minecraft and its clones. It provides a simple, unoptimized mesh for each voxel face. The specific implementations in [S1, unofficial] and [S2, unofficial] reflect common patterns in open-source voxel engines, where the geometry is defined as a static constant to minimize runtime allocation.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S2] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
