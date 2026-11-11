# Voxels Grid to Mesh Conversion (Uncolored/Expanded)

Also called: VoxelsGridToMesh

## Overview
Converts a voxel grid into an expanded mesh representation where each occupied voxel is converted into a full cube with 8 vertices and 12 faces (6 sides, 2 triangles per side) [S1, unofficial]. This method does not share vertices between voxels, resulting in a "blocky" or faceted appearance distinct from smooth surface extraction methods like Marching Cubes. It serves as the foundational geometry generation step for [RealityKit Voxel Mesh Generation](8784-realitykit-voxel-mesh-generation.md), which packs this expanded geometry into a single draw call [S1, unofficial].

## How it works
**Inputs:**
- `grid`: A voxel grid object containing:
  - `Size()`: Total number of voxels in the grid.
  - `VoxelsPerSide()`: Number of voxels along one axis (assuming a cubic grid).
  - `VoxelSize()`: Physical size of a single voxel.
  - `OriginX()`, `OriginY()`, `OriginZ()`: World-space coordinates of the grid's origin.
  - `colors(x, y, z)`: A `Grid<float>` containing Signed Distance Field (SDF) values for each voxel [S1, unofficial].
- `colors`: The SDF grid used for vertex color computation [S1, unofficial].

**Outputs:**
- A mesh object with pre-allocated vertices and faces, containing the expanded cube geometry for all occupied voxels.

**Data Structures:**
- **Vertices**: Pre-allocated array of size `max_vertices_num = grid.Size() * 8`. Each vertex stores position and color.
- **Faces**: Pre-allocated array of size `max_faces_num = grid.Size() * 6`. Note: The source reserves space for `grid.Size() * 6` faces, but inserts 12 triangles (6 sides * 2 triangles) per voxel. This implies the reserve count might refer to quads or is a discrepancy in the source implementation details [S1, unofficial]. Each face is defined by 3 vertex indices.
- **Normals**: A fixed array of 6 predefined normals corresponding to the 6 sides of a cube:
  1. BACK: `(0, 0, 1)`
  2. TOP: `(0, 1, 0)`
  3. RIGHT: `(1, 0, 0)`
  4. FRONT: `(0, 0, -1)`
  5. BOTTOM: `(0, -1, 0)`
  6. LEFT: `(-1, 0, 0)` [S1, unofficial]

**Parameters:**
- `SDFToRGB(value, max)`: A function mapping an SDF value to an RGB color, normalized by `max`.
- `max`: The maximum possible distance for color normalization, calculated as:
  $$ max = \sqrt{3 \cdot (grid.VoxelsPerSide() \cdot grid.VoxelSize())^2} $$
  This represents the diagonal length of the entire voxel grid [S1, unofficial].

**Steps:**
1. **Initialization**:
   - Calculate `max_vertices_num = grid.Size() * 8`.
   - Calculate `max_faces_num = grid.Size() * 6`.
   - Reserve memory for vertices and faces in the mesh object using these maximums [S1, unofficial].
   - Initialize the 6 predefined normals as listed above [S1, unofficial].
   - Calculate `max` using the formula above [S1, unofficial].
   - Initialize `numberVoxelInsert = 0` to track the number of processed voxels [S1, unofficial].

2. **Iteration**:
   - Iterate through all voxel coordinates $(x, y, z)$ in the grid.
   - **Skip Condition**: If the voxel at $(x, y, z)$ is empty or its SDF value is `INFINITY`, skip to the next voxel [S1, unofficial].

3. **Vertex Generation**:
   - For each occupied voxel, generate 8 vertices by iterating $dx, dy, dz \in \{0, 1\}$.
   - Compute the world-space position for each vertex:
     $$
     \begin{aligned}
     pos_x &= grid.OriginX() + (x \cdot grid.VoxelSize()) + (grid.VoxelSize() \cdot dx) \\
     pos_y &= grid.OriginY() + (y \cdot grid.VoxelSize()) + (grid.VoxelSize() \cdot dy) \\
     pos_z &= grid.OriginZ() + (z \cdot grid.VoxelSize()) + (grid.VoxelSize() \cdot dz)
     \end{aligned}
     $$
   - Compute the vertex color using the SDF value at the voxel's center:
     $$ color = SDFToRGB(\sqrt{colors(x, y, z)}, max) $$
     Note: The source uses `std::sqrt(colors(x, y, z))` for the color input [S1, unofficial].
   - Add the vertex (position, color) to the mesh [S1, unofficial].

4. **Face Generation**:
   - Add 12 triangles (6 sides, 2 triangles per side) for the voxel.
   - The vertex indices for the triangles are based on `numberVoxelInsert * 8`. The 8 vertices of the current voxel are indexed from `numberVoxelInsert * 8` to `numberVoxelInsert * 8 + 7`.
   - The normals array index determines the normal for each side:
     - Side 0 (BACK): Normal index 0 `(0,0,1)`
     - Side 1 (TOP): Normal index 1 `(0,1,0)`
     - Side 2 (RIGHT): Normal index 2 `(1,0,0)`
     - Side 3 (FRONT): Normal index 3 `(0,0,-1)`
     - Side 4 (BOTTOM): Normal index 4 `(0,-1,0)`
     - Side 5 (LEFT): Normal index 5 `(-1,0,0)` [S1, unofficial]
   - Insert the 3 vertex indices for each of the 2 triangles per side into the mesh's face list [S1, unofficial].

5. **Finalization**:
   - Increment `numberVoxelInsert` after processing each voxel [S1, unofficial].
   - Shrink the mesh containers to fit the actual number of vertices and faces generated [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxels_grid_to_mesh_expanded(grid, colors):
    # 1. Initialization
    max_vertices_num = grid.size() * 8
    max_faces_num = grid.size() * 6  # Note: Source reserves this, but inserts 12 triangles per voxel
    
    mesh.vertices.reserve(max_vertices_num)
    mesh.faces.reserve(max_faces_num)
    
    # Predefined normals: BACK, TOP, RIGHT, FRONT, BOTTOM, LEFT
    normals = [
        (0, 0, 1),   # 0: BACK
        (0, 1, 0),   # 1: TOP
        (1, 0, 0),   # 2: RIGHT
        (0, 0, -1),  # 3: FRONT
        (0, -1, 0),  # 4: BOTTOM
        (-1, 0, 0)   # 5: LEFT
    ]
    
    # Calculate max distance for color normalization
    voxels_per_side = grid.voxels_per_side()
    voxel_size = grid.voxel_size()
    max_dist = sqrt(3 * (voxels_per_side * voxel_size) ** 2)
    
    number_voxel_insert = 0
    
    # 2. Iterate through all voxels
    for x in range(grid.dim_x):
        for y in range(grid.dim_y):
            for z in range(grid.dim_z):
                # Skip empty or infinite SDF voxels
                if not grid.is_occupied(x, y, z) or grid.sdf(x, y, z) == INFINITY:
                    continue
                
                # 3. Generate 8 vertices
                base_index = number_voxel_insert * 8
                for dx in [0, 1]:
                    for dy in [0, 1]:
                        for dz in [0, 1]:
                            pos_x = grid.origin_x() + (x * voxel_size) + (voxel_size * dx)
                            pos_y = grid.origin_y() + (y * voxel_size) + (voxel_size * dy)
                            pos_z = grid.origin_z() + (z * voxel_size) + (voxel_size * dz)
                            
                            # Compute color from SDF
                            sdf_val = grid.sdf(x, y, z)
                            color = sdf_to_rgb(sqrt(sdf_val), max_dist)
                            
                            mesh.vertices.append((pos_x, pos_y, pos_z, color))
                
                # 4. Add 12 triangles (6 sides, 2 triangles per side)
                # Define vertex indices for the 8 corners of the cube
                # Order: dx, dy, dz from 000 to 111
                # Index = dx + 2*dy + 4*dz
                corner_indices = []
                for dx in [0, 1]:
                    for dy in [0, 1]:
                        for dz in [0, 1]:
                            idx = dx + 2 * dy + 4 * dz
                            corner_indices.append(base_index + idx)
                
                # Define triangles for each side
                # Side 0: BACK (normal 0,0,1) - typically dz=1 plane
                # Side 1: TOP (normal 0,1,0) - typically dy=1 plane
                # Side 2: RIGHT (normal 1,0,0) - typically dx=1 plane
                # Side 3: FRONT (normal 0,0,-1) - typically dz=0 plane
                # Side 4: BOTTOM (normal 0,-1,0) - typically dy=0 plane
                # Side 5: LEFT (normal -1,0,0) - typically dx=0 plane
                
                # Note: The exact triangle winding and vertex mapping for each side
                # is not explicitly detailed in the source beyond the normal index.
                # Standard cube triangulation is assumed here based on normal indices.
                
                # Example for BACK (dz=1): vertices with dz=1 are indices 4,5,6,7
                # Triangles: (4,5,7), (4,7,6) or similar depending on winding
                # The source states "12 triangles (6 sides, 2 triangles per side)"
                # and uses normals array index to select the normal.
                
                # Since the exact vertex ordering for the 12 triangles is not provided,
                # we assume a standard cube triangulation pattern consistent with the normals.
                
                # Side 0: BACK (dz=1)
                mesh.faces.append((corner_indices[4], corner_indices[5], corner_indices[7]))
                mesh.faces.append((corner_indices[4], corner_indices[7], corner_indices[6]))
                
                # Side 1: TOP (dy=1)
                mesh.faces.append((corner_indices[2], corner_indices[3], corner_indices[7]))
                mesh.faces.append((corner_indices[2], corner_indices[7], corner_indices[6]))
                
                # Side 2: RIGHT (dx=1)
                mesh.faces.append((corner_indices[1], corner_indices[3], corner_indices[7]))
                mesh.faces.append((corner_indices[1], corner_indices[7], corner_indices[5]))
                
                # Side 3: FRONT (dz=0)
                mesh.faces.append((corner_indices[0], corner_indices[6], corner_indices[2]))
                mesh.faces.append((corner_indices[0], corner_indices[4], corner_indices[6]))
                
                # Side 4: BOTTOM (dy=0)
                mesh.faces.append((corner_indices[0], corner_indices[1], corner_indices[5]))
                mesh.faces.append((corner_indices[0], corner_indices[5], corner_indices[4]))
                
                # Side 5: LEFT (dx=0)
                mesh.faces.append((corner_indices[0], corner_indices[2], corner_indices[6]))
                mesh.faces.append((corner_indices[0], corner_indices[6], corner_indices[4]))
                
                # 5. Increment counter
                number_voxel_insert += 1
    
    # Shrink containers to fit
    mesh.vertices.shrink_to_fit()
    mesh.faces.shrink_to_fit()
    
    return mesh
```

**Edge Cases and Pitfalls:**
- **Empty Voxels**: Voxels with `INFINITY` SDF or marked as empty must be skipped to avoid generating degenerate geometry [S1, unofficial].
- **Memory Reservation**: The source reserves `grid.Size() * 6` faces but inserts 12 triangles per voxel. This discrepancy should be noted; the implementation may need to adjust the reservation or the insertion logic to avoid out-of-bounds errors if the reservation is strictly enforced [S1, unofficial].
- **Color Normalization**: The `max` value is based on the grid's diagonal. If the grid is not cubic or the SDF values are not normalized, the color mapping may be incorrect [S1, unofficial].

## Subtopics

### [RealityKit Voxel Mesh Generation](8784-realitykit-voxel-mesh-generation.md)
Generates a single `MeshResource` from an array of voxel positions and colors to render ~1M voxels in real time, avoiding the per-entity overhead of instantiating individual `ModelEntity` objects. It builds on [Voxels Grid to Mesh Conversion (Uncolored/Expanded)](8733-voxels-grid-to-mesh-conversion-uncolored-expanded.md) by packing the expanded cube geometry into a single draw call via `MeshDescriptor`.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the grid. Each voxel is processed once, and a constant number of vertices and faces are generated per voxel.
- **Memory Complexity**: $O(N)$ for vertices and faces. Specifically, $8N$ vertices and $12N$ triangles (or $6N$ faces as per the source's reservation) are generated.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Linear scaling with the number of voxels. The method is suitable for dense voxel grids but may result in large mesh sizes due to the lack of vertex sharing.
- **Strengths**: Simple implementation, deterministic output, easy to parallelize.
- **Limitations**: High vertex count due to no vertex sharing, resulting in larger memory footprint and potentially slower rendering compared to optimized meshing techniques. The "blocky" appearance may not be suitable for all applications.
- **When to use**: When a simple, fast conversion from voxel grid to mesh is required, and the visual fidelity of a smooth surface is not critical. Useful for debugging, visualization, or as a base for further processing.
- **When not to use**: When smooth surfaces are required, or when memory efficiency is a primary concern.

## Background
This method is part of the [cuda-mesh-voxelization](https://github.com/bigmat18/cuda-mesh-voxelization) library, which supports mesh voxelization, CSG operations, and SDF calculation [S1, unofficial]. The library uses the Jump Flooding Algorithm (JFA) for SDF computation [S1, unofficial]. The specific method for converting the voxel grid to an expanded mesh is described in the library's documentation [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
