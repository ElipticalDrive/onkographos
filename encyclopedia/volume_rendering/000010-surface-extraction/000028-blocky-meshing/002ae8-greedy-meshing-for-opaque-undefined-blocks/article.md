# Greedy Meshing for Opaque/Undefined Blocks


## Overview
This method extends standard greedy meshing to handle voxel types that are either fully opaque or explicitly undefined (empty), ensuring that only visible faces of non-empty blocks are meshed while enforcing strict appearance definitions. It differs from general greedy meshing by introducing explicit assertions for missing block appearances and standardized UV/color padding for undefined blocks, preventing rendering artifacts in engines that require complete vertex attributes.

## How it works
The method processes voxel chunks or grids, identifying visible faces of opaque blocks while rejecting or handling undefined blocks based on appearance definitions.

**Inputs and Outputs:**
- Input: A voxel grid (chunk or world) containing block types (Opaque/Undefined).
- Output: A mesh containing vertices, indices, UVs, and colors.

**Data Structures:**
- **Mesh**: Contains arrays for vertices, indices, UVs, and colors.
- **Block Type**: Enum or ID representing `Opaque` or `Undefined`.

**Steps:**
1. Iterate through all voxels in the input grid.
2. For each voxel, determine if it is opaque or undefined.
3. If the block is opaque:
   - Check for visible faces (adjacent voxels are empty or different type).
   - If visible, generate a quad for the face.
   - Assign UVs and colors to the quad.
4. If the block is undefined:
   - Assert that an appearance is defined for this block face. If not, fail with "No appearance defined for block face" [S1, unofficial].
   - If an appearance is defined, proceed to generate the quad.
5. For every generated quad, extend the mesh's UVs array with 4 sets of `[0.0; 2]` UV coordinates [S1, unofficial].
6. For every generated quad, extend the mesh's colors array with 4 sets of `[0.0; 4]` color coordinates [S1, unofficial].
7. Combine all quads into the final mesh.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_opaque_undefined(voxel_grid):
    mesh_vertices = []
    mesh_indices = []
    mesh_uvs = []
    mesh_colors = []
    
    # Iterate through all voxels
    for x, y, z in voxel_grid:
        block = voxel_grid.get(x, y, z)
        
        # Determine visible faces
        visible_faces = get_visible_faces(block, voxel_grid, x, y, z)
        
        for face in visible_faces:
            # Check for appearance definition
            if not has_appearance(block, face):
                raise AssertionError("No appearance defined for block face")
            
            # Generate quad geometry
            quad_vertices = generate_quad_vertices(block, face, x, y, z)
            quad_indices = [0, 1, 2, 2, 3, 0] # Two triangles
            
            # Add vertices and indices
            base_vertex = len(mesh_vertices)
            mesh_vertices.extend(quad_vertices)
            mesh_indices.extend([i + base_vertex for i in quad_indices])
            
            # Extend UVs with 4 sets of [0.0, 2.0]
            for _ in range(4):
                mesh_uvs.extend([0.0, 2.0])
            
            # Extend Colors with 4 sets of [0.0, 0.0, 0.0, 1.0] (assuming default)
            for _ in range(4):
                mesh_colors.extend([0.0, 0.0, 0.0, 1.0])
                
    return Mesh(mesh_vertices, mesh_indices, mesh_uvs, mesh_colors)
```

**Parameters:**
- `voxel_grid`: The input voxel data structure.
- `has_appearance(block, face)`: Function to check if a block type has a defined appearance for a specific face.

**Edge Cases and Pitfalls:**
- **Missing Appearance**: If a block face lacks an appearance definition, the method asserts failure. This must be handled by ensuring all block types have valid appearances [S1, unofficial].
- **UV/Color Padding**: The method explicitly pads UVs and colors for undefined blocks. Ensure the renderer handles these default values correctly [S1, unofficial].

## Subtopics

### Chunk Neighbor Optimization
An optimization where chunks are made aware of their neighboring chunks to remove inner polygons that are never visible, solving the issue where neighboring chunks previously did not know about each other [S2, unofficial]. Construct the world cube by cube. Pass the neighbor's coordinates (if one exists) to the current cube. Check against each of the cube's sides as it is created to determine visibility [S2, unofficial]. Unity restricts a mesh to approximately 65,000 vertices. A Minecraft-style chunk of 65,536 cubes exceeds this limit, so a single chunk cannot be made. The solution is to recombine meshes by stacking multiple chunks on top of each other [S2, unofficial].

## Operating characteristics
- **Time Complexity**: O(N) where N is the number of voxels, as each voxel is processed once.
- **Memory Complexity**: O(F) where F is the number of visible faces, as only visible faces are meshed.
- **Performance**: Reduces triangle count significantly compared to naive meshing by merging adjacent faces.
- **Strengths**: Efficient for large voxel worlds; handles undefined blocks gracefully with assertions.
- **Limitations**: Requires careful handling of chunk boundaries and neighbor visibility; Unity vertex limit requires chunk stacking.

## Background
Originated from voxel-based procedural terrain generation techniques, specifically addressing the need for efficient meshing in game engines like Unity. The method was developed to handle the limitations of mesh vertex counts and the need for dynamic world generation [S2, unofficial].

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
