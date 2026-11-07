# Section-based Mesh Generation

Also called: Sectioned Chunk Meshing, Vertical Section Rebuild

## Overview
Section-based mesh generation renders a large voxel chunk as a vertical stack of smaller, independent sections to enable efficient partial rebuilds when voxels change [S1, unofficial]. It differs from full-chunk rebuilds by limiting computation to the minimal set of sections intersecting the modified voxel, and from adjacency-based chunk rebuilding by operating at the sub-chunk section level within a single chunk [S1, S2, unofficial].

## How it works
**Inputs and outputs:**
- Input: A chunk of voxels (e.g., $16 \times 16 \times 256$), a section side length $S$ (typically 16), and a section ID $id$.
- Output: A `MeshData` structure containing vertices, indices, normals, UVs, and colors for the specific section, which is uploaded to the rendering component.

**Data structures:**
The chunk's voxel data is stored in a 1D array `blocks` of size $N_{sections} \times (S+1)^3$. The extra dimension $(S+1)$ accounts for boundary padding required for neighbor checks.
- `blocks`: `TArray<BlockType>` or equivalent flat array.
- `MeshData`: Contains `vertices`, `Triangles` (indices), `normals`, `UV0`, `vertexColors`, `tangents`.

**Steps:**
1. **Calculate Section Bounds:** Determine the range of $i$ indices (height) for the current section.
   $$ i_{start} = id \times S $$
   $$ i_{end} = (id + 1) \times S $$
2. **Iterate Voxel Space:** Iterate through the section's volume. The source code iterates $j$ and $k$ (width/depth) from $0$ to $S$ (inclusive) to cover the full section including boundary padding [S1, unofficial].
3. **Generate Geometry:** For each voxel, determine visible faces (e.g., adjacent to air) and generate mesh primitives.
4. **Upload Mesh:** After generating `MeshData` for a section, upload it to the `UProceduralMeshComponent` using `CreateMeshSection_LinearColor` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_section_mesh(chunk_i, chunk_j, section_id, section_side, section_count, blocks):
    """
    Generates mesh data for a specific section of a chunk.
    """
    # Calculate range of i indices for the current section
    initial_i = section_id * section_side
    last_i = (section_id + 1) * section_side
    
    # Initialize mesh data containers
    vertices = []
    triangles = []
    normals = []
    uvs = []
    colors = []
    tangents = []
    
    # Iterate j and k from 0 to section_side (inclusive) to cover boundary padding
    for j in range(0, section_side + 1):
        for k in range(0, section_side + 1):
            # Iterate i through the section's height range
            for i in range(initial_i, last_i):
                # Get block type at (i, j, k)
                # Note: Index calculation depends on the specific 1D array layout
                # described in [S1, unofficial], typically involving sideSize = section_side + 1
                block_type = get_block_at(blocks, i, j, k, section_side)
                
                # Generate faces for visible voxels (e.g., neighbor is air)
                # This step involves calculating vertex positions, normals, UVs
                # based on the block type and face direction
                faces = generate_voxel_faces(block_type, i, j, k)
                
                for face in faces:
                    vertices.append(face.vertex)
                    triangles.append(face.index_offset)
                    normals.append(face.normal)
                    uvs.append(face.uv)
                    colors.append(face.color)
                    tangents.append(face.tangent)
                    
    # Create MeshData object
    mesh_data = MeshData(vertices, triangles, normals, uvs, colors, tangents)
    return mesh_data

def upload_section_to_component(mesh_component, section_id, mesh_data):
    """
    Uploads the generated mesh data to the procedural mesh component.
    """
    # Use the specific engine function to create/update the section
    # Arguments: section index, vertices, indices, normals, UVs, colors, tangents
    mesh_component.CreateMeshSection_LinearColor(
        section_id, 
        mesh_data.vertices, 
        mesh_data.triangles, 
        mesh_data.normals, 
        mesh_data.uvs, 
        mesh_data.colors, 
        mesh_data.tangents, 
        True
    )
```

**Parameters:**
- `sectionSide`: The width/depth of a section in voxels, typically 16 [S1, unofficial].
- `sectionCount`: The number of vertical sections in a chunk, calculated as $total\_height / sectionSide$ (e.g., $256 / 16 = 16$) [S1, unofficial].

**Edge cases and pitfalls:**
- **Boundary Padding:** The voxel array must include an extra row/column ($S+1$) to correctly handle neighbor checks for voxels on the section boundary [S1, unofficial].
- **Index Calculation:** Incorrect mapping from 3D coordinates $(i,j,k)$ to the 1D array index will cause rendering artifacts or crashes.

## Subtopics

### [Real-time Voxel Editing with Section Rebuild](4531-real-time-voxel-editing-with-section-rebuild.md)
This method enables immediate visual feedback for voxel placement and removal by regenerating mesh data for the affected 16x16x16 section and any adjacent sections if the edit occurs on a section boundary. It differs from full-chunk rebuilds by limiting computation to the minimal set of sections intersecting the modified voxel, and from adjacency-based chunk rebuilding by operating at the sub-chunk section level within a single chunk.

## Operating characteristics
- **Time Complexity:** $O(S^3)$ per section, where $S$ is the section side length. Partial rebuilds are $O(S^3)$ for the affected section(s) rather than $O(H \cdot W \cdot D)$ for the whole chunk.
- **Memory Complexity:** $O(N_{sections} \cdot S^3)$ for the voxel data structure with padding.
- **Performance:** Enables efficient partial updates. Full chunk rebuilds are avoided for minor edits.
- **Strengths:** Low latency for voxel edits; modular mesh generation.
- **Limitations:** Requires careful handling of section boundaries and neighbor lookups across section lines.

## Background
Originating from implementations like the UE5 Minecraft Clone [S1, unofficial], this method adapts chunk-based rendering to allow for dynamic voxel modification. It builds on [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by decomposing the chunk into manageable vertical slices.

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
