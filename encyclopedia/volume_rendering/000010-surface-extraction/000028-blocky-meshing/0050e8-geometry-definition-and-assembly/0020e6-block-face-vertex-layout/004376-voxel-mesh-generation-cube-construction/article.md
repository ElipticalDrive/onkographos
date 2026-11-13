# Voxel Mesh Generation (Cube Construction)

Also called: CreateQuadSmoothVoxel, CreateQuadVoxel, Cube model, FVox::CreateMesh, Single Cube Mesh Generation, VoxelBlockyModelCube

## Overview
Voxel Mesh Generation (Cube Construction) constructs a static, axis-aligned cube mesh for a single voxel block by explicitly defining its geometry, normals, and per-vertex attributes. It solves the problem of converting discrete voxel data into a renderable triangular mesh suitable for immediate mode or indexed rendering. Unlike smooth surface extraction methods (e.g., Marching Cubes) that produce curved surfaces, this method preserves the blocky, discrete nature of the voxel grid. It differs from [Block face vertex layout](2374-block-face-vertex-layout.md) by handling the complete assembly of a single voxel's 24 vertices and indices, whereas the parent article defines the abstract vertex structure for a unit cube face.

## How it works
**Inputs:**
- `relativePosition`: The 3D coordinates of the voxel block's origin or center.
- `BLOCK_SIZE`: The half-extent of the voxel cube (distance from center to face).
- `blockType`: An enum or integer identifying the voxel material (e.g., grass, dirt, stone).
- `textureAtlas`: A square texture containing tile definitions.
- `atlas_size_in_tiles`: The number of tiles along one axis of the atlas (typically 16).

**Outputs:**
- A mesh buffer containing 24 vertices per voxel.
- Vertex positions, normals, texture coordinates, and colors.
- Index buffer defining the triangles.

**Data Structures:**
The mesh stores 24 vertices per voxel, organized as 4 vertices per face $\times$ 6 faces.
- **Positions:** 24 vectors.
- **Normals:** 24 vectors (duplicated per vertex, 4 per face).
- **Colors:** 24 vectors (per-vertex, typically RGBA).
- **Texture Coordinates:** 24 vectors (UVs mapped to the atlas).
- **Indices:** 36 indices (6 triangles $\times$ 2 triangles per face $\times$ 6 faces).

**Steps:**

1.  **Calculate Corner Points:**
    Compute the 8 corner points of the cube relative to `relativePosition` using `BLOCK_SIZE`. Let $P$ be `relativePosition`. The corners are defined by adding/subtracting `BLOCK_SIZE` from each axis:
    $$
    \begin{aligned}
    point1 &= (P_x - S, P_y - S, P_z + S) \\
    point2 &= (P_x + S, P_y - S, P_z + S) \\
    point3 &= (P_x + S, P_y + S, P_z + S) \\
    point4 &= (P_x - S, P_y + S, P_z + S) \\
    point5 &= (P_x - S, P_y - S, P_z - S) \\
    point6 &= (P_x + S, P_y - S, P_z - S) \\
    point7 &= (P_x + S, P_y + S, P_z - S) \\
    point8 &= (P_x - S, P_y + S, P_z - S)
    \end{aligned}
    $$
    where $S = \text{BLOCK\_SIZE}$ [S2, unofficial].

2.  **Assign Vertex Colors:**
    Set the per-vertex color for all 24 vertices based on `blockType` [S2, unofficial]:
    - `BLOCK_GRASS`: $(0.0, 1.0, 0.0, 1.0)$
    - `BLOCK_DIRT`: $(0.435, 0.305, 0.215, 1.0)$
    - `BLOCK_STONE`: $(0.45, 0.45, 0.45, 1.0)$
    [G] Other block types should map to their respective material colors or default to white $(1.0, 1.0, 1.0, 1.0)$.

3.  **Generate Face Geometry:**
    Iterate through the 6 faces (Front, Back, Right, Left, Top, Bottom). For each face:
    a.  **Vertices:** Add the 4 corner vertices associated with that face to the mesh.
    b.  **Normals:** Add the axis-aligned normal vector for that face to the 4 vertices.
        - Front: $(0, 0, 1)$
        - Back: $(0, 0, -1)$
        - Right: $(1, 0, 0)$
        - Left: $(-1, 0, 0)$
        - Top: $(0, 1, 0)$
        - Bottom: $(0, -1, 0)$
    c.  **Texture Coordinates:** Calculate UV coordinates for the 4 vertices. The coordinates are in "tiles", not pixels. If the atlas has $N$ tiles per side (e.g., 16), and the tile for the face is at grid position $(u, v)$ (0-indexed), the UVs for the face corners are:
        $$
        \begin{aligned}
        UV_{min} &= \left( \frac{u}{N}, \frac{v}{N} \right) \\
        UV_{max} &= \left( \frac{u+1}{N}, \frac{v+1}{N} \right)
        \end{aligned}
        $$
        Assign these to the 4 vertices of the face. [S3]
    d.  **Indices:** Add 6 indices to form two triangles (a quad) for the face. The winding order must be consistent (e.g., counter-clockwise) to ensure correct culling. [S1, unofficial]

4.  **Material Assignment:**
    Assign the material using the `albedo_texture` from the texture atlas. If "Vertex Color/Use as albedo" is enabled, the mesher may bake ambient occlusion into the vertex colors [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(relative_position, block_size, block_type, atlas_tiles_per_side, tile_uv_map):
    """
    Generates a mesh for a single voxel cube.
    
    Args:
        relative_position: (x, y, z) float tuple.
        block_size: float, half-extent of the cube.
        block_type: int or enum, identifies the voxel material.
        atlas_tiles_per_side: int, e.g., 16.
        tile_uv_map: dict mapping block_type to (u, v) tile indices.
        
    Returns:
        vertices: list of (pos, normal, uv, color) tuples.
        indices: list of int indices.
    """
    vertices = []
    indices = []
    
    # 1. Calculate 8 corners
    # point1 to point8 as defined in [S2, unofficial]
    # Note: The specific mapping of point1-8 to faces depends on the coordinate system.
    # Assuming standard right-handed or left-handed system consistent with [S2, unofficial].
    # Let's define corners explicitly based on [S2, unofficial] code snippet:
    # point1: (-S, -S, +S) relative to center
    # point2: (+S, -S, +S)
    # point3: (+S, +S, +S)
    # point4: (-S, +S, +S)
    # point5: (-S, -S, -S)
    # point6: (+S, -S, -S)
    # point7: (+S, +S, -S)
    # point8: (-S, +S, -S)
    
    corners = [
        (relative_position[0] - block_size, relative_position[1] - block_size, relative_position[2] + block_size), # 1
        (relative_position[0] + block_size, relative_position[1] - block_size, relative_position[2] + block_size), # 2
        (relative_position[0] + block_size, relative_position[1] + block_size, relative_position[2] + block_size), # 3
        (relative_position[0] - block_size, relative_position[1] + block_size, relative_position[2] + block_size), # 4
        (relative_position[0] - block_size, relative_position[1] - block_size, relative_position[2] - block_size), # 5
        (relative_position[0] + block_size, relative_position[1] - block_size, relative_position[2] - block_size), # 6
        (relative_position[0] + block_size, relative_position[1] + block_size, relative_position[2] - block_size), # 7
        (relative_position[0] - block_size, relative_position[1] + block_size, relative_position[2] - block_size), # 8
    ]
    
    # 2. Define faces by corner indices and normals
    # Face structure: (list of 4 corner indices, normal vector)
    # Mapping based on [S2, unofficial] and standard cube topology:
    # Front (Z+): 1, 2, 3, 4 -> Normal (0,0,1)
    # Back (Z-):  5, 6, 7, 8 -> Normal (0,0,-1)
    # Right (X+): 2, 3, 7, 6 -> Normal (1,0,0)
    # Left (X-):  1, 4, 8, 5 -> Normal (-1,0,0)
    # Top (Y+):   4, 3, 7, 8 -> Normal (0,1,0)
    # Bottom (Y-):1, 2, 6, 5 -> Normal (0,-1,0)
    
    faces = [
        ([0, 1, 2, 3], (0.0, 0.0, 1.0)),   # Front
        ([4, 5, 6, 7], (0.0, 0.0, -1.0)),  # Back
        ([1, 2, 6, 5], (1.0, 0.0, 0.0)),   # Right
        ([0, 3, 7, 4], (-1.0, 0.0, 0.0)),  # Left
        ([3, 2, 6, 7], (0.0, 1.0, 0.0)),   # Top
        ([0, 1, 5, 4], (0.0, -1.0, 0.0)),  # Bottom
    ]
    
    # 3. Get color for block type
    colors = {
        'GRASS': (0.0, 1.0, 0.0, 1.0),
        'DIRT': (0.435, 0.305, 0.215, 1.0),
        'STONE': (0.45, 0.45, 0.45, 1.0)
    }
    color = colors.get(block_type, (1.0, 1.0, 1.0, 1.0))
    
    # 4. Get tile UV for block type
    tile_uv = tile_uv_map.get(block_type, (0, 0))
    u, v = tile_uv
    tile_size = 1.0 / atlas_tiles_per_side
    u_min, v_min = u * tile_size, v * tile_size
    u_max, v_max = (u + 1) * tile_size, (v + 1) * tile_size
    
    vertex_offset = 0
    
    for face_corners, normal in faces:
        # Calculate UVs for the 4 corners of this face
        # The mapping of UVs to corners depends on the face orientation.
        # For simplicity, we assume a standard mapping where the face is a quad.
        # We need to determine which corner corresponds to (u_min, v_min) etc.
        # This is implementation-specific. A common approach is to map local face UVs.
        # Let's assume a standard local UV space for the face:
        # Corner 0: (0,0), Corner 1: (1,0), Corner 2: (1,1), Corner 3: (0,1)
        # Then map to atlas.
        
        face_uvs = [
            (u_min, v_min),
            (u_max, v_min),
            (u_max, v_max),
            (u_min, v_max)
        ]
        
        for i, corner_idx in enumerate(face_corners):
            pos = corners[corner_idx]
            vert = (pos, normal, face_uvs[i], color)
            vertices.append(vert)
            
        # Add indices for two triangles (quad)
        # Triangle 1: 0, 1, 2
        # Triangle 2: 0, 2, 3
        indices.extend([vertex_offset, vertex_offset + 1, vertex_offset + 2])
        indices.extend([vertex_offset, vertex_offset + 2, vertex_offset + 3])
        
        vertex_offset += 4
        
    return vertices, indices
```

## Subtopics

### isocubesGrob
Creates a grob of isometric cubes representing voxels at given coordinates [S4, unofficial]. It differs from standard axis-aligned cube construction by projecting voxels into an isometric view with specific shading intensities for top, left, and right faces. It is chosen for 2D visualization of 3D voxel data in R graphics. Parameters include `coords` (x,y,z), `fill` colors, `intensity` shading, `size`, and `xyplane` orientation [S4, unofficial].

### Legolizer Polygonization
Converts a voxel representation into a LEGOr brick mesh, generating uniform cubes with cylinders on top and disks covering them [S5]. It differs by adding cylindrical studs to the top face of each cube to mimic LEGO bricks. It is chosen when the goal is to render a voxel model as a physical LEGO set. The procedure emits a triangle strip with 6 cube faces, 1 cylinder, and 1 disk per voxel [S5].

## Operating characteristics
- **Time Complexity:** $O(V)$ where $V$ is the number of voxels. Each voxel generates a constant number of vertices (24) and indices (36).
- **Memory Complexity:** $O(V)$. Each voxel requires storage for 24 vertices and 36 indices.
- **Performance:** Rendering is efficient due to the simplicity of the geometry and the ability to use instancing or batching. Using texture atlases reduces draw calls significantly [S3].
- **Strengths:** Simple to implement, fast to generate, preserves the discrete voxel aesthetic.
- **Limitations:** High vertex count compared to merged mesh techniques (e.g., marching cubes or blocky meshing with face culling). No smooth surfaces.
- **When to use:** When the blocky aesthetic is desired, or as a base for further optimization (like face culling).
- **When not to use:** When smooth surfaces are required, or when memory is extremely constrained and vertex count is a bottleneck.

## Background
The method originates from early voxel-based games and engines where simple cube meshes were used to represent terrain and objects. The specific implementation details, such as the 24-vertex layout and per-vertex coloring, are found in various open-source voxel engines and tools [S1, S2, unofficial][S3]. The "Legolizer" variant was developed for real-time LEGO model rendering [S5]. The `isocubesGrob` function is part of the R `isocubes` package for statistical visualization [S4, unofficial].

## Sources
[S1] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
[S2] Shilghter/VoxelTerrain, https://github.com/Shilghter/VoxelTerrain, fetched 2026-10-27 (unofficial)
[S3] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
[S4] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
[S5] Legolizer: A Real-Time System for Modeling and Rendering LEGO Representations of Boundary Models, https://doi.org/10.1109/sibgrapi.2009.18, fetched 2026-11-02
