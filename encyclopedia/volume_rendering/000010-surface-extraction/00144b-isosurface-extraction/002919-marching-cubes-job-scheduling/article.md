# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job, Chunk-based Marching Cubes Scheduling

## Overview
Marching Cubes Job Scheduling is a parallel execution strategy for isosurface extraction that decomposes a volumetric dataset into spatial chunks and processes them concurrently using a job system [S1, unofficial]. It solves the performance bottleneck of sequential voxel traversal by distributing the computational load of vertex and triangle generation across multiple worker threads or processes [S1, unofficial][S3]. Unlike single-threaded implementations, this method manages memory allocation and synchronization explicitly to handle the variable output size of the Marching Cubes algorithm per chunk [S1, unofficial].

## How it works
The method operates on a discretized domain, typically a regular grid, where the volume is partitioned into fixed-size chunks. Each chunk is processed independently to extract the local isosurface, which is then merged or rendered as part of the global mesh [S1, unofficial].

### Inputs and Outputs
*   **Input**: A voxel data structure containing scalar values (e.g., `VoxelDataVolume<byte>`) and optional auxiliary data (e.g., `VoxelDataVolume<Color32>`) [S1, unofficial].
*   **Input**: An isovalue threshold (`Isolevel`) used to determine surface presence [S1, unofficial].
*   **Output**: A list of vertices (`NativeArray<MeshingVertexData>`) and a list of triangle indices (`NativeArray<ushort>`) [S1, unofficial].
*   **Output**: A counter tracking the total number of generated vertices (`NativeCounter`) [S1, unofficial].

### Data Structures
The core scheduling unit is a job struct that encapsulates the necessary data and output buffers.

**MarchingCubesJob Struct Layout:**
*   `VoxelData`: Reference to the scalar voxel volume (`VoxelDataVolume<byte>`) [S1, unofficial].
*   `VoxelColors`: Reference to the color voxel volume (`VoxelDataVolume<Color32>`) [S1, unofficial].
*   `Isolevel`: The threshold float for surface extraction [S1, unofficial].
*   `VertexCountCounter`: A thread-safe atomic counter (`NativeCounter`) to aggregate vertex counts from parallel jobs [S1, unofficial].
*   `OutputVertices`: A pre-allocated array (`NativeArray<MeshingVertexData>`) to store generated vertices [S1, unofficial].
*   `OutputTriangles`: A pre-allocated array (`NativeArray<ushort>`) to store triangle indices [S1, unofficial].

### Algorithm Steps
1.  **Chunk Initialization**: For a given chunk coordinate, retrieve the corresponding voxel data and color chunks. If either is unavailable, return null or skip [S1, unofficial].
2.  **Memory Allocation**:
    *   Calculate `voxelCount` as the product of the chunk dimensions ($Width \times Height \times Depth$) [S1, unofficial].
    *   Calculate `maxLength` as $15 \times voxelCount$. This upper bound accounts for the maximum number of triangles/vertices a single cell can generate in Marching Cubes [S1, unofficial].
    *   Allocate `outputVertices` and `outputTriangles` arrays of size `maxLength` using temporary job-allocator memory [S1, unofficial].
    *   Allocate a `vertexCountCounter` with temporary job allocator [S1, unofficial].
3.  **Job Configuration**: Initialize the `MarchingCubesJob` struct with the retrieved voxel data, colors, isovalue, counter, and output arrays [S1, unofficial].
4.  **Scheduling**: Schedule the `MarchingCubesJob` for execution on the job system [S1, unofficial].
5.  **Completion Handling**: Wrap the job handle and data in a result structure (e.g., `JobHandleWithData<IMesherJob>`) to allow the main thread to wait for completion and access the generated mesh data [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assuming a job system interface similar to Unity's Job System or similar parallel frameworks

struct MarchingCubesJob:
    VoxelData: VoxelDataVolume<byte>
    VoxelColors: VoxelDataVolume<Color32>
    Isolevel: float
    VertexCountCounter: NativeCounter
    OutputVertices: NativeArray<MeshingVertexData>
    OutputTriangles: NativeArray<ushort>

    # The actual marching cubes logic for a single chunk is executed here.
    # It iterates over all voxels in the chunk, determines edge intersections,
    # and writes vertices/triangles to OutputVertices/OutputTriangles.
    # It atomically increments VertexCountCounter with the number of new vertices.
    def Execute(self):
        # Implementation details of Marching Cubes per chunk
        # ...
        pass

function CreateMesh(chunkCoordinate):
    # 1. Retrieve data
    voxelData = GetVoxelDataVolume(chunkCoordinate)
    if voxelData is null:
        return null
    
    voxelColors = GetVoxelColorVolume(chunkCoordinate)
    if voxelColors is null:
        return null

    # 2. Allocate resources
    vertexCountCounter = AllocateNativeCounter(Allocator.TempJob)
    
    chunkSize = VoxelWorld.WorldSettings.ChunkSize
    voxelCount = chunkSize.x * chunkSize.y * chunkSize.z
    maxLength = 15 * voxelCount
    
    outputVertices = AllocateNativeArray<MeshingVertexData>(maxLength, Allocator.TempJob)
    outputTriangles = AllocateNativeArray<ushort>(maxLength, Allocator.TempJob)

    # 3. Configure job
    job = MarchingCubesJob(
        VoxelData=voxelData,
        VoxelColors=voxelColors,
        Isolevel=0.5f, # Default isovalue
        VertexCountCounter=vertexCountCounter,
        OutputVertices=outputVertices,
        OutputTriangles=outputTriangles
    )

    # 4. Schedule
    jobHandle = job.Schedule()

    # 5. Return handle and data wrapper
    return JobHandleWithData<IMesherJob>(jobHandle, job)
```

### Parameters
*   **Isolevel**: The density threshold for surface extraction. Default value is 0.5f [S1, unofficial].
*   **ChunkSize**: The dimensions of the voxel chunk. Typical values depend on memory constraints and cache efficiency, often powers of 2 (e.g., 16x16x16 or 32x32x32) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Bounds**: The `maxLength` calculation ($15 \times voxelCount$) is an upper bound. If the actual number of generated primitives exceeds this, buffer overflows will occur. This assumes a standard Marching Cubes configuration where each cell generates at most 14 triangles (often simplified to 15 for safety or specific variants) [S1, unofficial].
*   **Thread Safety**: The `VertexCountCounter` must be atomic to prevent race conditions when multiple jobs increment the global vertex count simultaneously [S1, unofficial].
*   **Data Availability**: If a voxel chunk is not yet loaded or generated, the job must handle this gracefully by returning null or skipping, preventing crashes [S1, unofficial].

## Subtopics

### Multi-Process Ground Truth Generation
A variant that orchestrates parallel processing of 3D models to generate ground truth data using multiple worker processes and a queue for synchronization [S2, unofficial]. It distributes object names across processes based on an even distribution list, creates worker processes targeting a specific ground truth generation function, and manages synchronization via a queue with timeout checks to ensure all workers complete and the queue is empty [S2, unofficial].

### Tandem Algorithm with Dumping
A strategy that enhances the tandem algorithm by reducing the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration [S3]. It drastically reduces triangle count compared to brute-force approaches and filters disconnected components based on geometric criteria (e.g., volume). On a 1626x7028x2000 dataset, it reduced 18,111 components to 328 [S3].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of voxels, $O(N)$, where $N$ is the total number of voxels in the volume [S1, unofficial][S3]. Parallelization reduces wall-clock time proportionally to the number of available processors, up to the overhead of synchronization [S3].
*   **Memory Complexity**: $O(N)$ for voxel storage plus $O(M)$ for output buffers, where $M$ is the maximum possible output size per chunk ($15 \times voxelCount$) [S1, unofficial]. The tandem variant with dumping reduces memory usage by filtering out small disconnected components [S3].
*   **Measured Performance**: On a 1626x7028x2000 dataset (40 GB), the parallel tandem approach extracted surfaces in 5 minutes using 56 processors, compared to more than 3 hours for the sequential tandem algorithm [S3].
*   **Strengths**: High scalability on multi-core systems; efficient handling of large datasets; explicit memory management prevents fragmentation [S1, unofficial][S3].
*   **Limitations**: Requires careful management of chunk boundaries and vertex merging if a coherent global mesh is needed (not explicitly detailed in the job scheduling article itself, but implied by the chunk-based approach) [S1, unofficial]. The upper-bound memory allocation can be wasteful if the surface density is low [S1, unofficial].
*   **When to Use**: Large volumetric datasets where real-time or near-real-time extraction is required, and the hardware supports parallel processing [S3].
*   **When Not to Use**: Small datasets where the overhead of job scheduling outweighs the benefits; systems with limited memory where the upper-bound allocation cannot be afforded [S1, unofficial].

## Background
The concept of parallelizing Marching Cubes has been explored in various contexts, including GPU-based extraction and multi-threaded CPU implementations. The specific job scheduling approach described here is derived from practical implementations in voxel terrain generation, such as the Eldemarkki Marching Cubes Terrain project [S1, unofficial]. The tandem algorithm variant with dumping was developed for geophysical applications, specifically for extracting surfaces from large seismic cubes for hydrocarbon exploration, where interactive exploration of disconnected components is crucial [S3].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
