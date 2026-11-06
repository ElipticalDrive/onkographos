# Section-based Mesh Generation

Also called: Sectioned Chunk Meshing, Vertical Section Rebuild

## Overview
Section-based mesh generation is a method for constructing surface meshes from large voxel chunks by dividing the volume into smaller, manageable vertical slabs (sections). It solves the performance bottleneck of regenerating an entire large chunk mesh when only a small portion of voxels changes. By isolating geometry generation to specific vertical ranges, the method allows for efficient partial rebuilds, where only the affected sections and their immediate neighbors are regenerated. It differs from full-chunk regeneration by limiting computational scope to a fraction of the total volume, and from region-based dirty-flagging by operating on fixed vertical slices rather than arbitrary 3D bounding boxes.

## How it works
**Inputs and Outputs**
- **Input**: A voxel chunk defined by dimensions $(W, W, H)$, where $W$ is the width/depth and $H$ is the height. The chunk is logically divided into $N$ vertical sections.
- **Input**: A 3D array of block types, padded to accommodate boundary checks.
- **Output**: A list of mesh data structures (vertices, indices, normals, UVs, tangents, colors), one per section.

**Data Structures**
The voxel data is stored in a flattened 1D array representing a 3D grid. To simplify neighbor lookups at section boundaries, the grid includes padding.
- **Layout**: The array size is $N \times (W+1) \times (W+1) \times (W+1)$.
- **Padding**: An extra row and column are added to the width and depth dimensions to handle edge cases where faces are adjacent to the chunk boundary or adjacent chunks.
- **Indexing**: A position $(i, j, k)$ in the logical grid maps to the 1D array index using a helper function `GetIJKFromPositionInTArray`. The exact stride depends on the implementation, but typically follows row-major order for the padded dimensions.

**Parameters**
- `sectionSide`: The width and depth of a section in voxels. Typically 16 [S1, unofficial].
- `sectionHeight`: The height of a section in voxels. Typically 16 [S1, unofficial].
- `sectionCount`: The number of vertical sections in a chunk, calculated as $H / \text{sectionHeight}$. For a 256-unit high chunk, this is 16 [S1, unofficial].

**Algorithm Steps**
1. **Initialize Chunk Data**:
   - Determine `sectionSide` and `sectionCount`.
   - Allocate the block array with size `sectionCount * (sectionSide + 1)^3` to include padding [S1, unofficial].
   - Populate the array with voxel data.

2. **Iterate Sections**:
   - For each section index `s` from 0 to `sectionCount - 1`:
     - Calculate the vertical range of indices for this section. Let `sectionSide` be the height of the section.
     - `initialI = s * sectionSide`
     - `lastI = (s + 1) * sectionSide`
     - Note: The iteration must cover the full section including boundary padding to ensure faces on the top and bottom of the section are correctly generated relative to adjacent sections [S1, unofficial].

3. **Generate Mesh Data for Section**:
   - Call a mesh generation function (e.g., `GetMeshData`) with the section index, total section count, block array, and section side length.
   - Inside the mesh generation function:
     - Iterate through the voxels within the section's vertical range.
     - For each voxel, check its 6 neighbors.
     - If a neighbor is air (or a transparent block type), generate a face.
     - Apply neighbor-based face culling to eliminate interior geometry [S1, unofficial].
     - Compute vertex positions, indices, normals, UVs (using a texture atlas), and tangents.
     - Store the result in a `MeshData` structure.

4. **Upload to GPU**:
   - For each generated `MeshData` structure:
     - Upload the data to the procedural mesh component (e.g., `UProceduralMeshComponent` in Unreal Engine).
     - Use a function like `CreateMeshSection_LinearColor` to create a new mesh section with the calculated vertices, indices, normals, UVs, and colors [S1, unofficial].
     - Enable physics collision data if required (`ContainsPhysicsTriMeshData(true)`) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk_i, chunk_j, section_side_width, number_of_sections, populate_block_func):
    """
    Generates mesh data for all sections of a chunk.
    """
    side_width = section_side_width
    section_count = number_of_sections
    
    # Allocate block array with padding
    side_size = section_side_width + 1
    blocks = allocate_array(section_count * side_size * side_size * side_size)
    
    # Populate blocks
    for i in range(section_count * side_size * side_size * side_size):
        # Helper to get IJK from 1D index
        i_idx, j_idx, k_idx = get_ijk_from_position(i, section_side_width)
        blocks[i] = populate_block_func(chunk_i, chunk_j, i_idx, j_idx, k_idx)
    
    chunk_mesh_data = []
    
    # Iterate over each vertical section
    for section in range(section_count):
        # Get mesh data for this specific section
        mesh_data = get_mesh_data_for_section(
            chunk_i, chunk_j, section, section_count, blocks, section_side_width
        )
        chunk_mesh_data.append(mesh_data)
        
    return chunk_mesh_data

def get_mesh_data_for_section(chunk_i, chunk_j, section, section_count, blocks, section_side):
    """
    Generates mesh geometry for a single vertical section.
    """
    # Calculate vertical range for this section
    # Note: The source implies iterating j and k from 0 to sectionSide (inclusive)
    # and i based on the section ID.
    initial_i = section * section_side
    last_i = (section + 1) * section_side
    
    vertices = []
    triangles = []
    normals = []
    uvs = []
    colors = []
    tangents = []
    
    # Iterate through the section's volume
    # The source mentions iterating j and k from 0 to sectionSide (inclusive)
    # This suggests the padding is handled within the iteration bounds or by the neighbor check logic.
    for i in range(initial_i, last_i + 1): # +1 to include boundary if needed
        for j in range(section_side + 1): # +1 for padding
            for k in range(section_side + 1): # +1 for padding
                # Get block type at this position
                block_type = blocks[get_index(i, j, k, section_side + 1)]
                
                # Check neighbors for face generation
                # If neighbor is air, add face
                # ... (standard marching cubes or face culling logic) ...
                pass
                
    return MeshData(vertices, triangles, normals, uvs, colors, tangents)
```

**Edge Cases and Pitfalls**
- **Boundary Padding**: Failure to pad the voxel array by at least one voxel on all sides can lead to out-of-bounds memory access when checking neighbors at the chunk edges [S1, unofficial].
- **Section Boundaries**: Faces generated at the top or bottom of a section must be consistent with the adjacent section. The mesh generation for each section must be independent but logically continuous.
- **Precision**: Floating-point precision issues may arise when converting voxel coordinates to world space, especially for large chunks. Using integer arithmetic for voxel indices and converting to floats only at the vertex stage is recommended.

## Subtopics

### [Real-time Voxel Editing with Section Rebuild](4531-real-time-voxel-editing-with-section-rebuild.md)
Updating the mesh immediately when a voxel is placed or removed by regenerating the mesh data for the affected section and any adjacent sections if the edit is on a boundary.

## Operating characteristics
- **Time Complexity**: $O(W^2 \cdot H)$ for full chunk generation, where $W$ is width/depth and $H$ is height. Partial rebuilds are $O(W^2 \cdot \text{sectionHeight})$, offering a significant speedup for small edits.
- **Memory Complexity**: $O(W^2 \cdot H)$ for the voxel data structure. The mesh data is proportional to the surface area, which is typically much smaller than the volume.
- **Performance**: Efficient for large, tall chunks (e.g., 16x16x256) where only small portions change. The fixed section size (16x16x16) allows for predictable memory allocation and cache-friendly access patterns.
- **Strengths**: Enables real-time editing of large voxel worlds. Reduces GPU upload overhead by only updating changed sections.
- **Limitations**: Requires careful handling of section boundaries to avoid gaps or overlaps. Not suitable for highly dynamic scenes with frequent, widespread changes, as the overhead of managing multiple sections may outweigh the benefits.

## Background
This method is derived from implementations in voxel-based games, specifically those aiming to replicate Minecraft-like gameplay in modern game engines like Unreal Engine 5. The approach of splitting large chunks into smaller sections for efficient partial updates is a common optimization technique in voxel rendering. The specific implementation details, such as the use of `UProceduralMeshComponent` and the 16x16x16 section size, are documented in community-driven projects [S1, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
