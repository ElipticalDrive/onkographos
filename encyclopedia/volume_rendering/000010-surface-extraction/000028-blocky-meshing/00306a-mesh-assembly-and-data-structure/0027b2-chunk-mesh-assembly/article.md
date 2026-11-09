# Chunk Mesh Assembly

Also called: Mesh Assembly, Mesh Buffer Packing, PackedMeshBuffers, Procedural Voxel Mesh Generation, Section-based Mesh Building, Unity Mesh Configuration

## Overview
Chunk Mesh Assembly aggregates mesh data from multiple rendering layers (e.g., opaque, water, transparent solid, fluids) into contiguous typed arrays for efficient GPU submission [S2, 4100-chunk-mesh-assembly.md, unofficial]. It solves the problem of fragmented geometry by mapping block types to materials and handling per-vertex attributes such as ambient occlusion and light levels within a unified buffer structure [S1, S3, 4100-chunk-mesh-assembly.md, unofficial]. Unlike [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md), which may use instancing or separate draw calls per block type, this method packs all quads for a chunk into a single mesh object with submesh descriptors [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel data store (block types), color/light store, chunk coordinate, and a map of block types to materials.
*   **Output:** A configured `Mesh` object (e.g., Unity `Mesh`) attached to a `MeshFilter` and `MeshCollider`, containing vertex positions, normals, indices, and material assignments.

**Data Structures:**
The core data structure for assembly is `PackedMeshBuffers`, which holds contiguous typed arrays [S2, unofficial]:
*   `positions`: `Float32Array` (3 floats per vertex: x, y, z).
*   `normals`: `Int8Array` (3 signed bytes per vertex: nx, ny, nz).
*   `blockIds`: `Uint16Array` (1 uint16 per vertex).
*   `ambientOcclusion`: `Uint8Array` (1 uint8 per vertex).
*   `blockLight`: `Uint8Array` (1 uint8 per vertex).
*   `skyLight`: `Uint8Array` (1 uint8 per vertex).
*   `indices`: `Uint32Array` (6 indices per quad, representing 2 triangles).
*   `groups`: `MeshBufferGroup[]` array tracking offsets per layer.

The `MeshBufferGroup` structure tracks the layout for each layer [S2, unofficial]:
*   `layer`: Enum identifying the render layer (e.g., Opaque, Water).
*   `vertexOffset`: Start index in the vertex arrays.
*   `vertexCount`: Number of vertices in this layer.
*   `indexOffset`: Start index in the indices array.
*   `indexCount`: Number of indices in this layer.

**Numbered Steps:**
1.  **Change Detection:** Check if the chunk has modified voxel data (`chunkProperties.HasChanges`). If false, skip assembly [S1, unofficial].
2.  **Mesh Generation:** Call the voxel mesher (e.g., `VoxelWorld.VoxelMesher.CreateMesh`) with the voxel data, color data, and chunk coordinate. If the result is null, return [S1, unofficial].
3.  **Quad Count Calculation:** Calculate the total number of quads across all defined `MESH_BUFFER_LAYERS` [S2, unofficial].
4.  **Buffer Allocation:** Allocate output arrays based on the total quad count [S2, unofficial]:
    *   Positions/Normals size: `quadCount * 4 * 3` (assuming 4 vertices per quad).
    *   Indices size: `quadCount * 6` (2 triangles per quad).
    *   Other attributes (blockIds, AO, lights): `quadCount * 4` per attribute.
5.  **Initialization:** Initialize a writer with offsets (`indexOffset: 0`, `vertexOffset: 0`) and the allocated output arrays [S2, unofficial].
6.  **Layer Iteration:** Iterate over each layer in `MESH_BUFFER_LAYERS` [S2, unofficial]:
    a.  Record current offsets.
    b.  Append quads for the current layer. For each quad, compute vertex positions from bounds, write vertex attributes (position, normal, blockId, AO, lights), and write 6 indices [S2, unofficial].
    c.  Look up the material for the block type. If not found, use a pre-scanned `FallbackMat` (the first non-null material in `BlockMaterials`) [S3, unofficial].
    d.  Record new offsets in `MeshBufferGroup`.
7.  **Mesh Configuration:** Configure the target mesh object [S1, unofficial]:
    a.  Set vertex buffer parameters with the total `vertexCount` and `MeshingVertexData.VertexBufferMemoryLayout`.
    b.  Set index buffer parameters with `vertexCount` and `IndexFormat.UInt16` (or UInt32 if vertex count exceeds 65535, though sources specify UInt16) [S1, unofficial].
    c.  Set submesh count to 1. Create a `SubMeshDescriptor` with the total index count and assign it to submesh 0.
    d.  Recalculate mesh bounds.
8.  **Assignment:** Assign the mesh to the chunk's `MeshFilter` and `MeshCollider`. Enable the collider and renderer. Reset `HasChanges` to false and set `IsMeshGenerated` to true [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assemble_chunk_mesh(chunk, voxel_data, color_data, block_materials):
    # Step 1: Check for changes
    if not chunk.properties.has_changes:
        return

    # Step 2: Generate mesh data
    mesh_job = voxel_data.mesher.create_mesh(voxel_data, color_data, chunk.coordinate)
    if mesh_job is None:
        return

    # Step 3: Calculate total quads
    total_quads = 0
    for layer in MESH_BUFFER_LAYERS:
        total_quads += mesh_job.layer_quad_count[layer]

    # Step 4: Allocate buffers
    # 4 vertices per quad, 3 components per vertex
    vertex_count = total_quads * 4
    positions = allocate_float_array(vertex_count * 3)
    normals = allocate_int8_array(vertex_count * 3)
    block_ids = allocate_uint16_array(vertex_count)
    ao = allocate_uint8_array(vertex_count)
    block_light = allocate_uint8_array(vertex_count)
    sky_light = allocate_uint8_array(vertex_count)
    indices = allocate_uint32_array(total_quads * 6)
    
    # Pre-scan for fallback material
    fallback_mat = None
    for mat in block_materials.values():
        if mat is not None:
            fallback_mat = mat
            break

    # Step 5 & 6: Iterate layers and write data
    groups = []
    current_vertex_offset = 0
    current_index_offset = 0

    for layer in MESH_BUFFER_LAYERS:
        start_v_offset = current_vertex_offset
        start_i_offset = current_index_offset
        
        quads = mesh_job.layer_quads[layer]
        layer_vertex_count = len(quads) * 4
        layer_index_count = len(quads) * 6

        for i, quad in enumerate(quads):
            # Compute vertex positions from quad bounds
            verts = compute_quad_vertices(quad, chunk.coordinate)
            
            # Write positions (3 floats per vertex)
            for v in range(4):
                positions[(current_vertex_offset + v) * 3 + 0] = verts[v].x
                positions[(current_vertex_offset + v) * 3 + 1] = verts[v].y
                positions[(current_vertex_offset + v) * 3 + 2] = verts[v].z
            
            # Write normals (3 signed bytes per vertex)
            for v in range(4):
                normals[(current_vertex_offset + v) * 3 + 0] = int8(quad.normal.x)
                normals[(current_vertex_offset + v) * 3 + 1] = int8(quad.normal.y)
                normals[(current_vertex_offset + v) * 3 + 2] = int8(quad.normal.z)
            
            # Write attributes (1 per vertex)
            for v in range(4):
                block_ids[current_vertex_offset + v] = quad.block_id
                ao[current_vertex_offset + v] = quad.ao if quad.ao is not None else 0 # NO_AMBIENT_OCCLUSION
                block_light[current_vertex_offset + v] = quad.block_light if quad.block_light is not None else 0 # NO_LIGHT
                sky_light[current_vertex_offset + v] = quad.sky_light if quad.sky_light is not None else 0

            # Write indices (6 indices per quad: triangle fan 0-1-2, 0-2-3)
            base_idx = current_index_offset
            indices[base_idx + 0] = current_vertex_offset + 0
            indices[base_idx + 1] = current_vertex_offset + 1
            indices[base_idx + 2] = current_vertex_offset + 2
            indices[base_idx + 3] = current_vertex_offset + 0
            indices[base_idx + 4] = current_vertex_offset + 2
            indices[base_idx + 5] = current_vertex_offset + 3
            
            current_vertex_offset += 4
            current_index_offset += 6

        # Material assignment logic (conceptual, as mesh assembly usually handles materials via submeshes or vertex colors)
        # In this specific implementation, materials are looked up per block type, 
        # but the final mesh object uses a single material or submesh mapping.
        # The source implies material lookup happens during quad processing for validation/fallback.
        
        groups.append({
            'layer': layer,
            'vertexOffset': start_v_offset,
            'vertexCount': layer_vertex_count,
            'indexOffset': start_i_offset,
            'indexCount': layer_index_count
        })

    # Step 7: Configure Mesh Object
    mesh = create_mesh()
    mesh.set_vertex_buffer(positions, normals, block_ids, ao, block_light, sky_light)
    mesh.set_index_buffer(indices)
    mesh.set_submesh_count(1)
    mesh.set_submesh(0, SubMeshDescriptor(index_count=current_index_offset, topology=TriangleList))
    mesh.recalculate_bounds()

    # Step 8: Assign to Chunk
    chunk.mesh_filter.mesh = mesh
    chunk.mesh_renderer.enabled = True
    chunk.mesh_collider.enabled = True
    chunk.mesh_collider.shared_mesh = mesh
    chunk.properties.has_changes = False
    chunk.properties.is_mesh_generated = True
```

**Parameters:**
*   `NO_AMBIENT_OCCLUSION`: Constant value `0` used when AO is not provided [S2, unofficial].
*   `NO_LIGHT`: Constant value `0` used when light data is not provided [S2, unofficial].
*   `BlockSize`: Static constexpr float defining world units per voxel edge. Default value is `100.f` [S3, unofficial].
*   `FallbackMat`: `UMaterialInterface*` assigned to the first non-null material found in `BlockMaterials` map during initialization [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Sections:** If a layer has no quads, the vertex array for that layer is empty. The implementation must skip creating mesh sections or submeshes with zero vertices to avoid invalid mesh state [S3, unofficial].
*   **Missing Materials:** If a block type is not found in the `BlockMaterials` map, the system must not dereference a null pointer. It falls back to `FallbackMat` [S3, unofficial].
*   **Index Format:** The sources specify `UInt16` for indices [S1, unofficial]. If the vertex count exceeds 65,535, this will overflow. The implementer must check `vertexCount` and switch to `UInt32` if necessary, or ensure chunk sizes limit vertex counts.

## Subtopics

### [Section-based Mesh Generation](4220-section-based-mesh-generation.md)
Renders a large voxel chunk as a vertical stack of smaller, independent sections to enable efficient partial rebuilds when voxels change. It differs from full-chunk rebuilds by limiting computation to the minimal set of sections intersecting the modified voxel, and from adjacency-based chunk rebuilding by operating at the sub-chunk section level within a single chunk.

### [Chunk Region Meshing with Dirty Regions](4402-chunk-region-meshing-with-dirty-regions.md)
An incremental surface extraction method that generates geometry for a voxel chunk by processing only a specified subset of "dirty" cells, while consulting the complete chunk and its neighbors for visibility, ambient occlusion (AO), and fluid data. It solves the problem of expensive full-chunk remeshing by allowing partial updates, ensuring that only changed geometry is regenerated. Unlike Section-based Mesh Generation, which processes the entire chunk layer by layer, this method restricts computation to a local `dirtyRegion`, significantly reducing work for sparse updates.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as every voxel face is potentially checked and written. The assembly step itself is linear with respect to the number of generated quads.
*   **Memory Complexity:** $O(Q)$ where $Q$ is the number of quads. The buffers scale directly with the surface area of the chunk.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales with the number of exposed faces. Greedy meshing (often a precursor to assembly) can reduce $Q$ significantly by merging coplanar quads, reducing memory bandwidth and draw call overhead [S2, unofficial].
*   **Strengths:** Contiguous memory layout enables efficient GPU submission. Layer separation allows for different rendering passes (e.g., water transparency) without complex shader logic.
*   **Limitations:** Full chunk rebuilds are expensive. Memory usage can be high for highly detailed terrain. Index buffer size is limited by the chosen index format (UInt16 vs UInt32).
*   **When to use:** Suitable for voxel games where chunks are relatively small and static or change infrequently.
*   **When not to use:** Unsuitable for highly dynamic scenes with frequent voxel changes per frame, where incremental methods like Dirty Regions are preferred.

## Background
Chunk Mesh Assembly is a standard technique in procedural voxel rendering, particularly in engines like Unity. It is used in projects such as Eldemarkki's Marching-Cubes-Terrain [S1, unofficial], nerima-games' mc-meshing [S2, unofficial], and HawkDev15's VoxelWorld [S3, unofficial]. The method evolved from the need to efficiently submit large amounts of geometry generated by surface extraction algorithms (like Marching Cubes or Greedy Meshing) to the GPU.

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S3] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
