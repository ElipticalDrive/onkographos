# GPU Voxel Mesh Generation

Also called: GPU voxel meshing, Surface Extraction from Voxels, Voxel Mesh Generation (Marching Cubes-like Surface Extraction), Voxel chunk mesh generation

## Overview
GPU Voxel Mesh Generation is a technique to generate a static triangle mesh from a 3D voxel grid entirely on the GPU, avoiding CPU readback [S1, unofficial]. It identifies exposed faces of active voxels and generates geometry for each face using a multi-pass compute shader pipeline [S1, unofficial]. Unlike CPU-based meshing, which is significantly slower (e.g., 5 FPS versus >1000 FPS for generation alone), this method leverages parallel processing to achieve high performance [S1, unofficial]. It builds on [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md) by implementing the face culling logic within a GPU compute shader environment using indirect drawing [S1, unofficial].

## How it works
The method uses a three-pass compute shader approach followed by indirect procedural drawing [S1, unofficial]. It assumes a single voxel chunk of fixed dimensions [S1, unofficial].

### Inputs and Outputs
- **Input**: A 3D voxel grid where active voxels have a value > 0 and inactive voxels (air) have a value <= 0, stored in a 1D buffer [S1, unofficial].
- **Output**: A triangle mesh defined by vertex and index buffers, drawn using `Graphics.DrawProceduralIndirect` [S1, unofficial].

### Data Structures
The implementation uses the following constants and structures [S1, unofficial]:

**Constants**:
- `CHUNK_SIZE`: 80 (dimension of the voxel chunk).
- `WorkGroupSize`: 8 (threads per group dimension).
- `SubChunkSize`: 10 (`CHUNK_SIZE / WorkGroupSize`).
- `VoxelCount`: 512,000 (`CHUNK_SIZE^3`).

**Structures**:
1.  **ChunkFeedback**:
    ```hlsl
    struct ChunkFeedback {
        uint vertexOffset;
        uint vertexCount;
        uint indexOffset;
        uint indexCount;
    };
    ```
    Stored in a buffer indexed by chunk ID calculated as `groupID.x + SubChunkSize * (groupID.y + SubChunkSize * groupID.z)` [S1, unofficial].

2.  **GeometryData** (Global Feedback):
    ```hlsl
    struct GeometryData {
        uint vertexCount;
        uint indexCount;
    };
    ```
    Stored at index 0 of the global feedback buffer [S1, unofficial].

3.  **Vertex**:
    Contains `position` (float3), `texcoord` (float2), and `normal` (float3) [S1, unofficial].

**Buffers**:
- `_voxelBuffer`: `ComputeBuffer` of size `VoxelCount` (512,000), type `int` [S1, unofficial].
- `_chunkFeedbackBuffer`: `ComputeBuffer` of size 1, type `ChunkFeedback` [S1, unofficial].
- `_subChunkFeedbackBuffer`: `ComputeBuffer` of size `SubChunkSize^3` (1,000), type `SubChunkFeedback` [S1, unofficial].
- `_vertexBuffer`: `ComputeBuffer` of size `VoxelCount * 4 * 6` (1,920,000), type `Vertex` [S1, unofficial].
- `_indexBuffer`: `ComputeBuffer` of size `VoxelCount * 6 * 6` (2,880,000), type `int` [S1, unofficial].
- `_indirectBuffer`: `ComputeBuffer` of size 1, type `uint[5]`, `ComputeBufferType.IndirectArguments` [S1, unofficial].

### Steps

1.  **Voxel Generation (Optional)**: Generate voxel data (0/1) using a 3D noise function in a compute shader [S1, unofficial].
2.  **Feedback Pass**: Iterate all voxels to calculate the total count of vertices and indices required [S1, unofficial].
3.  **Voxelizer Pass**: Iterate all voxels and write vertex and index data into pre-allocated buffers [S1, unofficial].
4.  **Drawing**: Draw the mesh using `Graphics.DrawProceduralIndirect` [S1, unofficial].

#### Step 2: Feedback Pass
The feedback pass determines how many vertices and indices are needed by counting exposed faces [S1, unofficial].

**Pseudocode (Python-like)**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feedback_pass(voxels, chunk_size=80, workgroup_size=8):
    sub_chunk_size = chunk_size // workgroup_size
    global_vertex_count = 0
    global_index_count = 0
    chunk_feedback = []

    def to_1d(x, y, z):
        return x + chunk_size * (y + chunk_size * z)

    def has_voxel(x, y, z):
        if x < 0 or x >= chunk_size or y < 0 or y >= chunk_size or z < 0 or z >= chunk_size:
            return False
        return voxels[to_1d(x, y, z)] > 0

    for g_x in range(sub_chunk_size):
        for g_y in range(sub_chunk_size):
            for g_z in range(sub_chunk_size):
                s_vertex_count = 0
                s_index_count = 0
                for l_x in range(workgroup_size):
                    for l_y in range(workgroup_size):
                        for l_z in range(workgroup_size):
                            global_idx = to_1d(g_x * workgroup_size + l_x,
                                               g_y * workgroup_size + l_y,
                                               g_z * workgroup_size + l_z)
                            if voxels[global_idx] > 0:
                                coord = (global_idx % chunk_size,
                                         (global_idx // chunk_size) % chunk_size,
                                         global_idx // (chunk_size * chunk_size))
                                local_v = 0
                                local_i = 0
                                for dx, dy, dz in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
                                    if not has_voxel(coord[0]+dx, coord[1]+dy, coord[2]+dz):
                                        local_v += 4
                                        local_i += 6
                                s_vertex_count += local_v
                                s_index_count += local_i

                global_vertex_count += s_vertex_count
                global_index_count += s_index_count
                chunk_idx = g_x + sub_chunk_size * (g_y + sub_chunk_size * g_z)
                chunk_feedback.append({
                    'vertex_offset': global_vertex_count - s_vertex_count,
                    'vertex_count': s_vertex_count,
                    'index_offset': global_index_count - s_index_count,
                    'index_count': s_index_count
                })
    return global_vertex_count, global_index_count, chunk_feedback
```

#### Step 3: Voxelizer Pass
The voxelizer pass writes the actual geometry data into the buffers [S1, unofficial].

**Pseudocode (Python-like)**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelizer_pass(voxels, chunk_feedback, chunk_size=80, workgroup_size=8):
    sub_chunk_size = chunk_size // workgroup_size
    vertices = []
    indices = []

    def to_1d(x, y, z):
        return x + chunk_size * (y + chunk_size * z)

    def has_voxel(x, y, z):
        if x < 0 or x >= chunk_size or y < 0 or y >= chunk_size or z < 0 or z >= chunk_size:
            return False
        return voxels[to_1d(x, y, z)] > 0

    for g_x in range(sub_chunk_size):
        for g_y in range(sub_chunk_size):
            for g_z in range(sub_chunk_size):
                chunk_idx = g_x + sub_chunk_size * (g_y + sub_chunk_size * g_z)
                v_offset = chunk_feedback[chunk_idx]['vertex_offset']
                i_offset = chunk_feedback[chunk_idx]['index_offset']

                for l_x in range(workgroup_size):
                    for l_y in range(workgroup_size):
                        for l_z in range(workgroup_size):
                            global_idx = to_1d(g_x * workgroup_size + l_x,
                                               g_y * workgroup_size + l_y,
                                               g_z * workgroup_size + l_z)
                            if voxels[global_idx] > 0:
                                coord = (global_idx % chunk_size,
                                         (global_idx // chunk_size) % chunk_size,
                                         global_idx // (chunk_size * chunk_size))
                                faces = [
                                    ((1,0,0), [(0.5, -0.5, -0.5), (0.5, 0.5, -0.5), (0.5, 0.5, 0.5), (0.5, -0.5, 0.5)]),
                                    ((-1,0,0), [(-0.5, 0.5, -0.5), (-0.5, -0.5, -0.5), (-0.5, -0.5, 0.5), (-0.5, 0.5, 0.5)]),
                                    ((0,1,0), [(-0.5, 0.5, -0.5), (0.5, 0.5, -0.5), (0.5, 0.5, 0.5), (-0.5, 0.5, 0.5)]),
                                    ((0,-1,0), [(-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, -0.5, -0.5), (-0.5, -0.5, -0.5)]),
                                    ((0,0,1), [(-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, 0.5, 0.5), (-0.5, 0.5, 0.5)]),
                                    ((0,0,-1), [(-0.5, 0.5, -0.5), (0.5, 0.5, -0.5), (0.5, -0.5, -0.5), (-0.5, -0.5, -0.5)])
                                ]
                                for offset, verts in faces:
                                    if not has_voxel(coord[0]+offset[0], coord[1]+offset[1], coord[2]+offset[2]):
                                        base_v = v_offset
                                        base_i = i_offset
                                        for v in verts:
                                            vertices.append((coord[0]+v[0], coord[1]+v[1], coord[2]+v[2]))
                                        indices.append(base_v)
                                        indices.append(base_v+1)
                                        indices.append(base_v+2)
                                        indices.append(base_v)
                                        indices.append(base_v+2)
                                        indices.append(base_v+3)
                                        v_offset += 4
                                        i_offset += 6
    return vertices, indices
```

#### Step 4: Drawing
The mesh is drawn using `Graphics.DrawProceduralIndirect` [S1, unofficial]. The indirect buffer is initialized to `{0, 1, 0, 0, 0}` and relies on the feedback shader to update the counts in the buffers bound to the indirect arguments, though the specific binding mechanism is implicit in the source [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```csharp
// C# Side
uint[] indirectArgs = new uint[5];
indirectArgs[0] = totalIndexCount; // Number of indices
indirectArgs[1] = 0;               // Start index location
indirectArgs[2] = 0;               // Start vertex location
indirectArgs[3] = 0;               // Number of instances
indirectArgs[4] = 0;               // Start instance location

_indirectBuffer.SetData(indirectArgs);
Graphics.DrawProceduralIndirect(drawMaterial, bounds, MeshTopology.Triangles, _indirectBuffer);
```

### Parameters
- `CHUNK_SIZE`: 80 [S1, unofficial].
- `WorkGroupSize`: 8 [S1, unofficial].
- `SubChunkSize`: 10 [S1, unofficial].

### Edge Cases and Pitfalls
- **Boundary Checks**: Neighbor lookups must check for out-of-bounds coordinates to avoid memory access violations [S1, unofficial].
- **Indirect Buffer Update**: The indirect buffer must be reset or updated with correct counts before drawing; the code initializes it with `{0, 1, 0, 0, 0}` but relies on the feedback shader to update the counts in the buffers bound to the indirect arguments [S1, unofficial].
- **Precision**: Using `uint` for offsets and counts is sufficient for the given chunk size [S1, unofficial].

## Operating characteristics
- **Performance**: Mesh generation runs at >1000 FPS with camera disabled on an RTX 4090, and total meshing and drawing runs at ~650 FPS [S1, unofficial].
- **Comparison**: GPU-only generation is much faster than reading data back to CPU to create a Mesh object, which runs at 5 FPS [S1, unofficial].
- **Memory**: The method pre-allocates buffers for the maximum possible number of vertices and indices (`VoxelCount * 4 * 6` and `VoxelCount * 6 * 6`) [S1, unofficial].
- **Scalability**: The method is designed for a single chunk of size 80x80x80, and scaling to larger volumes would require chunking and handling inter-chunk faces, which is not covered in the source [S1, unofficial].

## Background
This method is an experiment to generate a cubic voxel chunk mesh as efficiently as possible, leveraging Unity's compute shaders and indirect procedural drawing [S1, unofficial]. It builds on OpenGL voxelizer compute shaders by Demurzasty and GPU Noise by keijiro [S1, unofficial].

## Sources
[S1] artnas/UnityVoxelMeshGPU, https://github.com/artnas/UnityVoxelMeshGPU, fetched 2026-10-27 (unofficial)
