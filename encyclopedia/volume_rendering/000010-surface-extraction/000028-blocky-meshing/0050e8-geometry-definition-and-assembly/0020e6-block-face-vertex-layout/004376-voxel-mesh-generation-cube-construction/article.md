# Voxel Mesh Generation (Cube Construction)

Also called: CreateQuadSmoothVoxel, CreateQuadVoxel

## Overview
Voxel Mesh Generation (Cube Construction) constructs a static, axis-aligned cube mesh for a single voxel block by explicitly defining its geometry, normals, and per-vertex attributes. It solves the problem of rendering individual voxel units as distinct geometric primitives with specific material properties, such as texture tiles and ambient occlusion colors. Unlike smooth surface extraction methods that merge vertices to create continuous surfaces, this method duplicates vertices for each face to support independent face normals and per-vertex colors, resulting in a "blocky" aesthetic. It builds on the vertex data layout defined in [Block face vertex layout](2374-block-face-vertex-layout.md).

## How it works

### Inputs and Outputs
**Inputs:**
*   `relativePosition`: The center coordinates $(x, y, z)$ of the voxel block in world space.
*   `BLOCK_SIZE`: The half-extent of the voxel cube (distance from center to face).
*   `blockType`: An enumeration identifying the voxel material (e.g., grass, dirt, stone).
*   `atlas_size_in_tiles`: The number of tiles along one axis of the texture atlas (typically 16).
*   `texture_atlas`: A square texture containing the tile images.
*   `material`: A material object with an assigned `albedo_texture` pointing to the atlas.

**Outputs:**
*   A mesh buffer containing 24 vertices, 24 normals, and 24 colors per voxel.
*   Index data defining 6 quads (12 triangles) per voxel.

### Data Structures and Layout
The mesh stores data sequentially per voxel. For each voxel, the layout is as follows:

1.  **Vertices**: 24 vertices total. Each of the 6 faces uses 4 unique vertices.
    *   Each vertex contains position $(x, y, z)$, texture coordinates $(u, v)$, and color $(r, g, b, a)$.
    *   Texture coordinates are derived from the tile indices assigned to each face and the `atlas_size_in_tiles`.
2.  **Normals**: 24 normals total. One normal is duplicated for each of the 4 vertices of a face.
    *   Normals are axis-aligned unit vectors pointing outward from the cube center.
3.  **Colors**: 24 colors total. One color is assigned to each vertex.
    *   Colors are determined by the `blockType` and are used as vertex colors (often multiplied with the albedo texture).
4.  **Indices**: 36 indices total (6 quads $\times$ 2 triangles $\times$ 3 vertices).
    *   Indices are calculated using a stride offset based on the voxel counter.

### Parameters
*   `BLOCK_SIZE`: Half-extent of the cube. Typical value: $0.5$ (if the cube spans $[-1, 1]$) or $1.0$ (if the cube spans $[-2, 2]$). The sources use `BLOCK_SIZE` as an offset from the center.
*   `atlas_size_in_tiles`: The width/height of the atlas in tiles. Typical value: $16$.
*   `stride`: The byte or vertex offset for the current voxel. Calculated as $24 \times \text{voxelCounter}$.

### Step-by-Step Method

1.  **Calculate Corner Points**:
    Compute the 8 corner points of the cube relative to `relativePosition` $(cx, cy, cz)$ using `BLOCK_SIZE` ($s$). The points are defined as:
    $$
    \begin{aligned}
    P_1 &= (cx - s, cy - s, cz + s) \\
    P_2 &= (cx + s, cy - s, cz + s) \\
    P_3 &= (cx + s, cy + s, cz + s) \\
    P_4 &= (cx - s, cy + s, cz + s) \\
    P_5 &= (cx - s, cy - s, cz - s) \\
    P_6 &= (cx + s, cy - s, cz - s) \\
    P_7 &= (cx + s, cy + s, cz - s) \\
    P_8 &= (cx - s, cy + s, cz - s)
    \end{aligned}
    $$
    *Note: The coordinate system assumes Z+ is Front, Z- is Back, Y+ is Top, Y- is Bottom, X+ is Right, X- is Left.*

2.  **Determine Face Vertices**:
    Assign the 4 corner points to each of the 6 faces based on the vertex layout from [Block face vertex layout](2374-block-face-vertex-layout.md).
    *   **Front (Z+)**: $P_1, P_2, P_3, P_4$
    *   **Back (Z-)**: $P_5, P_6, P_7, P_8$
    *   **Right (X+)**: $P_2, P_6, P_7, P_3$
    *   **Left (X-)**: $P_1, P_5, P_8, P_4$
    *   **Top (Y+)**: $P_4, P_3, P_7, P_8$
    *   **Bottom (Y-)**: $P_1, P_5, P_6, P_2$

3.  **Assign Texture Coordinates**:
    For each face, determine the tile coordinates $(tx, ty)$ from the `Cube tiles` properties of the block type. Convert these to UV coordinates:
    $$
    u = \frac{tx + 0.5}{\text{atlas\_size\_in\_tiles}}, \quad v = \frac{ty + 0.5}{\text{atlas\_size\_in\_tiles}}
    $$
    *Note: The sources state coordinates are in tiles, not pixels. The exact UV mapping for the 4 corners of a quad depends on the specific face orientation and the atlas layout, but typically each corner gets a UV offset of $\pm \frac{1}{\text{atlas\_size\_in\_tiles}}$ from the center tile UV.*

4.  **Assign Colors**:
    Set the per-vertex color based on `blockType`. The sources provide specific RGB values (with alpha 1.0):
    *   `BLOCK_GRASS`: $(0.0, 1.0, 0.0, 1.0)$
    *   `BLOCK_DIRT`: $(0.435, 0.305, 0.215, 1.0)$
    *   `BLOCK_STONE`: $(0.45, 0.45, 0.45, 1.0)$
    *   All 4 vertices of a face receive the same color.

5.  **Assign Normals**:
    Add the axis-aligned normal for each of the 4 vertices of each face.
    *   Front: $(0, 0, 1)$
    *   Back: $(0, 0, -1)$
    *   Right: $(1, 0, 0)$
    *   Left: $(-1, 0, 0)$
    *   Top: $(0, 1, 0)$
    *   Bottom: $(0, -1, 0)$

6.  **Add Indices**:
    Add indices for the 6 quads. Each quad consists of 2 triangles (6 indices). The indices are relative to the start of the voxel's vertex buffer segment.
    *   Stride offset: $24 \times \text{voxelCounter}$.
    *   Example for Front face (vertices 0-3): Indices $0, 1, 2, 0, 2, 3$ (plus stride offset).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(voxel_counter, relative_position, block_size, block_type, atlas_size_in_tiles):
    """
    Generates mesh data for a single voxel cube.
    
    Args:
        voxel_counter: Index of the current voxel in the chunk.
        relative_position: (x, y, z) center of the voxel.
        block_size: Half-extent of the voxel.
        block_type: Enum for the block material.
        atlas_size_in_tiles: Number of tiles per side in the texture atlas.
        
    Returns:
        vertices: List of vertex data (pos, uv, color).
        indices: List of index integers.
        normals: List of normal vectors.
    """
    cx, cy, cz = relative_position
    s = block_size
    
    # 1. Calculate 8 corner points
    # P1: -x, -y, +z (Front-Left-Bottom relative to center? No, based on source: x-s, y-s, z+s)
    # Source mapping:
    # point1: (cx-s, cy-s, cz+s)
    # point2: (cx+s, cy-s, cz+s)
    # point3: (cx+s, cy+s, cz+s)
    # point4: (cx-s, cy+s, cz+s)
    # point5: (cx-s, cy-s, cz-s)
    # point6: (cx+s, cy-s, cz-s)
    # point7: (cx+s, cy+s, cz-s)
    # point8: (cx-s, cy+s, cz-s)
    
    points = [
        (cx - s, cy - s, cz + s), # 1
        (cx + s, cy - s, cz + s), # 2
        (cx + s, cy + s, cz + s), # 3
        (cx - s, cy + s, cz + s), # 4
        (cx - s, cy - s, cz - s), # 5
        (cx + s, cy - s, cz - s), # 6
        (cx + s, cy + s, cz - s), # 7
        (cx - s, cy + s, cz - s)  # 8
    ]
    
    # 2. Define faces by point indices
    # Front (Z+): 1, 2, 3, 4
    # Back (Z-): 5, 6, 7, 8
    # Right (X+): 2, 6, 7, 3
    # Left (X-): 1, 5, 8, 4
    # Top (Y+): 4, 3, 7, 8
    # Bottom (Y-): 1, 5, 6, 2
    
    faces = [
        {'name': 'Front', 'points': [0, 1, 2, 3], 'normal': (0, 0, 1)},
        {'name': 'Back', 'points': [4, 5, 6, 7], 'normal': (0, 0, -1)},
        {'name': 'Right', 'points': [1, 5, 6, 2], 'normal': (1, 0, 0)},
        {'name': 'Left', 'points': [0, 4, 7, 3], 'normal': (-1, 0, 0)},
        {'name': 'Top', 'points': [3, 2, 6, 7], 'normal': (0, 1, 0)},
        {'name': 'Bottom', 'points': [0, 4, 5, 1], 'normal': (0, -1, 0)}
    ]
    
    # 3. Get block color
    colors = {
        'BLOCK_GRASS': (0.0, 1.0, 0.0, 1.0),
        'BLOCK_DIRT': (0.435, 0.305, 0.215, 1.0),
        'BLOCK_STONE': (0.45, 0.45, 0.45, 1.0)
    }
    vertex_color = colors.get(block_type, (1.0, 1.0, 1.0, 1.0))
    
    # 4. Get tile coordinates for each face
    # Note: The specific tile mapping depends on the block type's 'Cube tiles' properties.
    # For this generic function, we assume a function get_face_tile(face_name, block_type) exists.
    # If not given, we cannot generate specific UVs. We will assume a default tile (0,0) for demonstration.
    # In practice, this data comes from the block definition.
    
    vertices = []
    normals = []
    indices = []
    
    stride = 24 * voxel_counter
    
    for face in faces:
        face_points = face['points']
        normal = face['normal']
        
        # Get tile UV for this face
        # Assuming get_face_tile returns (tx, ty)
        tx, ty = get_face_tile(face['name'], block_type) # [G] Helper to retrieve tile indices from block definition
        
        tile_size_uv = 1.0 / atlas_size_in_tiles
        center_u = (tx + 0.5) * tile_size_uv
        center_v = (ty + 0.5) * tile_size_uv
        
        # Define UV offsets for the 4 corners of the quad
        # Assuming standard quad layout: 0:BL, 1:BR, 2:TR, 3:TL
        # The source points order for Front is 1,2,3,4 which corresponds to:
        # 1: (cx-s, cy-s, cz+s) -> Bottom-Left in screen space if Y is up?
        # Let's map point indices to UV corners based on standard quad construction
        # Point 1: x-s, y-s -> u-min, v-min
        # Point 2: x+s, y-s -> u-max, v-min
        # Point 3: x+s, y+s -> u-max, v-max
        # Point 4: x-s, y+s -> u-min, v-max
        
        uv_offsets = [
            (-tile_size_uv/2, -tile_size_uv/2), # 0
            (tile_size_uv/2, -tile_size_uv/2),  # 1
            (tile_size_uv/2, tile_size_uv/2),   # 2
            (-tile_size_uv/2, tile_size_uv/2)   # 3
        ]
        
        face_vertex_indices = []
        for i, pt_idx in enumerate(face_points):
            px, py, pz = points[pt_idx]
            u = center_u + uv_offsets[i][0]
            v = center_v + uv_offsets[i][1]
            
            vertices.append((px, py, pz, u, v, vertex_color))
            normals.append(normal)
            face_vertex_indices.append(stride + len(vertices) - 1)
            
        # Add indices for 2 triangles (quad)
        # Triangle 1: 0, 1, 2
        # Triangle 2: 0, 2, 3
        indices.extend([
            face_vertex_indices[0], face_vertex_indices[1], face_vertex_indices[2],
            face_vertex_indices[0], face_vertex_indices[2], face_vertex_indices[3]
        ])
        
    return vertices, normals, indices

def get_face_tile(face_name, block_type):
    """
    [G] Placeholder for retrieving tile coordinates from block definition.
    In the actual implementation, this reads from the VoxelBlockyModelCube properties.
    """
    # Example: if block_type is GRASS, Top might be (0,0), others might be different
    return (0, 0)
```

### Edge Cases and Pitfalls
*   **Vertex Duplication**: Vertices are duplicated per face to allow independent normals and colors. This increases memory usage by 4x compared to a shared-vertex mesh but is necessary for the blocky aesthetic and per-face lighting.
*   **Texture Atlas Size**: The `atlas_size_in_tiles` must match the actual grid size of the texture atlas. If mismatched, texture coordinates will be incorrect, causing texture stretching or sampling errors.
*   **Material Overrides**: Materials can be overridden at multiple levels (mesh, model, terrain). Ensure the correct material is assigned to the `albedo_texture` property to use the atlas.
*   **Ambient Occlusion**: If `Vertex Color/Use as albedo` is enabled, the vertex colors (which can be baked with AO) will modulate the texture. If disabled, the texture is used as-is.
*   **Instancing**: Do not create instances of `VoxelBlockyModel` directly; use derived classes like `VoxelBlockyModelCube` to avoid legacy conversion issues.

## Operating characteristics

### Time and Memory Complexity
*   **Time**: $O(1)$ per voxel. The mesh generation for a single voxel involves a fixed number of arithmetic operations and data assignments.
*   **Memory**: $O(V)$ where $V$ is the number of voxels. Each voxel consumes memory for 24 vertices, 24 normals, and 24 colors.
    *   Vertex size: 3 (pos) + 2 (uv) + 4 (color) = 9 floats (36 bytes) per vertex.
    *   Normal size: 3 floats (12 bytes) per vertex.
    *   Total per vertex: 48 bytes.
    *   Total per voxel: $24 \times 48 = 1152$ bytes.
    *   Index size: 36 indices (144 bytes if 32-bit).
    *   Total per voxel: ~1.3 KB.

### Measured Performance
*   Not given in the sources.

### Scaling
*   The method scales linearly with the number of voxels.
*   Using texture atlases reduces the number of draw calls by allowing multiple voxel types to share a single material.

### Strengths and Limitations
*   **Strengths**: Simple to implement; supports per-vertex colors for ambient occlusion; supports different textures per face; low memory overhead per voxel compared to high-poly meshes.
*   **Limitations**: High vertex count compared to merged meshes; not suitable for smooth surfaces; requires careful management of texture atlases to avoid texture bleeding.

### When to Use
*   Use for blocky, voxel-based games (e.g., Minecraft-like) where the aesthetic is intentionally angular.
*   Use when per-vertex lighting or ambient occlusion is desired without complex shading techniques.

### When Not to Use
*   Do not use for smooth, organic surfaces.
*   Do not use if memory is extremely constrained and vertex count is a bottleneck.

## Background

### Origin
The method is described in the context of the VoxelTerrain module and Voxel Tools for game engines. It is used to render individual voxel blocks as cubes with specific textures and materials.

### History and Development
The method is part of the VoxelTerrain and Voxel Tools ecosystems, which provide tools for creating blocky terrains. The specific implementation details are found in the source code of the VoxelTerrain module and the documentation for Voxel Tools.

### Where It Is Used
*   VoxelTerrain module [S1, unofficial].
*   Voxel Tools documentation [S2].

## Sources

[S1] Shilghter/VoxelTerrain, https://github.com/Shilghter/VoxelTerrain, fetched 2026-10-27 (unofficial)
[S2] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
