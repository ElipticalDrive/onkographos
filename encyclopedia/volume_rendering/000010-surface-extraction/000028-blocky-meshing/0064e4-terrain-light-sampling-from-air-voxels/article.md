# Terrain light sampling from air voxels

Also called: SG-F7-05, Terrain shows the light around it

## Overview
This method ensures that terrain mesh vertices sample light data from the air voxel adjacent to the surface they face, rather than from the solid block interior or the sky directly, ensuring surfaces display the lighting of the space they bound [S1, unofficial]. It differs from trilinear sampling by using endpoint sampling; since every vertex on a 0/1 density boundary lies on an edge with exactly one air sample, it requires only one lookup per vertex with no blending rules [S1, unofficial]. It builds on [Blocky meshing](31-blocky-meshing.md) by providing the correct lighting data for the geometry it produces.

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid with density values (0 for air, 1 for solid) and light data (block light RGB, sky exposure). The grid uses a padded range of `-1..=16` for both density and light access [S1, unofficial].
*   **Output:** A mesh where each vertex carries a block light RGB value (`[u8; 3]`) and a sky exposure boolean (`bool`) corresponding to the air voxel in front of the surface at that vertex.

**Data Structures:**
*   `light_at`: A function closure with signature `Fn([usize; 3]) -> ([u8; 3], bool)`. It maps a voxel coordinate to block light RGB and sky exposure [S1, unofficial].
*   `light_storage::reading_chunk_offsets`: A data structure or rule set defining which chunks read a specific border voxel, used to determine re-meshing targets when light changes [S1, unofficial].

**Steps:**

1.  **Construct Light Accessor:** Create the `light_at` closure. For hybrid meshing, this closure accesses `grid.block_light_rgb_at` and `grid.sky_exposed_at` using the padded range `-1..=16` [S1, unofficial].

2.  **Sample Light for Smooth Terrain (Marching Cubes/Hybrid):**
    *   During mesh generation, identify vertices lying on an edge between a solid sample (density 1) and an air sample (density 0) [S1, unofficial].
    *   For each such vertex, retrieve the light data from the air sample's position using `light_at` [S1, unofficial].
    *   Assign the returned RGB and sky exposure to the vertex [S1, unofficial].
    *   Note: The smooth mesher reads one voxel past each positive face [S1, unofficial].

3.  **Sample Light for Blocky Terrain:**
    *   For each face, determine the voxel position defined by the face's position plus its normal vector (the voxel in front of the face) [S1, unofficial].
    *   Retrieve the light data from this position using `light_at` [S1, unofficial].
    *   Assign the returned RGB and sky exposure to the face [S1, unofficial].

4.  **Handle Border Re-meshing:**
    *   When a light value is written to a voxel in a chunk's outer layer, identify all chunks that read from that voxel using `light_storage::reading_chunk_offsets` [S1, unofficial].
    *   Mark the chunk across the face for re-meshing (for blocky) [S1, unofficial].
    *   Mark chunks below on every low axis, including diagonals (for smooth, which reads one voxel past each positive face) [S1, unofficial].
    *   This ensures that a light change re-meshes every chunk whose surfaces read the changed voxels [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for light sampling and re-meshing logic
# Assumes grid has methods block_light_rgb_at(pos) and sky_exposed_at(pos)
# Assumes grid uses padded range -1..=16

def get_light_at(pos: tuple[int, int, int]) -> tuple[list[int], bool]:
    """
    Returns block light RGB (list of 3 u8) and sky exposure (bool)
    for the voxel at pos.
    """
    # Implementation details depend on grid storage, but accesses padded range
    rgb = grid.block_light_rgb_at(pos)
    sky = grid.sky_exposed_at(pos)
    return rgb, sky

def mesh_smooth_terrain(density_grid, light_at_func):
    """
    Generates smooth terrain mesh.
    Vertices on 0/1 density edges sample light from the air (0) side.
    """
    mesh_vertices = []
    # Iterate over edges where density changes from 1 to 0
    for edge in density_grid.edges_with_density_change():
        solid_pos = edge.solid_position
        air_pos = edge.air_position
        
        # Calculate vertex position on the edge (e.g., via interpolation)
        vertex_pos = calculate_vertex_position(edge)
        
        # Sample light from the air voxel
        rgb, sky = light_at_func(air_pos)
        
        mesh_vertices.append(Vertex(pos=vertex_pos, light=rgb, sky_exposed=sky))
    
    return mesh_vertices

def mesh_blocky_terrain(block_grid, light_at_func):
    """
    Generates blocky terrain mesh.
    Faces sample light from the voxel in front of them (position + normal).
    """
    mesh_faces = []
    for face in block_grid.exposed_faces:
        # Face position is the block position
        block_pos = face.block_position
        normal = face.normal
        
        # The voxel in front of the face
        front_pos = tuple(p + n for p, n in zip(block_pos, normal))
        
        # Sample light from the front voxel
        rgb, sky = light_at_func(front_pos)
        
        mesh_faces.append(Face(position=block_pos, normal=normal, light=rgb, sky_exposed=sky))
        
    return mesh_faces

def update_light_and_remesh(grid, changed_voxel_pos, affected_chunks, reading_chunk_offsets):
    """
    Updates light in the grid and marks chunks for re-meshing.
    """
    # Update light in grid
    grid.set_light(changed_voxel_pos, new_rgb, new_sky)
    
    # Identify chunks that read from this voxel
    chunks_to_remesh = reading_chunk_offsets.get_chunks_reading(changed_voxel_pos)
    
    for chunk in chunks_to_remesh:
        # Mark chunk for re-meshing
        chunk.dirty = True
        
        # Additional logic for smooth terrain reading past positive faces
        # might require marking diagonal neighbors below if the changed voxel
        # is on a low edge/corner. This is handled by reading_chunk_offsets.
```

**Parameters:**
*   `light_at`: Function closure `Fn([usize; 3]) -> ([u8; 3], bool)` [S1, unofficial].
*   Grid padding range: `-1..=16` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Smooth ground meeting a blocky block:** The blocky block counts as air in the smooth density field. Consequently, the smooth vertex at the foot of a building reads the block's own (dark) light, resulting in contact darkening [S1, unofficial]. This is left as is, with proper contact shading potentially handled by a separate method [SG-F7-06] [S1, unofficial].
*   **Chunk arrival:** Chunks entering the grid do not dirty their neighbors for sky changes because arrival callers already re-mesh the arrived chunks and their face neighbors. However, streamed-in chunks' low-side diagonal neighbors are not re-meshed for sky on arrival [S1, unofficial].
*   **Border re-meshing precision:** A light change does not re-mesh all six neighbors, only those that actually read the changed voxel, ensuring precision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(V) for mesh generation, where V is the number of vertices, as each vertex requires one light lookup [S1, unofficial]. Re-meshing is localized to affected chunks based on light propagation boundaries [S1, unofficial].
*   **Memory Complexity:** O(V) for storing light data per vertex. The `reading_chunk_offsets` structure adds overhead proportional to the number of border voxels and their reading chunks [S1, unofficial].
*   **Strengths:** Ensures surfaces show the lighting of the space they bound. Simple endpoint sampling avoids blending artifacts. Precise re-meshing minimizes unnecessary work [S1, unofficial].
*   **Limitations:** Contact darkening at smooth/blocky interfaces. Incomplete re-meshing for diagonal neighbors on chunk arrival for sky changes [S1, unofficial].
*   **When to use:** When accurate per-vertex lighting based on adjacent air voxels is required for terrain meshes.
*   **When not to use:** If contact shading is critical and not handled separately, or if global illumination effects requiring trilinear sampling are needed (though point lights are out of scope) [S1, unofficial].

## Background
This method was implemented in the `flagstone_voxel` project, specifically in `mesh/marching_cubes.rs`, `mesh/hybrid.rs`, `mesh/blocky.rs`, `light_propagation.rs`, and `light_system.rs` [S1, unofficial]. It addresses the issue where smooth terrain was drawn fully sky-lit with no block light, and blocky faces read the light inside their own opaque block, which is always 0 [S1, unofficial]. The feature was closed as completed in issue #537 on Oct 22, 2026 [S1, unofficial]. It is related to [SG-F7-03] Digging and building relight the world in game #473 for sky refresh logic [S1, unofficial].

## Sources
[S1] [SG-F7-05] Terrain shows the light around it, https://github.com/WrackedFella/moho/issues/490, fetched 2026-10-27 (unofficial)
