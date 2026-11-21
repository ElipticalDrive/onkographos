# VoxelMesherBlocky
Also called: Blocky mesher, Minecraft-like mesher
## Overview
VoxelMesherBlocky is a surface extraction method for blocky, voxel-based terrains that generates meshes by combining small geometric primitives corresponding to specific model IDs into larger chunks. It solves the problem of rendering large voxel worlds efficiently by culling faces that occlude each other, thereby reducing draw calls and vertex count without performing greedy meshing. It is distinguished from its siblings by its reliance on a predefined library of block models and its chunk-based assembly approach, similar to the technique used in Minecraft.
## How it works
**Inputs and Outputs:**
- Input: Voxel data stored in `VoxelBuffer.CHANNEL_TYPE`.
- Input: A `VoxelBlockyLibraryBase` resource containing a list of models (e.g., grass, dirt, wood) where the list index corresponds to the model ID.
- Output: A chunked mesh where faces occluding each other are culled.

**Data Structures:**
- `VoxelBlockyLibraryBase`: A resource holding a list of models. The index in this list serves as the unique ID for the model.
- `VoxelBuffer`: Stores the voxel data, specifically using the `CHANNEL_TYPE` channel.

**Steps:**
1. **Library Initialization**: Create or load a `VoxelBlockyLibraryBase` resource. This library contains the geometric definitions for all block types (e.g., grass, dirt, wood, leaves, water, shrubs, stairs, door parts) [S1].
2. **Voxel Data Access**: Read the voxel data from the `VoxelBuffer` using the `CHANNEL_TYPE` channel [S1].
3. **Model Identification**: For each voxel or group of voxels, identify the corresponding model ID based on the data in the buffer. The index in the library list corresponds to this ID [S1].
4. **Mesh Combination**: Combine small meshes corresponding to these model IDs into chunks. This involves assembling the geometric primitives defined in the library for the active models [S1].
5. **Face Culling**: Cull faces that occlude each other within the chunks. This reduces the number of visible faces but does not merge adjacent faces into larger quads (greedy meshing) [S1].

**Parameters:**
- Library: `VoxelBlockyLibraryBase` instance.
- Data Channel: `VoxelBuffer.CHANNEL_TYPE`.

**Edge Cases and Pitfalls:**
- The method does not perform greedy meshing, so adjacent identical faces are not merged into larger polygons, potentially leading to higher vertex counts compared to greedy methods [S1].
- The library must be properly initialized with all required models before meshing [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize the library
library = VoxelBlockyLibraryBase()
# Add models to the library (e.g., grass, dirt, wood)
# The index in the library list corresponds to the model ID
library.add_model(grass)
library.add_model(dirt)
library.add_model(wood)

# Access voxel data
voxel_buffer = get_voxel_buffer()
voxel_data = voxel_buffer.get_channel(VoxelBuffer.CHANNEL_TYPE)

# Process voxels into chunks
chunks = []
for chunk in voxel_chunks:
    chunk_mesh = Mesh()
    for voxel in chunk.voxels:
        model_id = voxel_data.get_id(voxel.position)
        if model_id is not None:
            model = library.get_model(model_id)
            # Add model geometry to chunk mesh
            chunk_mesh.add_geometry(model.geometry)
    # Cull occluding faces within the chunk
    chunk_mesh.cull_faces()
    chunks.append(chunk_mesh)

return chunks
```
## Subtopics

### Voxel Mesher Bake Process
This variant involves a baking process for the voxel mesher. It likely pre-processes or optimizes the mesh generation for static or semi-static voxel data. Specific details on the baking algorithm or optimizations are not provided in the sources.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.
**Measured Performance:**
Not given in the sources.
**Scaling:**
Not given in the sources.
**Strengths and Limitations:**
- Strengths: Efficiently reduces draw calls by culling occluding faces; simple to implement with a predefined library of models.
- Limitations: Does not perform greedy meshing, which may result in higher vertex counts compared to methods that merge adjacent faces.
**When to Use:**
Suitable for blocky, Minecraft-like terrains where a library of predefined block models is available and greedy meshing is not required.
**When Not to Use:**
Not suitable for terrains requiring greedy meshing or where memory efficiency is critical due to higher vertex counts.
## Background
VoxelMesherBlocky is part of the Voxel Tools documentation, focusing on blocky terrains similar to those found in Minecraft. It is designed to work with `VoxelBuffer.CHANNEL_TYPE` for voxel data storage and utilizes a `VoxelBlockyLibraryBase` for model definitions.
## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
