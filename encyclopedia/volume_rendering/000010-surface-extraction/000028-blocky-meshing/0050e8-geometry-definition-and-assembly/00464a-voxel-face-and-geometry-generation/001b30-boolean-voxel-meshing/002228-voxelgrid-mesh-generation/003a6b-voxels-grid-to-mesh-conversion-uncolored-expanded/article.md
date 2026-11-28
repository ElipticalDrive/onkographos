# Voxels Grid to Mesh Conversion (Uncolored/Expanded)

Also called: FVox::CreateRawMeshes, Raw Mesh Generation (Per-Voxel), VoxelsGridToMesh

## Overview
Converts a voxel grid into an expanded mesh representation where each occupied voxel is converted into a full cube with 8 vertices and 12 faces (6 sides, 2 triangles per side) [S1, S2, unofficial]. This method does not share vertices between voxels [S1, S2, unofficial].

## How it works
**Inputs:**
- `grid`: A voxel grid object containing `Size()`, `VoxelsPerSide()`, `VoxelSize()`, `OriginX()`, `OriginY()`, `OriginZ()`, `dim_x`, `dim_y`, `dim_z`, `is_occupied(x, y, z)`, `sdf(x, y, z)`, and `import_xy_center()` [S1, S2, unofficial].
- `colors`: A `Grid<float>` containing SDF values for each voxel, used to compute vertex colors [S2, unofficial].

**Outputs:**
- A mesh containing expanded cube geometry for all occupied voxels [S1, S2, unofficial].

**Data Structures:**
- **Vertices**: Pre-allocated array of size `max_vertices_num = grid.Size() * 8` [S1, S2, unofficial]. Each vertex stores a 3D position and a color [S1, S2, unofficial].
- **Faces**: Pre-allocated array reserved for `max_faces_num = grid.Size() * 6` [S1, S2, unofficial]. Each face is defined by 3 vertex indices [S1, S2, unofficial].
- **Normals**: A fixed array of 6 predefined normals: `(0,0,1)`, `(0,1,0)`, `(1,0,0)`, `(0,0,-1)`, `(0,-1,0)`, `(-1,0,0)` [S1, S2, unofficial].
- **Vertex Layout**: `Vertexes[8]` ordered as `(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)` [S1, unofficial].
- **Face Layout**: `Faces[6][4]` defines quads: `Up={5,4,7,6}`, `Down={0,1,2,3}`, `Forward={5,6,2,1}`, `Backward={7,4,0,3}`, `Right={6,7,3,2}`, `Left={4,5,1,0}` [S1, unofficial].
- **Polygon Split**: `Polygons[2][3]` splits each quad into two triangles: `{0,1,2}` and `{2,3,0}` [S1, unofficial].

**Parameters:**
- `SDFToRGB(value, max)`: Maps an SDF value to an RGB color normalized by `max` [S2, unofficial].
- `max`: Maximum distance for color normalization, calculated as `max = sqrt(3 * (grid.VoxelsPerSide() * grid.VoxelSize())^2)` [S2, unofficial].

**Steps:**
1. **Initialization**: Calculate `max_vertices_num = grid.Size() * 8` and `max_faces_num = grid.Size() * 6` [S1, S2, unofficial]. Reserve memory for vertices and faces [S1, S2, unofficial]. Initialize the 6 predefined normals [S1, S2, unofficial]. Calculate `max` [S2, unofficial]. Initialize `numberVoxelInsert = 0` [S2, unofficial].
2. **Iteration**: Iterate through all voxel coordinates $(x, y, z)$ in the grid [S2, unofficial]. If the voxel is empty or its SDF value is `INFINITY`, skip it [S2, unofficial].
3. **Vertex Generation**: For each occupied voxel, generate 8 vertices by iterating $dx, dy, dz \in \{0, 1\}$ [S2, unofficial]. Compute the world-space position as `grid.OriginX() + (x * grid.VoxelSize()) + (grid.VoxelSize() * dx)` (and similarly for Y and Z) [S2, unofficial]. Compute the vertex color using `SDFToRGB(sqrt(colors(x, y, z)), max)` [S2, unofficial]. Add the vertex to the mesh [S1, S2, unofficial].
4. **Face Generation**: Add 12 triangles (6 sides, 2 triangles per side) for the voxel [S1, S2, unofficial]. The vertex indices are based on `numberVoxelInsert * 8` [S2, unofficial]. The normals array index selects the normal for each side: 0 for BACK, 3 for FRONT, 1 for TOP, 4 for BOTTOM, 2 for RIGHT, 5 for LEFT [S2, unofficial]. Insert the 3 vertex indices for each triangle into the mesh's face list [S1, unofficial].
5. **Finalization**: Increment `numberVoxelInsert` after processing each voxel [S2, unofficial]. Shrink the mesh containers to fit [S2, unofficial]. If `ImportOption->bImportXYCenter` is true, subtract `(Size.X*0.5, Size.Y*0.5, 0)` from all `VertexPositions` [S1, unofficial]. Call `CompactMaterialIndices()` and verify validity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def voxels_grid_to_mesh_expanded(grid, colors):
    # Evidence-defined layouts
    VERTEXES = [
        (0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0),
        (0, 0, 1), (1, 0, 1), (1, 1, 1), (0, 1, 1)
    ]
    FACES = [
        [5, 4, 7, 6], # Up
        [0, 1, 2, 3], # Down
        [5, 6, 2, 1], # Forward
        [7, 4, 0, 3], # Backward
        [6, 7, 3, 2], # Right
        [4, 5, 1, 0]  # Left
    ]
    POLYGONS = [[0, 1, 2], [2, 3, 0]]
    NORMALS = [
        (0, 0, 1), (0, 1, 0), (1, 0, 0),
        (0, 0, -1), (0, -1, 0), (-1, 0, 0)
    ]

    # Pre-allocate containers
    max_vertices = grid.size() * 8
    max_faces = grid.size() * 6
    vertices = [None] * max_vertices
    faces = []
    normals_out = []
    colors_out = []

    # Color normalization constant
    voxels_per_side = grid.voxels_per_side()
    voxel_size = grid.voxel_size()
    max_dist = math.sqrt(3 * (voxels_per_side * voxel_size) ** 2)

    number_voxel_insert = 0

    for x in range(grid.dim_x):
        for y in range(grid.dim_y):
            for z in range(grid.dim_z):
                # Skip empty or infinite SDF voxels
                if not grid.is_occupied(x, y, z) or grid.sdf(x, y, z) == float('inf'):
                    continue

                base_idx = number_voxel_insert * 8

                # Generate 8 vertices
                for i, (dx, dy, dz) in enumerate(VERTEXES):
                    px = grid.origin_x() + (x * voxel_size) + (voxel_size * dx)
                    py = grid.origin_y() + (y * voxel_size) + (voxel_size * dy)
                    pz = grid.origin_z() + (z * voxel_size) + (voxel_size * dz)

                    # Compute color from SDF value
                    sdf_val = grid.sdf(x, y, z)
                    color_val = math.sqrt(sdf_val) if sdf_val >= 0 else 0.0
                    color = sdf_to_rgb(color_val, max_dist)

                    vertices[base_idx + i] = (px, py, pz, color)

                # Generate 12 triangles (6 sides, 2 per side)
                for face_idx in range(6):
                    quad = FACES[face_idx]
                    normal = NORMALS[face_idx]
                    normals_out.append(normal)
                    normals_out.append(normal)

                    for poly_idx in range(2):
                        tri_indices = [quad[POLYGONS[poly_idx][j]] for j in range(3)]
                        face_verts = [base_idx + idx for idx in tri_indices]
                        faces.append(face_verts)
                        # Add color 3 times per triangle
                        c = vertices[base_idx + tri_indices[0]][3]
                        colors_out.append(c)
                        colors_out.append(c)
                        colors_out.append(c)

                number_voxel_insert += 1

    # Apply XY center offset if required
    if grid.import_xy_center():
        offset_x = grid.dim_x * 0.5
        offset_y = grid.dim_y * 0.5
        for i in range(len(vertices)):
            if vertices[i] is not None:
                px, py, pz, c = vertices[i]
                vertices[i] = (px - offset_x, py - offset_y, pz, c)

    # Shrink containers to fit actual data
    vertices = [v for v in vertices if v is not None]
    return Mesh(vertices, faces, normals_out, colors_out)

def sdf_to_rgb(val, max_val):
    # Standard linear normalization [G]
    t = val / max_val if max_val > 0 else 0.0
    return (t, t, t)

class Mesh:
    def __init__(self, vertices, faces, normals, colors):
        self.vertices = vertices
        self.faces = faces
        self.normals = normals
        self.colors = colors
```

**Edge Cases and Pitfalls:**
- **Empty Voxels**: Voxels with `INFINITY` SDF or marked as empty must be skipped to avoid generating degenerate geometry [S2, unofficial].
- **Memory Reservation**: The source reserves `grid.Size() * 6` faces but inserts 12 triangles per voxel [S1, S2, unofficial].
- **Color Normalization**: The `max` value is based on the grid's diagonal [S2, unofficial].
- **Precision**: Vertex positions are computed using floating-point arithmetic; ensure grid origin and voxel size are compatible with the target coordinate space [G].

## Subtopics

### [RealityKit Voxel Mesh Generation](8784-realitykit-voxel-mesh-generation.md)
Generates a single `MeshResource` from an array of voxel positions and colors to render ~1M voxels in real time, avoiding the per-entity overhead of instantiating individual `ModelEntity` objects. It builds on [Voxels Grid to Mesh Conversion (Uncolored/Expanded)](8733-voxels-grid-to-mesh-conversion-uncolored-expanded.md) by packing the expanded cube geometry into a single draw call via `MeshDescriptor`.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the grid [S1, S2, unofficial].
- **Memory Complexity**: $O(N)$ for vertices and faces. Specifically, $8N$ vertices and $12N$ triangles are generated [S1, S2, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Linear scaling with the number of voxels [S1, S2, unofficial].
- **Strengths**: Simple implementation, deterministic output, easy to parallelize [S1, S2, unofficial].
- **Limitations**: High vertex count due to no vertex sharing, resulting in larger memory footprint [S1, S2, unofficial].
- **When to use**: When a simple, fast conversion from voxel grid to mesh is required [S1, S2, unofficial].
- **When not to use**: When smooth surfaces are required, or when memory efficiency is a primary concern [S1, S2, unofficial].

## Background
This method is implemented in the `VOX4U` library as `FVox::CreateRawMeshes` and in the `cuda-mesh-voxelization` library for SDF-based voxel processing [S1, S2, unofficial]. The library supports mesh voxelization, CSG operations, and SDF calculation using the Jump Floading Algorithm (JFA) [S2, unofficial].

## Sources
[S1] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
[S2] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
