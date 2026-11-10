# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job Scheduling, Chunk-based MC Job Scheduling

## Overview
Marching Cubes Job Scheduling is a parallel execution strategy that decomposes a volumetric dataset into spatial chunks and dispatches independent Marching Cubes extraction tasks to a job system (typically a multi-threaded worker pool). It solves the performance bottleneck of sequential voxel processing by enabling concurrent mesh generation across multiple CPU cores or GPU compute units. Unlike [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md), which optimizes the lookup logic itself, this method focuses on the orchestration of work distribution. It differs from [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by often targeting CPU-based job schedulers (e.g., Unity's Job System) or explicit process queues, though the logical decomposition is similar.

## How it works
The method relies on the precomputed data structures from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md), specifically the `edgeTable` and `triangleTable`. The scheduling layer manages the lifecycle of these computations across discrete data chunks.

### Inputs and Outputs
*   **Input**: A 3D voxel volume (scalar field), an isovalue threshold, and a chunking configuration (chunk dimensions).
*   **Output**: A collection of mesh primitives (vertices and triangle indices) corresponding to the extracted isosurface, often returned as a job handle or asynchronous task object.

### Data Structures
The implementation typically employs a job struct to encapsulate all necessary state for a single chunk's extraction. Based on common implementations [S1, unofficial], the job structure includes:

*   `VoxelData`: A reference to the scalar field data for the specific chunk (e.g., `VoxelDataVolume<byte>`).
*   `VoxelColors`: A reference to the color/normal data for the chunk (e.g., `VoxelDataVolume<Color32>`).
*   `Isolevel`: The scalar threshold for surface extraction (float).
*   `VertexCountCounter`: A thread-safe atomic counter (e.g., `NativeCounter`) to track the number of vertices generated, allowing for dynamic buffer sizing or offset calculation.
*   `OutputVertices`: A pre-allocated array (e.g., `NativeArray<MeshingVertexData>`) to store the resulting vertex positions and attributes.
*   `OutputTriangles`: A pre-allocated array (e.g., `NativeArray<ushort>`) to store the resulting triangle indices.

### Algorithm Steps
1.  **Chunk Initialization**: The scheduler iterates over the 3D grid of chunks. For each chunk coordinate $(c_x, c_y, c_z)$:
    *   Retrieve or generate the voxel data chunk. If the chunk is invalid or empty, skip scheduling [S1, unofficial].
    *   Calculate the maximum possible output size. If a chunk has dimensions $W \times H \times D$, the maximum number of voxels is $V = W \cdot H \cdot D$. Since each voxel can produce at most 5 triangles (15 indices), the maximum buffer size is $15 \cdot V$ for both vertices and indices [S1, unofficial].
2.  **Job Construction**:
    *   Allocate temporary or persistent memory buffers for `OutputVertices` and `OutputTriangles` with the calculated maximum size [S1, unofficial].
    *   Initialize a `MarchingCubesJob` instance, populating it with the chunk's voxel data, color data, the global `Isolevel`, the shared `VertexCountCounter`, and the output buffers.
3.  **Scheduling**:
    *   Dispatch the job to the job scheduler (e.g., via `Schedule()`). This returns a `JobHandle` [S1, unofficial].
    *   The scheduler manages the execution of these jobs across available worker threads.
4.  **Execution (Worker Side)**:
    *   The worker thread processes the voxels in the assigned chunk.
    *   For each voxel, it calculates the case index $C$ based on the 8 corner signs relative to `Isolevel` [2518-marching-cubes-lookup-and-configuration.md].
    *   It uses `edgeTable` and `triangleTable` to determine intersections and triangle connectivity.
    *   It writes vertices and indices to the `OutputVertices` and `OutputTriangles` arrays.
    *   It atomically increments `VertexCountCounter` for each generated vertex/triangle to manage memory offsets if buffers are shared or if post-processing requires exact counts.
5.  **Completion**:
    *   The scheduler waits for the `JobHandle` to complete.
    *   The resulting mesh data is assembled from the output buffers.

### Pseudocode
The following pseudocode illustrates the scheduling logic on the main thread and the execution logic on the worker thread.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Main Thread: Scheduling Logic
function ScheduleChunkMeshing(chunkCoordinate, voxelWorld):
    # 1. Retrieve chunk data
    voxelData = voxelWorld.GetVoxelDataChunk(chunkCoordinate)
    voxelColors = voxelWorld.GetVoxelColorChunk(chunkCoordinate)
    
    if voxelData is null or voxelColors is null:
        return null

    # 2. Calculate buffer sizes
    chunkDims = voxelWorld.ChunkSize # e.g., (16, 16, 16)
    voxelCount = chunkDims.x * chunkDims.y * chunkDims.z
    maxLength = 15 * voxelCount # Max triangles per voxel is 5 -> 15 indices

    # 3. Allocate output buffers
    # Using a thread-safe counter for global vertex tracking
    vertexCountCounter = new NativeCounter(Allocator.TempJob)
    outputVertices = new NativeArray<MeshingVertexData>(maxLength, Allocator.TempJob)
    outputTriangles = new NativeArray<ushort>(maxLength, Allocator.TempJob)

    # 4. Initialize Job Struct
    job = MarchingCubesJob(
        VoxelData=voxelData,
        VoxelColors=voxelColors,
        Isolevel=0.5f, # Default isovalue [S1, unofficial]
        VertexCountCounter=vertexCountCounter,
        OutputVertices=outputVertices,
        OutputTriangles=outputTriangles
    )

    # 5. Schedule Job
    jobHandle = job.Schedule()
    
    return JobHandleWithData(jobHandle, job)

# Worker Thread: Execution Logic (Inside MarchingCubesJob.Execute)
function Execute(job):
    voxelData = job.VoxelData
    voxelColors = job.VoxelColors
    isolevel = job.Isolevel
    outputVerts = job.OutputVertices
    outputTris = job.OutputTriangles
    
    # Precomputed LUTs from parent article
    edgeTable = GetEdgeTable()
    triangleTable = GetTriangleTable()

    vertexOffset = 0
    triangleOffset = 0

    # Iterate over all voxels in the chunk
    for z in 0 to chunkDims.z - 1:
        for y in 0 to chunkDims.y - 1:
            for x in 0 to chunkDims.x - 1:
                # 1. Get 8 corner values
                cornerValues = GetCornerValues(voxelData, x, y, z)
                
                # 2. Calculate Case Index C
                caseIndex = 0
                for i in 0 to 7:
                    if cornerValues[i] < isolevel: # Convention: < isolevel sets bit
                        caseIndex |= (1 << i)
                
                # 3. Lookup Triangles
                triIndices = triangleTable[caseIndex]
                
                # 4. Generate Triangles
                for k in 0 to 15:
                    edgeIdx = triIndices[k]
                    if edgeIdx == -1:
                        break
                    
                    # Get next two edges to form a triangle
                    # Note: triangleTable stores triplets. 
                    # We need to handle the triplet grouping explicitly.
                    # The table stores [e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, -1, -1, -1, -1]
                    # We iterate by 3.
                    pass 
                
                # Simplified triangle generation loop based on triplet structure
                triIdx = 0
                while triIdx < 15:
                    e1 = triIndices[triIdx]
                    e2 = triIndices[triIdx+1]
                    e3 = triIndices[triIdx+2]
                    
                    if e1 == -1:
                        break
                    
                    # Interpolate vertices on edges e1, e2, e3
                    v1 = InterpolateVertex(voxelData, e1, cornerValues, isolevel)
                    v2 = InterpolateVertex(voxelData, e2, cornerValues, isolevel)
                    v3 = InterpolateVertex(voxelData, e3, cornerValues, isolevel)
                    
                    # Write to output buffers
                    outputVerts[vertexOffset] = v1
                    outputVerts[vertexOffset+1] = v2
                    outputVerts[vertexOffset+2] = v3
                    
                    outputTris[triangleOffset] = vertexOffset
                    outputTris[triangleOffset+1] = vertexOffset+1
                    outputTris[triangleOffset+2] = vertexOffset+2
                    
                    vertexOffset += 3
                    triangleOffset += 3
                    
                    # Atomic increment for global tracking if needed
                    job.VertexCountCounter.Increment()
                    
                    triIdx += 3

    # Return results or signal completion
```

### Parameters
*   **Isolevel**: The density threshold for surface extraction. Default value is 0.5f [S1, unofficial].
*   **Chunk Size**: Determines the granularity of parallelism. Smaller chunks increase overhead but improve load balancing; larger chunks reduce overhead but risk load imbalance. Not given in the sources.

### Edge Cases and Pitfalls
*   **Boundary Conditions**: When chunks are adjacent, vertices on the shared faces must be consistent. The standard Marching Cubes algorithm handles this naturally if the voxel data is continuous and the interpolation is deterministic [2518-marching-cubes-lookup-and-configuration.md]. However, if chunks are processed independently without sharing boundary vertex data, cracks may appear if the isovalue is not handled consistently at the boundaries.
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points where the topology is ambiguous. The standard LUT uses a fixed triangulation, which can cause cracks if adjacent voxels resolve the ambiguity differently. This is a property of the underlying MC algorithm, not the scheduling, but parallel processing does not exacerbate it [2518-marching-cubes-lookup-and-configuration.md].
*   **Memory Allocation**: Allocating large buffers for every chunk can lead to high memory pressure. Using `Allocator.TempJob` (as in [S1, unofficial]) is efficient for short-lived jobs but requires careful management of lifetime.

## Subtopics

### Multi-Process Ground Truth Generation
A variant where the scheduling is done across multiple operating system processes rather than threads. It orchestrates parallel processing of 3D models to generate ground truth data using a queue for synchronization [S2, unofficial]. Worker processes are assigned subsets of input files (e.g., OBJ names) based on an even distribution strategy. A central queue manages the exchange of results, with workers polling for tasks and reporting completion. This approach is useful for distributed ground truth generation where memory isolation between processes is preferred [S2, unofficial].

### Tandem Algorithm with Dumping
A strategy that enhances the tandem algorithm by reducing the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration [S3]. It drastically reduces the number of triangles generated compared to the classical brute-force approach. On a 1626x7028x2000 dataset (40 GB), the parallel tandem approach extracted surfaces in 5 minutes using 56 processors, compared to more than 3 hours for the sequential tandem algorithm [S3]. The initial extraction produced 18,111 disconnected components, which were filtered down to 328 components based on geometric criteria (volume) [S3]. This method is applied to seismic cubes for hydrocarbon exploration [S3].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, assuming perfect load balancing and infinite parallelism. In practice, it scales with the number of available cores $P$, approaching $O(N/P)$.
*   **Memory Complexity**: $O(N)$ for the voxel data plus $O(N)$ for the output buffers (worst-case triangle count). The scheduling overhead is $O(C)$ where $C$ is the number of chunks.
*   **Measured Performance**: On a 1626x7028x2000 dataset (40 GB), a parallel tandem approach using 56 processors achieved extraction in 5 minutes, a significant improvement over the 3+ hours required for the sequential version [S3].
*   **Strengths**: High throughput on multi-core systems; scalable to large datasets; decouples data loading from mesh generation.
*   **Limitations**: Overhead of job scheduling and synchronization; potential for load imbalance if the isosurface is sparse or clustered; memory usage can be high if large buffers are allocated per chunk.
*   **When to Use**: Large volumetric datasets where sequential processing is too slow; applications requiring real-time or near-real-time feedback with chunked updates.
*   **When Not to Use**: Small datasets where overhead dominates; memory-constrained environments where buffering all chunks is infeasible.

## Background
The concept of scheduling Marching Cubes jobs emerged from the need to process large volumetric datasets efficiently. Early implementations were sequential, limiting their use to small volumes. The introduction of parallel processing techniques, such as the tandem algorithm and its variants, allowed for the exploitation of multi-core architectures. The specific implementation details described here, such as the use of `NativeCounter` and `NativeArray` in a job system, are characteristic of modern game engine frameworks (e.g., Unity) that provide data-oriented job systems for CPU parallelism [S1, unofficial]. The tandem algorithm with dumping was developed for geophysical applications, specifically for seismic data analysis, where large datasets and interactive exploration are critical [S3].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
