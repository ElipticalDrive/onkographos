# Chunk Mesh Assembly

Also called: Mesh Assembly, Mesh Buffer Packing, PackedMeshBuffers, Procedural Voxel Mesh Generation, Section-based Mesh Building, Unity Mesh Configuration

## Overview
Chunk Mesh Assembly aggregates mesh data generated from multiple voxel layers (e.g., opaque, water, transparent solid) into contiguous typed arrays for efficient GPU submission [S2, unofficial]. It solves the problem of fragmented geometry by consolidating per-layer mesh data into a single renderable structure, distinct from sibling methods that focus on plugin architecture [4921-mesh-plugin-architecture.md] or instance generation [6209-block-render-data-structure-instance-generation.md].

## How it works
**Inputs and Outputs:**
*   **Input:** Voxel data store, color store, chunk coordinate, and a map of block types to materials [S1, S3, unofficial].
*   **Output:** A unified `Mesh` object (or equivalent engine structure) containing positions, normals, indices, and attributes, assigned to the chunk's renderer and collider [S1, S3, unofficial].

**Data Structures:**
*   **PackedMeshBuffers:** Contains contiguous arrays for `positions` (Float32), `normals` (Int8), `blockIds` (Uint16), `ambientOcclusion` (Uint8), `blockLight` (Uint8), `skyLight` (Uint8), `indices` (Uint32), and `groups` (array of `MeshBufferGroup`) [S2, unofficial].
*   **MeshBufferGroup:** `{ layer: MeshBufferLayer, vertexOffset: number, vertexCount: number, indexOffset: number, indexCount: number }` [S2, unofficial].
*   **FChunkMeshSection:** Stores per-block-type mesh data: `Vertices` (TArray<FVector>), `Triangles` (TArray<int32>), `Normals` (TArray<FVector>), `UVs` (TArray<FVector2D>) [S3, unofficial].
*   **FActiveChunk:** Tracks runtime state: `MeshComp` (ProceduralMeshComponent), `PendingCollision`, `CollisionSectionIndex`, `bHasCollision`, `bCastsShadow` [S3, unofficial].

**Parameters:**
*   `BlockSize`: World units per voxel edge. Default: 100.f [S3, unofficial].
*   `NO_AMBIENT_OCCLUSION`: Constant 0 when AO is unavailable [S2, unofficial].
*   `NO_LIGHT`: Constant 0 when light data is unavailable [S3, unofficial].
*   `FallbackMat`: First non-null material in `BlockMaterials` map, used if a block type is missing [S3, unofficial].

**Steps:**
1.  Check if the chunk has changes (`chunkProperties.HasChanges`). If true, proceed to `GenerateMeshImmediate` [S1, unofficial].
2.  Calculate total quad count across all layers in `MESH_BUFFER_LAYERS` [S2, unofficial].
3.  Clear existing mesh sections on the component [S3, unofficial].
4.  Call `VoxelWorld.VoxelMesher.CreateMesh` with voxel data, color store, and chunk coordinate. If result is null, return [S1, unofficial].
5.  Allocate output arrays:
    *   Positions/Normals: `quadCount * 4 * 3` elements [S2, unofficial].
    *   Indices: `quadCount * 6` elements [S2, unofficial].
    *   Attributes: `quadCount * 4` elements [S2, unofficial].
6.  Iterate over `Prepared.Sections` map. Skip sections where `Section.Vertices.Num() == 0` to avoid empty mesh sections [S3, unofficial].
7.  Calculate `vertexCount` as `job.VertexCountCounter.Count * 3`. Dispose the counter [S1, unofficial].
8.  Initialize writer with offsets (`indexOffset: 0`, `vertexOffset: 0`) and output arrays [S2, unofficial].
9.  For each layer in `MESH_BUFFER_LAYERS`:
    *   Record current offsets.
    *   Append quads for that layer.
    *   Record new offsets in `MeshBufferGroup` [S2, unofficial].
10. Look up material for the block type. If not found, use `FallbackMat` [S3, unofficial].
11. Set vertex and index buffer data from job outputs (`job.OutputVertices`, `job.OutputTriangles`). Dispose job output arrays [S1, unofficial].
12. For opaque/water/transparentSolid layers:
    *   Iterate quads.
    *   Compute vertices from quad bounds.
    *   Write vertex data (position, normal, blockId, ao, lights).
    *   Write 6 indices (triangle fan: 0-1-2, 0-2-3).
    *   Update offsets [S2, S3, unofficial].
13. Create mesh section using `CreateMeshSection` with vertices, triangles, normals, UVs, empty color/tangent arrays, and `bUpdateBounds=false` [S3, unofficial].
14. Increment `SectionIndex` after each successfully created section [S3, unofficial].
15. Configure Unity Mesh:
    *   Set vertex buffer params with `vertexCount` and `MeshingVertexData.VertexBufferMemoryLayout` [S1, unofficial].
    *   Set index buffer params with `vertexCount` and `IndexFormat.UInt16` [S1, unofficial].
    *   Set submesh count to 1. Create `SubMeshDescriptor` with index count `vertexCount` and set as submesh 0. Recalculate bounds [S1, unofficial].
16. Assign component and metadata to `OutChunk`: `MeshComp`, `CollisionSectionIndex` (points to next available section), `PendingCollision`, `bHasCollision=false`, `bCastsShadow=false` [S3, unofficial].
17. Assign the mesh to the chunk's `MeshFilter` and `MeshCollider`. Enable collider and renderer. Reset `HasChanges` to false and set `IsMeshGenerated` to true [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ChunkMeshAssembly(chunk_properties, voxel_data, color_store, chunk_coordinate, block_materials):
    if not chunk_properties.HasChanges:
        return

    # 1. Generate Mesh Data
    mesh_result = VoxelMesher.CreateMesh(voxel_data, color_store, chunk_coordinate)
    if mesh_result is None:
        return

    # 2. Calculate Totals
    total_quads = 0
    for layer in MESH_BUFFER_LAYERS:
        total_quads += layer.quad_count

    # 3. Allocate Buffers
    # Positions: Float32, Normals: Int8, Indices: Uint32, Attributes: Uint8/Uint16
    positions = allocate_array(total_quads * 4 * 3)
    normals = allocate_array(total_quads * 4 * 3, type=Int8)
    indices = allocate_array(total_quads * 6, type=Uint32)
    attributes = allocate_array(total_quads * 4)

    # 4. Prepare Sections
    sections = mesh_result.Prepared.Sections
    valid_sections = [s for s in sections if s.Vertices.Num() > 0]
    
    vertex_count = mesh_result.job.VertexCountCounter.Count * 3
    mesh_result.job.VertexCountCounter.Dispose()

    # 5. Write Data
    index_offset = 0
    vertex_offset = 0
    section_index = 0
    mesh_groups = []

    for layer in MESH_BUFFER_LAYERS:
        layer_start_index = index_offset
        layer_start_vertex = vertex_offset
        
        for quad in layer.quads:
            # Resolve material
            mat = block_materials.get(quad.block_type)
            if mat is None:
                mat = FallbackMat # First non-null in BlockMaterials

            # Compute vertices from quad bounds
            verts = compute_quad_vertices(quad, BlockSize)
            
            # Write positions (3 floats per vertex)
            for v in verts:
                positions[vertex_offset] = v.x
                positions[vertex_offset + 1] = v.y
                positions[vertex_offset + 2] = v.z
                vertex_offset += 3

            # Write normals (3 signed bytes per vertex)
            for v in verts:
                normals[vertex_offset - 3] = v.nx
                normals[vertex_offset - 2] = v.ny
                normals[vertex_offset - 1] = v.nz

            # Write attributes (blockId, ao, lights)
            attributes[vertex_offset // 3] = quad.block_id
            attributes[vertex_offset // 3 + 1] = quad.ao if quad.ao else NO_AMBIENT_OCCLUSION
            attributes[vertex_offset // 3 + 2] = quad.light if quad.light else NO_LIGHT

            # Write indices (6 per quad, triangle fan 0-1-2, 0-2-3)
            indices[index_offset] = vertex_offset - 12
            indices[index_offset + 1] = vertex_offset - 9
            indices[index_offset + 2] = vertex_offset - 6
            indices[index_offset + 3] = vertex_offset - 12
            indices[index_offset + 4] = vertex_offset - 6
            indices[index_offset + 5] = vertex_offset - 3
            index_offset += 6

        # Record group
        mesh_groups.append(MeshBufferGroup(
            layer=layer.name,
            vertexOffset=layer_start_vertex,
            vertexCount=(vertex_offset - layer_start_vertex) // 3,
            indexOffset=layer_start_index,
            indexCount=index_offset - layer_start_index
        ))

    # 6. Create Mesh Section
    # CreateMeshSection(vertices, triangles, normals, uvs, colors, tangents, bUpdateBounds)
    CreateMeshSection(
        vertices=positions,
        triangles=indices,
        normals=normals,
        uvs=[], # Empty UVs as per source
        colors=[], # Empty colors
        tangents=[], # Empty tangents
        bUpdateBounds=False
    )

    # 7. Configure Unity Mesh
    mesh = GetComponent(Mesh)
    mesh.Clear()
    
    # Set buffer params
    mesh.SetVertexBufferParams(vertex_count, MeshingVertexData.VertexBufferMemoryLayout)
    mesh.SetIndexBufferParams(vertex_count, IndexFormat.UInt16) # Note: Source says UInt16 for index buffer params, but indices array is Uint32. Using UInt16 as per source [S1, unofficial].
    
    # Set submesh
    submesh = SubMeshDescriptor(indexCount=vertex_count) # Source says indexCount=vertexCount, likely a typo for indexCount=index_offset or similar, but following source [S1, unofficial]
    mesh.SetSubMesh(0, submesh)
    mesh.RecalculateBounds()

    # 8. Assign to Chunk
    out_chunk = OutChunk()
    out_chunk.MeshComp = GetComponent(ProceduralMeshComponent)
    out_chunk.CollisionSectionIndex = section_index + 1
    out_chunk.PendingCollision = None
    out_chunk.bHasCollision = False
    out_chunk.bCastsShadow = False

    # Assign to GameObject
    GetComponent(MeshFilter).mesh = mesh
    GetComponent(MeshCollider).mesh = mesh
    GetComponent(MeshRenderer).enabled = True
    GetComponent(MeshCollider).enabled = True

    chunk_properties.HasChanges = False
    chunk_properties.IsMeshGenerated = True
```

## Subtopics

### [Section-based Mesh Generation](4220-section-based-mesh-generation.md)
Renders a large voxel chunk as a vertical stack of smaller, independent sections to enable efficient partial rebuilds when voxels change. It differs from full-chunk rebuilds by limiting computation to the minimal set of sections intersecting the modified voxel, and from adjacency-based chunk rebuilding by operating at the sub-chunk section level within a single chunk.

### [Chunk Region Meshing with Dirty Regions](4402-chunk-region-meshing-with-dirty-regions.md)
Incremental surface extraction that generates geometry for a voxel chunk by processing only a specified subset of "dirty" cells, while consulting the complete chunk and its neighbors for visibility, ambient occlusion (AO), and fluid data. It solves the problem of expensive full-chunk remeshing by allowing partial updates, ensuring that only changed geometry is regenerated. Unlike Section-based Mesh Generation, which processes the entire chunk layer by layer, this method restricts computation to a local `dirtyRegion`, significantly reducing work for sparse updates.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of exposed quads across all layers. The assembly step itself is $O(Q)$ where $Q$ is the total quad count, involving buffer writes and offset calculations.
*   **Memory Complexity:** $O(Q)$ for contiguous buffers. Positions and normals take $12 \times Q$ bytes each (Float32/Int8), indices take $24 \times Q$ bytes (Uint32), and attributes take $4 \times Q$ bytes.
*   **Performance:** The contiguous buffer layout minimizes GPU draw call overhead and state changes. However, the method requires a full rebuild if `HasChanges` is true, which can be expensive for large chunks with frequent updates.
*   **Strengths:** Simple data structure, efficient GPU submission, clear separation of layers.
*   **Limitations:** Not incremental by default (relies on `HasChanges` flag). Empty sections must be explicitly skipped to avoid invalid mesh creation. Missing block types require fallback handling.
*   **When to use:** Static or semi-static chunks where full rebuilds are acceptable, or when combined with dirty region/section-based strategies for updates.
*   **When not to use:** Highly dynamic scenes requiring per-voxel updates without chunk rebuilds, unless combined with [Chunk Region Meshing with Dirty Regions](4402-chunk-region-meshing-with-dirty-regions.md).

## Background
Chunk Mesh Assembly is a method used in voxel terrain rendering systems, particularly within Unity-based engines, to aggregate mesh data from multiple voxel layers into a single renderable mesh. It is part of the broader "Mesh Assembly and Data Structure" family, which focuses on efficient buffer management and GPU submission. The method is implemented in projects such as Eldemarkki's Marching-Cubes-Terrain [S1, unofficial], nerima-games' mc-meshing [S2, unofficial], and HawkDev15's VoxelWorld [S3, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S3] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
