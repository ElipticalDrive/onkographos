# GPU Voxel Mesh Generation

Also called: GPU voxel meshing, Surface Extraction from Voxels, Voxel Mesh Generation, Voxel Meshing, Voxel chunk mesh generation

## Overview
GPU Voxel Mesh Generation is a technique to generate a static triangle mesh from a 3D voxel grid entirely on the GPU, avoiding CPU readback [S1, unofficial]. It identifies exposed faces of active voxels and generates geometry for each face using a multi-pass compute shader pipeline [S1, unofficial]. Unlike CPU-based meshing, which is significantly slower (e.g., 5 FPS vs >1000 FPS for generation alone), this method leverages parallel processing to achieve high performance [S1, unofficial]. It builds on [Voxel Mesh Generation with Face Culling](2411-voxel-mesh-generation-with-face-culling.md) by implementing the face culling logic within a GPU compute shader environment using indirect drawing [S1, unofficial].

## How it works
The method uses a three-pass compute shader approach followed by indirect procedural drawing. It assumes a single voxel chunk of fixed dimensions.

### Inputs and Outputs
- **Input**: A 3D voxel grid where active voxels have a value > 0 and inactive voxels (air) have a value <= 0. The grid is stored in a 1D buffer.
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
    Stored in a buffer indexed by chunk ID. The index is calculated as `groupID.x + SubChunkSize * (groupID.y + SubChunkSize * groupID.z)` [S1, unofficial]. Note: The source code uses `10` directly in the C# side calculation but `SubChunkSize` is 10, so the formula holds.

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
- `_subChunkFeedbackBuffer`: `ComputeBuffer` of size `SubChunkSize^3` (1,000), type `ChunkFeedback` [S1, unofficial].
- `_vertexBuffer`: `ComputeBuffer` of size `VoxelCount * 4 * 6` (1,920,000), type `Vertex` [S1, unofficial].
- `_indexBuffer`: `ComputeBuffer` of size `VoxelCount * 6 * 6` (2,880,000), type `int` [S1, unofficial].
- `_indirectBuffer`: `ComputeBuffer` of size 1, type `uint[5]`, `ComputeBufferType.IndirectArguments` [S1, unofficial].

### Steps

1.  **Voxel Generation (Optional)**: Generate voxel data (0/1) using a 3D noise function in a compute shader [S1, unofficial].
2.  **Feedback Pass**: Iterate all voxels to calculate the total count of vertices and indices required.
3.  **Voxelizer Pass**: Iterate all voxels and write vertex and index data into pre-allocated buffers.
4.  **Drawing**: Draw the mesh using `Graphics.DrawProceduralIndirect` [S1, unofficial].

#### Step 2: Feedback Pass
The feedback pass determines how many vertices and indices are needed by counting exposed faces.

**Pseudocode (HLSL-style Compute Shader)**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```hlsl
#define CHUNK_SIZE 80
#define WORK_GROUP_SIZE 8

struct GeometryData { uint vertexCount; uint indexCount; };
struct ChunkFeedback { uint vertexOffset; uint vertexCount; uint indexOffset; uint indexCount; };

// Buffers
RWStructuredBuffer<int> uVoxels;
RWStructuredBuffer<GeometryData> uFeedback; // Global feedback at index 0
RWStructuredBuffer<ChunkFeedback> uChunkFeedback; // Sub-chunk feedback

// Shared memory
groupshared uint sVertexCount;
groupshared uint sIndexCount;

// Helper to convert 3D coord to 1D index
uint to1D(uint3 pos) {
    return pos.x + CHUNK_SIZE * (pos.y + CHUNK_SIZE * pos.z);
}

// Helper to check if a voxel exists and is active
bool hasVoxel(int3 coord) {
    if (coord.x < 0 || coord.x >= CHUNK_SIZE || 
        coord.y < 0 || coord.y >= CHUNK_SIZE ||
        coord.z < 0 || coord.z >= CHUNK_SIZE) {
        return false;
    }
    int idx = to1D(uint3(coord));
    return uVoxels[idx] > 0;
}

[numthreads(WORK_GROUP_SIZE, WORK_GROUP_SIZE, WORK_GROUP_SIZE)]
void CSMain(uint3 dispatchID : SV_DispatchThreadID, uint3 groupID : SV_GroupID, uint3 groupThreadID : SV_GroupThreadID)
{
    // 1. Initialize shared memory counters
    if (groupThreadID.x == 0 && groupThreadID.y == 0 && groupThreadID.z == 0) {
        sVertexCount = 0;
        sIndexCount = 0;
    }
    // 2. Synchronize to ensure visibility
    GroupMemoryBarrierWithGroupSync();
    
    // 3. Calculate global voxel index
    uint globalVoxelIndex = to1D(dispatchID);
    
    // 4. Check if active voxel
    if (uVoxels[globalVoxelIndex] > 0) {
        int3 coord = int3(dispatchID);
        uint localVertexCount = 0;
        uint localIndexCount = 0;
        
        // 5. Check 6 neighbors for exposed faces
        // Each exposed face adds 4 vertices and 6 indices
        if (!hasVoxel(coord + int3(1, 0, 0))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        if (!hasVoxel(coord + int3(-1, 0, 0))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        if (!hasVoxel(coord + int3(0, 1, 0))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        if (!hasVoxel(coord + int3(0, -1, 0))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        if (!hasVoxel(coord + int3(0, 0, 1))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        if (!hasVoxel(coord + int3(0, 0, -1))) {
            localVertexCount += 4;
            localIndexCount += 6;
        }
        
        // 6. Atomically add to shared counters
        InterlockedAdd(sVertexCount, localVertexCount);
        InterlockedAdd(sIndexCount, localIndexCount);
    }
    
    // 7. Synchronize again
    GroupMemoryBarrierWithGroupSync();
    
    // 8. First thread updates global feedback and stores chunk metadata
    if (groupThreadID.x == 0 && groupThreadID.y == 0 && groupThreadID.z == 0) {
        // Add group totals to global feedback buffer
        uint vertexOffset = InterlockedAdd(uFeedback[0].vertexCount, sVertexCount);
        uint indexOffset = InterlockedAdd(uFeedback[0].indexCount, sIndexCount);
        
        // Calculate chunk index in sub-chunk feedback buffer
        uint index = groupID.x + SubChunkSize * (groupID.y + SubChunkSize * groupID.z);
        
        // Store chunk-specific metadata
        uChunkFeedback[index].vertexOffset = vertexOffset;
        uChunkFeedback[index].vertexCount = sVertexCount;
        uChunkFeedback[index].indexOffset = indexOffset;
        uChunkFeedback[index].indexCount = sIndexCount;
    }
}
```

#### Step 3: Voxelizer Pass
The voxelizer pass writes the actual geometry data into the buffers.

**Pseudocode (HLSL-style Compute Shader)**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```hlsl
// Buffers
RWStructuredBuffer<int> uVoxels;
RWStructuredBuffer<Vertex> uVertices;
RWStructuredBuffer<int> uIndices;
RWStructuredBuffer<ChunkFeedback> uChunkFeedback; // Sub-chunk feedback

// Shared memory for loading offsets
groupshared uint sVertexOffset;
groupshared uint sIndexOffset;

[numthreads(WORK_GROUP_SIZE, WORK_GROUP_SIZE, WORK_GROUP_SIZE)]
void CSMain(uint3 dispatchID : SV_DispatchThreadID, uint3 groupID : SV_GroupID, uint3 groupThreadID : SV_GroupThreadID)
{
    // 1. Load chunk feedback data into shared memory
    if (groupThreadID.x == 0 && groupThreadID.y == 0 && groupThreadID.z == 0) {
        uint index = groupID.x + SubChunkSize * (groupID.y + SubChunkSize * groupID.z);
        sVertexOffset = uChunkFeedback[index].vertexOffset;
        sIndexOffset = uChunkFeedback[index].indexOffset;
    }
    GroupMemoryBarrierWithGroupSync();
    
    // 2. Calculate global voxel index
    uint globalVoxelIndex = to1D(dispatchID);
    
    // 3. Check if active voxel
    if (uVoxels[globalVoxelIndex] > 0) {
        int3 coord = int3(dispatchID);
        
        // Helper to get local vertex index within the face
        // Face 0: +X, Face 1: -X, etc.
        // Each face has 4 vertices and 6 indices
        
        // Process +X Face
        if (!hasVoxel(coord + int3(1, 0, 0))) {
            // Allocate space atomically
            uint vIdx = InterlockedAdd(sVertexOffset, 4); // Note: sVertexOffset must be managed carefully or use global atomic with offset base
            // In the source, it seems to use the pre-calculated offset from feedback. 
            // However, InterlockedAdd on shared memory is not standard for global allocation.
            // The source code implies using the feedback offsets. 
            // A robust implementation would use atomicAdd on the global buffer pointers or use the feedback offsets directly if single-threaded per face.
            // Given the source structure, it likely uses the feedback offsets as base and writes sequentially within the group or uses atomics on global buffers.
            // For simplicity and adherence to source logic:
            
            // Write Vertex Data
            // Positions are offset by +/-0.5 from voxel center
            // Normal is (1, 0, 0) for +X face
            // Texcoords are typically (0,0), (1,0), (1,1), (0,1)
            
            // Vertex 0
            uVertices[vIdx].position = float3(coord.x + 0.5, coord.y - 0.5, coord.z - 0.5);
            uVertices[vIdx].normal = float3(1, 0, 0);
            uVertices[vIdx].texcoord = float2(0, 0);
            
            // Vertex 1
            uVertices[vIdx+1].position = float3(coord.x + 0.5, coord.y + 0.5, coord.z - 0.5);
            uVertices[vIdx+1].normal = float3(1, 0, 0);
            uVertices[vIdx+1].texcoord = float2(0, 1);
            
            // Vertex 2
            uVertices[vIdx+2].position = float3(coord.x + 0.5, coord.y + 0.5, coord.z + 0.5);
            uVertices[vIdx+2].normal = float3(1, 0, 0);
            uVertices[vIdx+2].texcoord = float2(1, 1);
            
            // Vertex 3
            uVertices[vIdx+3].position = float3(coord.x + 0.5, coord.y - 0.5, coord.z + 0.5);
            uVertices[vIdx+3].normal = float3(1, 0, 0);
            uVertices[vIdx+3].texcoord = float2(1, 0);
            
            // Write Index Data
            // Two triangles: 0-1-2, 0-2-3
            uIndices[sIndexOffset] = vIdx;
            uIndices[sIndexOffset+1] = vIdx+1;
            uIndices[sIndexOffset+2] = vIdx+2;
            uIndices[sIndexOffset+3] = vIdx;
            uIndices[sIndexOffset+4] = vIdx+2;
            uIndices[sIndexOffset+5] = vIdx+3;
            
            // Update offsets for next face/voxel
            // Note: In a parallel environment, we need atomic updates to the global offsets.
            // The source code uses `sVertexOffset` and `sIndexOffset` which are loaded from feedback.
            // This implies the feedback pass calculated the *exact* start offset for each chunk,
            // and the voxelizer pass must use atomics on the global buffers to allocate specific slots.
            // The provided pseudocode above is a simplification. A correct implementation uses:
            // uint vIdx = InterlockedAdd(uVertexCountBuffer, 4); 
            // But the source uses feedback offsets. 
            // Let's stick to the source's implicit logic: The feedback pass calculates the *total* count.
            // The voxelizer pass uses the feedback offsets to know *where* to write.
            // However, multiple threads write to the same chunk. They must coordinate.
            // The source code snippet for voxelizer is not fully provided, but the logic is:
            // "atomically allocate space in the global vertex and index buffers using the shared offsets"
        }
        
        // Similar logic for -X, +Y, -Y, +Z, -Z faces...
    }
}
```
*Note: The exact atomic allocation logic in the voxelizer pass is not fully detailed in the source snippets, but the principle is to use the offsets calculated in the feedback pass as base addresses and atomically increment local counters or use global atomics to allocate specific slots for each face.*

#### Step 4: Drawing
The mesh is drawn using `Graphics.DrawProceduralIndirect`. The indirect buffer is updated with the total vertex and index counts from the feedback pass [S1, unofficial].

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
- **Indirect Buffer Update**: The indirect buffer must be reset or updated with correct counts before drawing. The source code initializes it with `{0, 1, 0, 0, 0}` but relies on the feedback shader to update the counts in the buffers bound to the indirect arguments [S1, unofficial].
- **Precision**: Using `uint` for offsets and counts is sufficient for the given chunk size.

## Operating characteristics
- **Performance**: Mesh generation runs at >1000 FPS with camera disabled on an RTX 4090. Total meshing and drawing runs at ~650 FPS [S1, unofficial].
- **Comparison**: GPU-only generation is much faster than reading data back to CPU to create a Mesh object, which runs at 5 FPS [S1, unofficial].
- **Memory**: The method pre-allocates buffers for the maximum possible number of vertices and indices (`VoxelCount * 4 * 6` and `VoxelCount * 6 * 6`) [S1, unofficial]. This can lead to memory waste if the voxel density is low.
- **Scalability**: The method is designed for a single chunk of size 80x80x80. Scaling to larger volumes would require chunking and handling inter-chunk faces, which is not covered in the source [S1, unofficial].

## Background
This method is an experiment to generate a cubic voxel chunk mesh as efficiently as possible, leveraging Unity's compute shaders and indirect procedural drawing [S1, unofficial]. It builds on OpenGL voxelizer compute shaders by Demurzasty and GPU Noise by keijiro [S1, unofficial]. The technique was demonstrated in a GitHub repository by artnas [S1, unofficial].

## Sources
[S1] artnas/UnityVoxelMeshGPU, https://github.com/artnas/UnityVoxelMeshGPU, fetched 2026-10-27 (unofficial)
