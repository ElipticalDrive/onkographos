# Marching Cubes Job Scheduling

Also called: Unity Job System Marching Cubes, Chunk-based Parallel MC Scheduling

## Overview
Marching Cubes Job Scheduling is a parallel processing strategy that decomposes a volumetric dataset into spatial chunks and dispatches independent Marching Cubes extraction tasks to a job scheduler (specifically the Unity Job System) for concurrent execution [S1, unofficial]. It solves the problem of CPU-bound isosurface extraction bottlenecks by offloading the classification, interpolation, and triangle emission steps to background worker threads, allowing the main thread to continue rendering or handling user input [S1, unofficial]. Unlike immediate-mode CPU approaches, it leverages multi-core parallelism through explicit job scheduling; unlike pure GPU compute shaders, it manages memory allocation and job synchronization via a high-level job system API, often targeting dynamic terrain or voxel worlds where chunks are generated or modified independently [S1, unofficial].

## How it works
The method operates on a chunk-based decomposition of the voxel world. Each chunk is processed independently by a scheduled job, which reads voxel data, computes the isosurface, and writes the resulting geometry to a shared output buffer using atomic counters for safe concurrent writes.

### Inputs and Outputs
**Inputs:**
*   `chunkCoordinate`: The 3D integer coordinate $(x, y, z)$ identifying the chunk within the world grid [S1, unofficial].
*   `VoxelDataVolume<byte>`: A sparse or dense volume containing the scalar field values (density) for the chunk [S1, unofficial].
*   `VoxelDataVolume<Color32>`: An optional volume containing color data for the vertices [S1, unofficial].
*   `Isolevel`: The scalar threshold for surface extraction. Default is $0.5$ [S1, unofficial].
*   `ChunkSize`: The dimensions of the chunk in voxels (e.g., $16 \times 16 \times 16$) [S1, unofficial].

**Outputs:**
*   `outputVertices`: A `NativeArray` of `MeshingVertexData` structures containing vertex positions, normals, and colors [S1, unofficial].
*   `outputTriangles`: A `NativeArray` of `ushort` indices defining the triangles [S1, unofficial].
*   `vertexCountCounter`: An atomic counter tracking the number of vertices/triangles emitted, used to manage the shared output buffer [S1, unofficial].

### Data Structures
**1. `MarchingCubesJob` Struct**
This struct holds the context for a single job execution. It is passed to the job scheduler.
*   `VoxelData`: `VoxelDataVolume<byte>` (Reference to scalar field) [S1, unofficial].
*   `VoxelColors`: `VoxelDataVolume<Color32>` (Reference to color field) [S1, unofficial].
*   `Isolevel`: `float` (Scalar threshold) [S1, unofficial].
*   `VertexCountCounter`: `NativeCounter` (Atomic integer for output buffer indexing) [S1, unofficial].
*   `OutputVertices`: `NativeArray<MeshingVertexData>` (Shared output buffer) [S1, unofficial].
*   `OutputTriangles`: `NativeArray<ushort>` (Shared output buffer) [S1, unofficial].

**2. `MeshingVertexData` Structure**
Represents a single vertex in the output mesh.
*   `position`: `float3` (World or local space coordinates) [G].
*   `normal`: `float3` (Surface normal) [G].
*   `color`: `float4` or `Color32` (Vertex color) [G].
*   *Note: Exact layout depends on the specific implementation of `MeshingVertexData` in the host application, but it must be contiguous for `NativeArray` compatibility [S1, unofficial].*

### Algorithm Steps
1.  **Chunk Validation**: Retrieve the `VoxelData` and `VoxelColors` chunks for the given `chunkCoordinate`. If either chunk is missing or invalid, return null or an empty result [S1, unofficial].
2.  **Resource Allocation**:
    *   Calculate `voxelCount` as the product of the chunk dimensions: $N_{voxels} = W \times H \times D$ [S1, unofficial].
    *   Calculate `maxLength` as the maximum possible number of triangles/vertices. A safe upper bound is $15 \times N_{voxels}$, assuming each voxel can generate up to 15 triangles in the worst-case Marching Cubes configuration [S1, unofficial].
    *   Allocate `outputVertices` and `outputTriangles` as `NativeArray` types with size `maxLength` using temporary memory allocators (e.g., `Allocator.TempJob`) [S1, unofficial].
    *   Initialize `VertexCountCounter` with an initial value of 0 [S1, unofficial].
3.  **Job Initialization**: Populate the `MarchingCubesJob` struct with the retrieved voxel data, colors, isovalue, counter, and output arrays [S1, unofficial].
4.  **Job Scheduling**: Schedule the `MarchingCubesJob` to the job scheduler. This returns a `JobHandle` [S1, unofficial].
5.  **Execution**: The scheduler executes the job on an available worker thread. The job iterates over all voxels in the chunk.
6.  **Per-Voxel Processing (Inside Job)**:
    *   For each voxel, calculate the case index $C$ based on the 8 corner scalar values $V_i$ relative to `Isolevel` [S1, S4, S12].
    *   If $C = 0$ or $C = 255$, skip the voxel (no intersection) [S1, S4].
    *   Use `edgeTable` to determine intersected edges [S1, S4].
    *   Use `triangleTable` to determine the sequence of edge indices for triangle generation [S1, S4].
    *   Interpolate vertex positions and normals on the intersected edges [S1, S4].
    *   Use `VertexCountCounter` to atomically reserve space in `outputVertices` and `outputTriangles` [S1, unofficial].
    *   Write the generated vertex data and triangle indices to the reserved slots [S1, unofficial].
7.  **Result Compilation**: After the job completes (synchronized via `JobHandle`), the main thread reads the final count from `VertexCountCounter` and trims the `outputVertices` and `outputTriangles` arrays to the actual size [S1, unofficial].
8.  **Mesh Construction**: Create a `Mesh` object, assign the trimmed arrays to the mesh's vertices and triangles arrays, and update normals/UVs if necessary [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the Job Scheduling Orchestrator (Main Thread)
function CreateMesh(chunkCoordinate, voxelWorld):
    # 1. Retrieve Chunk Data
    voxelData = voxelWorld.GetVoxelDataChunk(chunkCoordinate)
    voxelColors = voxelWorld.GetVoxelColorChunk(chunkCoordinate)
    
    if voxelData is None or voxelColors is None:
        return null

    # 2. Calculate Dimensions and Max Length
    chunkSize = voxelWorld.WorldSettings.ChunkSize
    voxelCount = chunkSize.x * chunkSize.y * chunkSize.z
    maxLength = 15 * voxelCount  # Upper bound for triangles/vertices

    # 3. Allocate Native Arrays and Counter
    # Note: In a real system, these are allocated on the heap/job system memory
    outputVertices = NativeArray<MeshingVertexData>(maxLength, Allocator.TempJob)
    outputTriangles = NativeArray<ushort>(maxLength, Allocator.TempJob)
    vertexCountCounter = NativeCounter(0, Allocator.TempJob)

    # 4. Initialize Job Struct
    job = MarchingCubesJob(
        VoxelData=voxelData,
        VoxelColors=voxelColors,
        Isolevel=0.5f,
        VertexCountCounter=vertexCountCounter,
        OutputVertices=outputVertices,
        OutputTriangles=outputTriangles
    )

    # 5. Schedule Job
    jobHandle = job.Schedule()

    # 6. Return Handle for later completion
    return JobHandleWithData(jobHandle, job)

# Pseudocode for the Marching Cubes Job (Worker Thread)
struct MarchingCubesJob:
    VoxelData: VoxelDataVolume<byte>
    VoxelColors: VoxelDataVolume<Color32>
    Isolevel: float
    VertexCountCounter: NativeCounter
    OutputVertices: NativeArray<MeshingVertexData>
    OutputTriangles: NativeArray<ushort>

    function Execute():
        chunkSize = self.VoxelData.ChunkSize
        for z in 0..chunkSize.z:
            for y in 0..chunkSize.y:
                for x in 0..chunkSize.x:
                    # Get 8 voxel values
                    values = [
                        self.VoxelData.Get(x, y, z),
                        self.VoxelData.Get(x+1, y, z),
                        self.VoxelData.Get(x+1, y+1, z),
                        self.VoxelData.Get(x, y+1, z),
                        self.VoxelData.Get(x, y, z+1),
                        self.VoxelData.Get(x+1, y, z+1),
                        self.VoxelData.Get(x+1, y+1, z+1),
                        self.VoxelData.Get(x, y+1, z+1)
                    ]
                    
                    # Calculate Case Index
                    caseIndex = 0
                    for i in 0..7:
                        if values[i] > self.Isolevel:
                            caseIndex |= (1 << i)
                    
                    # Skip if no intersection
                    if caseIndex == 0 or caseIndex == 255:
                        continue

                    # Lookup Edges and Triangles
                    edgeMask = edgeTable[caseIndex]
                    triangleIndices = triangleTable[caseIndex]
                    
                    # Generate Vertices and Triangles
                    # This part involves interpolating edge intersections
                    # and writing to the shared output arrays using the counter
                    # The exact interpolation logic is defined in 
                    # [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md)
                    
                    # Example of atomic write (pseudo):
                    # vertexIndex = self.VertexCountCounter.AddAndGet(1)
                    # self.OutputVertices[vertexIndex] = interpolatedVertex
                    # ...
```

### Parameters
*   `Isolevel`: The density threshold. Default: $0.5$ [S1, unofficial].
*   `ChunkSize`: Typically $16 \times 16 \times 16$ or $32 \times 32 \times 32$ in voxel terrain engines [S1, unofficial].
*   `maxLength` multiplier: $15$ is used as a safe upper bound for the number of triangles per voxel [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Conditions**: The job must handle voxels at the edge of the chunk. If the chunk is part of a larger world, the 8 corners of a voxel at the chunk boundary may reference data in adjacent chunks. The implementation must ensure `VoxelData` access handles these boundary reads correctly, either by padding the chunk data or by accessing the global volume with boundary checks [S1, G, unofficial].
*   **Atomic Contention**: Multiple threads writing to the same `outputVertices` and `outputTriangles` arrays require atomic operations (`NativeCounter`) to prevent race conditions. High contention can reduce performance [S1, unofficial].
*   **Memory Allocation**: Using `Allocator.TempJob` requires careful management to avoid memory leaks. The job handle must be completed before the memory is freed [S1, unofficial].
*   **Precision**: Floating-point precision in vertex interpolation can lead to visible seams if not handled consistently across chunk boundaries [G].

## Subtopics

### Multi-Process Ground Truth Generation
A variant that orchestrates parallel processing of 3D models to generate ground truth data using multiple worker processes and a queue for synchronization [S2, unofficial]. It distributes object names across processes based on an even distribution list, creates worker processes targeting a ground truth extraction function, and uses a queue with a timeout to collect results and monitor worker exit codes [S2, unofficial]. This is used for generating training data for neural networks, not for real-time rendering [S2, unofficial].

### Tandem Algorithm with Dumping
A strategy that enhances the tandem algorithm by reducing the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration [S3]. It filters out small disconnected components based on geometric criteria (e.g., volume), reducing 18,111 components to 328 in a large seismic dataset [S3]. This approach drastically reduces memory consumption and triangle count compared to brute-force methods, making it suitable for large-scale geophysical data analysis [S3].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk. The parallel execution reduces wall-clock time by a factor proportional to the number of available CPU cores [S1, unofficial].
*   **Memory Complexity**: $O(N)$ for the voxel data and $O(N)$ for the output buffers (with a constant factor of 15 for triangles) [S1, unofficial].
*   **Performance**: Performance scales with the number of CPU cores. In the Unity Job System, this can achieve near-linear speedup on multi-core CPUs [S1, unofficial].
*   **Strengths**:
    *   Leverages multi-core CPU parallelism without the complexity of GPU shader programming [S1, unofficial].
    *   Modular and easy to integrate with existing Unity workflows [S1, unofficial].
    *   Supports dynamic updates to individual chunks [S1, unofficial].
*   **Limitations**:
    *   Limited by CPU performance compared to GPU-based methods [S1, unofficial].
    *   Memory overhead for managing native arrays and job structures [S1, unofficial].
    *   Not suitable for extremely large datasets that exceed CPU memory bandwidth [S1, unofficial].
*   **When to Use**: Ideal for voxel terrain generation in games where chunks are dynamically loaded and unloaded, and CPU resources are available [S1, unofficial].
*   **When Not to Use**: Not suitable for real-time rendering of massive static volumes where GPU acceleration is preferred [S1, unofficial].

## Background
The Marching Cubes Job Scheduling method is part of the Eldemarkki Voxel Terrain project, an open-source voxel engine for Unity [S1, unofficial]. It was developed to provide a performant and easy-to-use solution for dynamic terrain generation in Unity, leveraging the Unity Job System for parallelism [S1, unofficial]. The method builds on the standard Marching Cubes algorithm and its lookup tables [S1, S4, S12].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
