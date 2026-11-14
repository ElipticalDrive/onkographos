# Chunk Mesh Creation

Also called: Chunk Face Culling and Mesh Assembly

## Overview
Chunk Mesh Creation generates the final visible geometry for a voxel chunk by identifying exposed block faces and assembling them into a single draw call. It solves the problem of rendering only the surfaces of a voxel chunk that are not occluded by adjacent blocks or neighboring chunks, thereby reducing overdraw and triangle count. It differs from its parent [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md), which produces individual quads, by handling the spatial culling logic across chunk boundaries and combining multiple face meshes into one unified mesh structure.

## How it works
### Inputs and Outputs
- **Inputs**:
  - `chunkData`: A structure containing the voxel block data.
  - `chunkObj`: The chunk object instance, providing transform and position.
  - `_loadedChunkObjs`: A dictionary or map of loaded neighboring chunk objects, used to determine if adjacent chunks exist.
- **Outputs**:
  - A single `Mesh` object containing all visible faces for the chunk.

### Data Structures
- **Block Data Layout**: The chunk's block data is stored as a singular `byte[]` array of 4096 indices [S1, unofficial]. This corresponds to a $16 \times 16 \times 16$ voxel grid.
- **Block Index to Position**: For a block at linear index $i$ in the array, the 3D position $(x, y, z)$ is calculated as:
  $$
  x = i \pmod{16}
  $$
  $$
  y = \lfloor (i \pmod{256}) / 16 \rfloor
  $$
  $$
  z = \lfloor i / 256 \rfloor
  $$
  Note: The source code uses `Mathf.Floor(i % 256)` which is equivalent to `i % 256` for positive integers, and `Mathf.Floor((i % 256) / 16)` for the Y coordinate [S1, unofficial]. The Z coordinate is derived from `i // 256`.

### Algorithm Steps
1. **Initialization**: Create an empty `Mesh` object and a list `meshesToCombine` to hold individual face meshes.
2. **Iteration**: Loop through each block index $i$ from $0$ to $4095$ (length of `chunkData.blocks`).
3. **Block Check**: If `chunkData.blocks[i]` is non-zero (solid block), proceed to culling checks.
4. **Culling Checks**: Determine if the block has an exposed face in any of the six directions. A face is exposed if:
   - The adjacent voxel is air (value 0), OR
   - The adjacent chunk does not exist in `_loadedChunkObjs`.
   
   The specific checks are:
   - **Top (Y+)**: If $i \ge 256$ (top layer of the chunk) AND the chunk above (`chunkPosition + (0, 1, 0)`) is not in `_loadedChunkObjs`, add Top face.
   - **Bottom (Y-)**: If $i < 256$ (bottom layer) AND the chunk below (`chunkPosition + (0, -1, 0)`) is not in `_loadedChunkObjs`, add Bottom face.
   - **East (Z+)**: If $(i \pmod{256}) < 16$ (East layer) AND the chunk to the East (`chunkPosition + (0, 0, 1)`) is not in `_loadedChunkObjs`, add East face.
   - **West (Z-)**: If $(i \pmod{256}) \ge 240$ (West layer) AND the chunk to the West (`chunkPosition + (0, 0, -1)`) is not in `_loadedChunkObjs`, add West face.
   - **North (X+)**: If $i \pmod{16} == 0$ (North layer) AND the chunk to the North (`chunkPosition + (1, 0, 0)`) is not in `_loadedChunkObjs`, add North face.
   - **South (X-)**: If $i \pmod{16} == 1$ (South layer) AND the chunk to the South (`chunkPosition + (-1, 0, 0)`) is not in `_loadedChunkObjs`, add South face.
   - **Internal Top (Y+)**: If $i + 256 < 4096$ AND `chunkData.blocks[i + 256] == 0`, add Top face.
   - **Internal Bottom (Y-)**: If $i - 256 \ge 0$ AND `chunkData.blocks[i - 256] == 0`, add Bottom face.
   - **Internal North (X+)**: If $i + 1 < 4096$ AND `chunkData.blocks[i + 1] == 0`, add North face.
   - **Internal South (X-)**: If $i - 1 \ge 0$ AND `chunkData.blocks[i - 1] == 0`, add South face.
   - **Internal West (Z-)**: If $i + 16 < 4096$ AND `chunkData.blocks[i + 15] == 0`, add West face.
   - **Internal East (Z+)**: If $i - 16 \ge 0$ AND `chunkData.blocks[i - 16] == 0`, add East face.

   *Note: The source code checks both adjacent chunk existence AND internal neighbor values. If a neighbor chunk exists, it assumes the face is occluded or handled by that chunk, so it does not add the face. If the neighbor chunk does not exist, it adds the face regardless of the internal neighbor's value (effectively treating missing chunks as air/empty space for culling purposes).*

5. **Face Generation**: For each exposed face, call `GetMeshDataFromFace(faceDirection, blockPos)` to generate a temporary mesh containing the quad vertices, indices, and normals.
6. **Combination**: Create a `CombineInstance` array of size `meshesToCombine.Count`. For each mesh in the list, set `combine[i].mesh` to the mesh and `combine[i].transform` to `chunkObj.transform.localToWorldMatrix`. Call `mesh.CombineMeshes(combine)` to merge all faces into the final chunk mesh.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_chunk_mesh(chunk_obj, chunk_data, loaded_chunk_objs):
    """
    Generates a combined mesh for a voxel chunk by culling internal faces
    and merging exposed faces from neighboring chunks.
    
    Args:
        chunk_obj: Object containing transform and chunk position.
        chunk_data: Object containing blocks array (byte[], size 4096).
        loaded_chunk_objs: Dictionary mapping chunk positions to chunk objects.
        
    Returns:
        Mesh: The combined mesh for the chunk.
    """
    mesh = new Mesh()
    meshes_to_combine = []
    blocks = chunk_data.blocks
    chunk_pos = chunk_obj.chunk_position
    
    # Iterate through all 4096 blocks
    for i in range(4096):
        if blocks[i] == 0:
            continue
            
        # Calculate 3D position from linear index
        x = i % 16
        y = (i % 256) // 16
        z = i // 256
        block_pos = Vector3(x, y, z)
        
        # Check Top (Y+)
        if i >= 256:
            if (chunk_pos + Vector3Int(0, 1, 0)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.top, block_pos))
        else:
            # Internal check for top neighbor
            if blocks[i + 256] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.top, block_pos))
                
        # Check Bottom (Y-)
        if i < 256:
            if (chunk_pos + Vector3Int(0, -1, 0)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.bottom, block_pos))
        else:
            # Internal check for bottom neighbor
            if blocks[i - 256] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.bottom, block_pos))
                
        # Check East (Z+)
        if (i % 256) < 16:
            if (chunk_pos + Vector3Int(0, 0, 1)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.east, block_pos))
        else:
            # Internal check for east neighbor
            if blocks[i - 16] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.east, block_pos))
                
        # Check West (Z-)
        if (i % 256) >= 240:
            if (chunk_pos + Vector3Int(0, 0, -1)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.west, block_pos))
        else:
            # Internal check for west neighbor
            if blocks[i + 16] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.west, block_pos))
                
        # Check North (X+)
        if i % 16 == 0:
            if (chunk_pos + Vector3Int(1, 0, 0)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.north, block_pos))
        else:
            # Internal check for north neighbor
            if blocks[i - 1] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.north, block_pos))
                
        # Check South (X-)
        if i % 16 == 1:
            if (chunk_pos + Vector3Int(-1, 0, 0)) not in loaded_chunk_objs:
                meshes_to_combine.append(get_mesh_data_from_face(Face.south, block_pos))
        else:
            # Internal check for south neighbor
            if blocks[i + 1] == 0:
                meshes_to_combine.append(get_mesh_data_from_face(Face.south, block_pos))

    # Combine all meshes
    if len(meshes_to_combine) == 0:
        return mesh
        
    combine_instances = []
    for face_mesh in meshes_to_combine:
        instance = CombineInstance()
        instance.mesh = face_mesh
        instance.transform = chunk_obj.transform.local_to_world_matrix
        combine_instances.append(instance)
        
    mesh.combine_meshes(combine_instances)
    return mesh

def get_mesh_data_from_face(face, pos):
    """
    Generates a mesh for a single face of a block.
    """
    mesh = new Mesh()
    normals = []
    
    if face == Face.top:
        mesh.vertices = [
            pos + Vector3(0, 1, 0),
            pos + Vector3(1, 1, 0),
            pos + Vector3(1, 1, 1),
            pos + Vector3(0, 1, 1)
        ]
        mesh.triangles = [0, 3, 2, 2, 1, 0]
        normals = [Vector3.up] * 4
    elif face == Face.bottom:
        mesh.vertices = [
            pos + Vector3(0, 0, 0),
            pos + Vector3(1, 0, 0),
            pos + Vector3(1, 0, 1),
            pos + Vector3(0, 0, 1)
        ]
        mesh.triangles = [0, 2, 1, 1, 2, 3] # Note: Source snippet was cut off, inferred standard quad winding
        normals = [Vector3.down] * 4
    elif face == Face.west:
        mesh.vertices = [
            pos + Vector3(0, 0, 0),
            pos + Vector3(0, 1, 0),
            pos + Vector3(0, 1, 1),
            pos + Vector3(0, 0, 1)
        ]
        mesh.triangles = [2, 1, 0, 0, 3, 2]
        normals = [Vector3.back] * 4
    elif face == Face.east:
        mesh.vertices = [
            pos + Vector3(1, 0, 0),
            pos + Vector3(1, 1, 0),
            pos + Vector3(1, 1, 1),
            pos + Vector3(1, 0, 1)
        ]
        mesh.triangles = [0, 1, 2, 2, 3, 0]
        normals = [Vector3.forward] * 4
    elif face == Face.north:
        # Inferred from context: North is X+
        mesh.vertices = [
            pos + Vector3(1, 0, 0),
            pos + Vector3(1, 1, 0),
            pos + Vector3(1, 1, 1),
            pos + Vector3(1, 0, 1)
        ]
        mesh.triangles = [0, 1, 2, 2, 3, 0]
        normals = [Vector3.right] * 4
    elif face == Face.south:
        # Inferred from context: South is X-
        mesh.vertices = [
            pos + Vector3(0, 0, 0),
            pos + Vector3(0, 1, 0),
            pos + Vector3(0, 1, 1),
            pos + Vector3(0, 0, 1)
        ]
        mesh.triangles = [2, 1, 0, 0, 3, 2]
        normals = [Vector3.left] * 4
        
    mesh.set_normals(normals)
    return mesh
```

### Edge Cases and Pitfalls
- **Boundary Conditions**: The checks for adjacent chunks rely on `_loadedChunkObjs`. If a neighbor chunk is not loaded, the face is always drawn. This can lead to double-rendering if multiple chunks claim the same boundary face. The source code assumes that only one chunk (the one with the solid block) generates the face, but since it checks for the *existence* of the neighbor rather than the neighbor's block type, it may generate faces even if the neighbor chunk exists but has air at the boundary. This is a potential pitfall: if a neighbor chunk exists, the face is *not* generated, assuming the neighbor will handle it. If the neighbor chunk exists but has air, the face is still not generated by this chunk, which is correct. However, if the neighbor chunk is not loaded, the face is generated. This implies that the system relies on chunk loading order or a central manager to ensure consistency.
- **Index Out of Bounds**: The internal neighbor checks (e.g., `i + 256`) must be bounded by the array length (4096). The source code uses `if i + 256 <= chunkData.blocks.Length`, which is correct.
- **Winding Order**: The source code provides specific triangle indices for each face. Deviating from these indices will result in inverted normals or back-face culling issues.
- **Transform Application**: The `CombineInstance.transform` is set to `localToWorldMatrix`. This ensures that the combined mesh is in world space, which is necessary if the chunk moves or if the renderer expects world-space coordinates.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of blocks in the chunk (4096). Each block is visited once, and a constant number of checks are performed.
- **Memory Complexity**: $O(F)$ where $F$ is the number of exposed faces. Each exposed face generates a small mesh (4 vertices, 6 indices). The final combined mesh has size proportional to $F$.
- **Performance**: The method is efficient for static chunks. However, creating many small meshes and combining them can be costly if done frequently. The source code creates a new `Mesh` object for each face, which involves memory allocation. For high-performance applications, a more optimized approach would pre-allocate buffers and write directly to them.
- **Strengths**: Simple to implement; handles chunk boundaries correctly; reduces overdraw by culling internal faces.
- **Limitations**: Not suitable for dynamic voxel changes without regeneration; memory overhead from creating many small mesh objects; relies on external chunk loading state for culling.

## Background
This method is derived from a Unity C# implementation for generating custom voxel terrain meshes [S1, unofficial]. It is commonly used in voxel-based games and simulations where chunk-based rendering is necessary to manage large worlds. The approach of checking adjacent chunks and combining meshes is a standard technique in voxel engine development.

## Sources
[S1] Unity C# Generating custom mesh for voxel terrain not loading properly, https://stackoverflow.com/questions/72852301/unity-c-generating-custom-mesh-for-voxel-terrain-not-loading-properly, fetched 2026-10-27 (unofficial)
