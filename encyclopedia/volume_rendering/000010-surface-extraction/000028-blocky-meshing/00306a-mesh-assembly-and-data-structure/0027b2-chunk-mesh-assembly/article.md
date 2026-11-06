# Chunk Mesh Assembly

Also called: Chunk Mesh Generation, Mesh Assembly, Mesh Buffer Packing, PackedMeshBuffers, Procedural Voxel Mesh Generation, Section-based Mesh Building, Unity Mesh Configuration

## Overview
Chunk Mesh Assembly aggregates mesh data generated from multiple voxel layers (e.g., opaque, water, transparentSolid, fluids) into contiguous typed arrays for efficient GPU submission [S2, unofficial]. It solves the problem of managing fragmented geometry data from parallel or multi-pass meshing by packing it into a single `Mesh` structure [S2, unofficial]. Unlike [Mesh Plugin Architecture](4921-mesh-plugin-architecture.md), which defines an interface for mesh generation, or [Block Render Data Structure & Instance Generation](6209-block-render-data-structure-instance-generation.md), which focuses on instance culling, Chunk Mesh Assembly handles the final consolidation of vertex/index buffers and material assignment [S1, unofficial][S2, unofficial][S3, unofficial].

## How it works
**Inputs:**
- `chunkProperties`: Contains `HasChanges` flag, `ChunkCoordinate`, and references to `MeshFilter`, `MeshCollider`, `MeshRenderer` [S1, unofficial].
- `VoxelWorld.VoxelMesher`: Interface providing `CreateMesh` which returns prepared mesh sections [S1, unofficial][S3, unofficial].
- `BlockMaterials`: Map of block types to materials [S2, unofficial][S3, unofficial].
- `MESH_BUFFER_LAYERS`: List of layer types to process (e.g., opaque, water, transparentSolid) [S2, unofficial].

**Outputs:**
- A populated Unity `Mesh` object assigned to the chunk's `MeshFilter` and `MeshCollider` [S1, unofficial].
- `OutChunk`: Metadata including `MeshComp`, `CollisionSectionIndex`, and shadow/collision flags [S3, unofficial].

**Data Structures:**
- `PackedMeshBuffers`: Contains `positions` (Float32Array), `normals` (Int8Array), `blockIds` (Uint16Array), `ambientOcclusion` (Uint8Array), `blockLight` (Uint8Array), `skyLight` (Uint8Array), `indices` (Uint32Array), and `groups` (`MeshBufferGroup[]`) [S2, unofficial].
- `MeshBufferGroup`: `{ layer: MeshBufferLayer, vertexOffset: number, vertexCount: number, indexOffset: number, indexCount: number }` [S2, unofficial].
- `FPreparedChunkMesh` (C++): Holds `ChunkIndex`, `SizeX`, `SizeY`, `Sections` (map of `EBlockType` to `FChunkMeshSection`), and `CollisionSection` [S3, unofficial].
- `FChunkMeshSection`: Contains `Vertices` (`TArray<FVector>`), `Triangles` (`TArray<int32>`), `Normals` (`TArray<FVector>`), `UVs` (`TArray<FVector2D>`) [S3, unofficial].
- `FActiveChunk` (C++): Tracks `MeshComp`, `PendingCollision`, `CollisionSectionIndex`, `bHasCollision`, `bCastsShadow` [S3, unofficial].

**Parameters:**
- `BlockSize`: World units per voxel edge. Default: 100.f [S3, unofficial].
- `NO_AMBIENT_OCCLUSION`: Constant 0 [S2, unofficial].
- `NO_LIGHT`: Constant 0 [S2, unofficial].
- `FallbackMat`: First non-null material in `BlockMaterials` [S2, unofficial][S3, unofficial].

**Steps:**
1. Check `chunkProperties.HasChanges`. If false, skip [S1, unofficial].
2. Call `VoxelWorld.VoxelMesher.CreateMesh` with voxel data, color data, and chunk coordinate. If result is null, return [S1, unofficial].
3. Calculate total quad count across all `MESH_BUFFER_LAYERS` [S2, unofficial].
4. Clear existing mesh sections on the component [S1, unofficial].
5. Allocate output arrays:
   - Positions/Normals: `quadCount * 4 * 3` elements [S2, unofficial].
   - Indices: `quadCount * 6` elements [S2, unofficial].
   - Other attributes (blockIds, AO, etc.): `quadCount * 4` elements [S2, unofficial].
6. Iterate over `Prepared.Sections`. Skip sections where `Section.Vertices.Num() == 0` to avoid empty mesh sections [S3, unofficial].
7. Calculate `vertexCount` as `job.VertexCountCounter.Count * 3`. Dispose the counter [S1, unofficial].
8. Initialize writer with offsets (`indexOffset: 0`, `vertexOffset: 0`) and output arrays [S1, unofficial].
9. Configure Unity Mesh:
   - Set vertex buffer params with `vertexCount` and `MeshingVertexData.VertexBufferMemoryLayout` [S1, unofficial].
   - Set index buffer params with `vertexCount` and `IndexFormat.UInt16` [S1, unofficial].
10. For each layer in `MESH_BUFFER_LAYERS`:
    - Record current offsets [S2, unofficial].
    - Append quads for that layer [S2, unofficial].
    - Record new offsets in `MeshBufferGroup` [S2, unofficial].
    - Look up material for block type. If not found, use `FallbackMat` [S3, unofficial].
11. Set vertex and index buffer data from job outputs (`job.OutputVertices`, `job.OutputTriangles`). Dispose job output arrays [S1, unofficial].
12. For opaque/water/transparentSolid layers:
    - Iterate quads [S2, unofficial].
    - Compute vertices from quad bounds [S2, unofficial].
    - Write vertex data (position, normal, blockId, ao, lights) [S2, unofficial].
    - Write 6 indices per quad (triangle fan: 0-1-2, 0-2-3) [S2, unofficial].
    - Update offsets [S2, unofficial].
13. Set submesh count to 1. Create `SubMeshDescriptor` with index count `vertexCount` and set as submesh 0. Recalculate bounds [S1, unofficial].
14. Create mesh section using `CreateMeshSection` with vertices, triangles, normals, UVs, empty color/tangent arrays, and `bUpdateBounds=false` [S1, unofficial].
15. Assign component and metadata to `OutChunk`: `MeshComp`, `CollisionSectionIndex` (points to next available section), `PendingCollision`, `bHasCollision=false`, `bCastsShadow=false` [S3, unofficial].
16. Assign mesh to chunk's `MeshFilter` and `MeshCollider`. Enable collider and renderer [S1, unofficial].
17. Reset `HasChanges` to false and set `IsMeshGenerated` to true [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

NO_AMBIENT_OCCLUSION = 0
NO_LIGHT = 0

class SubMeshDescriptor:
    def __init__(self, indexStart, indexCount, vertexCount):
        self.indexStart = indexStart
        self.indexCount = indexCount
        self.vertexCount = vertexCount

def compute_quad_vertices(quad, block_size):
    # Standard operation: compute 4 corner positions from quad bounds and block size
    return (quad.min_x, quad.min_y, quad.min_z), (quad.max_x, quad.min_y, quad.min_z), \
           (quad.max_x, quad.max_y, quad.min_z), (quad.min_x, quad.max_y, quad.min_z)

def write_floats(arr, offset, v):
    arr[offset:offset+12] = struct.pack('<fff', *v)

def write_bytes(arr, offset, v):
    arr[offset:offset+3] = struct.pack('bbb', *v)

def write_uint16(arr, offset, val):
    arr[offset:offset+2] = struct.pack('<H', val)

def write_uint8(arr, offset, val):
    arr[offset] = val

def assemble_chunk_mesh(chunk_properties, voxel_mesher, block_materials, mesh_buffer_layers):
    if not chunk_properties.has_changes:
        return

    prepared_mesh = voxel_mesher.create_mesh(
        voxel_data=chunk_properties.voxel_data,
        color_data=chunk_properties.color_data,
        chunk_coordinate=chunk_properties.chunk_coordinate
    )
    
    if prepared_mesh is None:
        return

    # Calculate total quads across all layers
    total_quad_count = sum(prepared_mesh.layer_quad_count(layer) for layer in mesh_buffer_layers)

    # Clear existing mesh sections
    chunk_properties.mesh_filter.mesh.clear()

    # Allocate arrays
    # Positions: 3 floats per vertex, 4 vertices per quad
    positions = bytearray(total_quad_count * 4 * 3 * 4)
    # Normals: 3 signed bytes per vertex
    normals = bytearray(total_quad_count * 4 * 3)
    # Indices: 6 indices per quad (2 triangles)
    indices = bytearray(total_quad_count * 6 * 4)
    # Other attributes: 4 elements per vertex (blockId, AO, light, etc.)
    block_ids = bytearray(total_quad_count * 4 * 2)
    ao_data = bytearray(total_quad_count * 4)
    light_data = bytearray(total_quad_count * 4)

    # Find fallback material
    fallback_mat = None
    for mat in block_materials.values():
        if mat is not None:
            fallback_mat = mat
            break

    vertex_offset = 0
    index_offset = 0
    mesh_buffer_groups = []

    # Iterate over prepared sections
    for section in prepared_mesh.sections:
        if len(section.vertices) == 0:
            continue

        # Determine material for this section
        block_type = section.block_type
        material = block_materials.get(block_type, fallback_mat)
        if material is None:
            material = fallback_mat

        # Record offsets for this layer/section
        start_vertex_offset = vertex_offset
        start_index_offset = index_offset

        # Write vertices and indices for this section
        for quad in section.quads:
            # Compute vertex positions from quad bounds
            v0, v1, v2, v3 = compute_quad_vertices(quad, block_size=100.0)
            
            # Write positions (3 floats each)
            write_floats(positions, vertex_offset, v0)
            write_floats(positions, vertex_offset + 12, v1)
            write_floats(positions, vertex_offset + 24, v2)
            write_floats(positions, vertex_offset + 36, v3)
            vertex_offset += 48

            # Write normals (3 signed bytes each)
            write_bytes(normals, vertex_offset - 48, quad.normal)
            write_bytes(normals, vertex_offset - 36, quad.normal)
            write_bytes(normals, vertex_offset - 24, quad.normal)
            write_bytes(normals, vertex_offset - 12, quad.normal)

            # Write attributes (blockId, AO, Light, etc.)
            write_uint16(block_ids, vertex_offset - 48, block_type)
            write_uint8(ao_data, vertex_offset - 48, NO_AMBIENT_OCCLUSION)
            write_uint8(light_data, vertex_offset - 48, NO_LIGHT)

            # Write indices (triangle fan: 0-1-2, 0-2-3)
            base_idx = index_offset // 4
            indices[index_offset] = base_idx
            indices[index_offset + 1] = base_idx + 1
            indices[index_offset + 2] = base_idx + 2
            indices[index_offset + 3] = base_idx
            indices[index_offset + 4] = base_idx + 2
            indices[index_offset + 5] = base_idx + 3
            index_offset += 24

        # Store group info
        mesh_buffer_groups.append({
            "layer": section.layer,
            "vertexOffset": start_vertex_offset,
            "vertexCount": vertex_offset - start_vertex_offset,
            "indexOffset": start_index_offset,
            "indexCount": index_offset - start_index_offset
        })

    # Set mesh data
    mesh = chunk_properties.mesh_filter.mesh
    mesh.vertices = positions
    mesh.normals = normals
    mesh.triangles = indices

    # Configure submeshes
    mesh.submeshCount = 1
    mesh.SetSubMesh(0, SubMeshDescriptor(indexStart=0, indexCount=index_offset // 4, vertexCount=vertex_offset // 12))

    # Recalculate bounds
    mesh.RecalculateBounds()

    # Assign to collider
    if chunk_properties.mesh_collider is not None:
        chunk_properties.mesh_collider.sharedMesh = mesh
        chunk_properties.mesh_collider.enabled = True

    # Update renderer
    if chunk_properties.mesh_renderer is not None:
        chunk_properties.mesh_renderer.enabled = True
        chunk_properties.mesh_renderer.material = fallback_mat

    # Reset state
    chunk_properties.has_changes = False
    chunk_properties.is_mesh_generated = True

    return {
        "mesh_comp": chunk_properties.mesh_filter,
        "collision_section_index": len(mesh_buffer_groups),
        "pending_collision": None,
        "b_has_collision": False,
        "b_casts_shadow": False
    }
```

## Subtopics

### [Section-based Mesh Generation](4220-section-based-mesh-generation.md)
Section-based mesh generation is a method for constructing surface meshes from large voxel chunks by dividing the volume into smaller, manageable vertical slabs (sections). It solves the performance bottleneck of regenerating an entire large chunk mesh when only a small portion of voxels changes. By isolating geometry generation to specific vertical ranges, the method allows for efficient partial rebuilds, where only the affected sections and their immediate neighbors are regenerated. It differs from full-chunk regeneration by limiting computational scope to a fraction of the total volume, and from region-based dirty-flagging by operating on fixed vertical slices rather than arbitrary 3D bounding boxes.

### [Chunk Region Meshing with Dirty Regions](4402-chunk-region-meshing-with-dirty-regions.md)
Chunk Region Meshing with Dirty Regions generates geometry for a specific chunk by processing only "dirty" cells within a defined region, while consulting the complete chunk and its neighbors for visibility, ambient occlusion, and fluid data. It differs from full-chunk meshing by returning an `ownedRegion` that includes a 1-cell halo around the dirty region for face culling, AO, and fluid corner reads, clamped at chunk boundaries. This allows incremental updates without regenerating the entire chunk mesh, maintaining compatibility with the standard `meshChunk` API by returning empty buffers for empty regions.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of exposed quads. The assembly step itself is $O(Q)$ where $Q$ is the total quad count, involving array allocations and writes [S2, unofficial].
- **Memory Complexity:** $O(Q)$ for vertex and index buffers, where $Q$ is the total quad count [S2, unofficial].
- **Performance:** The assembly step is deterministic and byte-identical across runs for the same input [S2, unofficial]. It is optimized to avoid polymorphic calls in the inner loop [S2, unofficial].
- **Strengths:** Efficient packing of multi-layer data; handles material fallbacks; prevents empty mesh sections [S2, unofficial][S3, unofficial].
- **Limitations:** Requires full chunk rebuild if `HasChanges` is true; does not handle partial updates within a chunk (handled by [Section-based Mesh Generation](4220-section-based-mesh-generation.md)) [S1, unofficial][S2, unofficial].
- **When to use:** When a complete chunk mesh needs to be generated or regenerated after voxel changes [S1, unofficial].
- **When not to use:** For incremental updates within a chunk; use [Section-based Mesh Generation](4220-section-based-mesh-generation.md) or [Chunk Region Meshing with Dirty Regions](4402-chunk-region-meshing-with-dirty-regions.md) instead [S2, unofficial][S3, unofficial].

## Background
Chunk Mesh Assembly is a core component of procedural voxel terrain systems, particularly those using Unity [S1, unofficial] or Unreal Engine [S3, unofficial]. It aggregates mesh data from multiple layers (opaque, water, transparent, fluids) into contiguous arrays for GPU submission [S2, unofficial]. The method is often paired with greedy meshing to reduce quad counts [S2, unofficial]. It handles material assignment, fallback logic, and mesh configuration for rendering and collision [S1, unofficial][S3, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S3] HawkDev15/VoxelWorld, https://github.com/HawkDev15/VoxelWorld, fetched 2026-10-28 (unofficial)
