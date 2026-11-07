# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job, Chunk-based MC Job Scheduling

## Overview
Marching Cubes Job Scheduling is a parallel execution strategy for isosurface extraction that decomposes a volumetric dataset into spatial chunks and processes them concurrently using a job system [S1, unofficial]. It solves the performance bottleneck of single-threaded mesh generation by distributing the computational load of vertex and triangle generation across multiple worker threads or processes [S1, S2, unofficial]. Unlike sequential implementations, this method relies on atomic counters for output buffer management and explicit synchronization mechanisms to handle data dependencies between chunk processing and ground truth generation tasks [S1, S2, unofficial].

## How it works
The method operates by dividing the voxel world into fixed-size chunks, scheduling independent marching cubes computations for each chunk, and aggregating results into a global mesh structure.

### Inputs and Outputs
*   **Input**: A voxel data volume (`VoxelDataVolume<byte>`) and an optional color volume (`VoxelDataVolume<Color32>`).
*   **Input**: The isovalue threshold (`Isolevel`), typically a float in the range [0, 1] [S1, unofficial].
*   **Input**: Chunk coordinates defining the spatial region to process.
*   **Output**: A list of mesh vertices (`NativeArray<MeshingVertexData>`) and triangle indices (`NativeArray<ushort>`).
*   **Output**: A job handle representing the asynchronous execution state.

### Data Structures
The core scheduling unit is the `MarchingCubesJob` struct, which encapsulates all necessary data for a single chunk's extraction [S1, unofficial].

**`MarchingCubesJob` Layout:**
*   `VoxelData`: Reference to the byte volume containing scalar values.
*   `VoxelColors`: Reference to the color volume (optional).
*   `Isolevel`: Float threshold for surface detection.
*   `VertexCountCounter`: An atomic counter (`NativeCounter`) to manage global vertex allocation.
*   `OutputVertices`: A pre-allocated `NativeArray` of `MeshingVertexData` structures.
*   `OutputTriangles`: A pre-allocated `NativeArray` of `ushort` indices.

**`MeshingVertexData` Layout:**
*   `Position`: 3D vector (x, y, z).
*   `Color`: 3D vector (r, g, b).
*   `Normal`: 3D vector (nx, ny, nz).

### Algorithm Steps
1.  **Chunk Initialization**: The scheduler receives a `chunkCoordinate`. It attempts to retrieve the corresponding `VoxelData` and `VoxelColors` chunks. If either is missing, the job returns null or an empty handle [S1, unofficial].
2.  **Buffer Allocation**:
    *   Calculate `voxelCount` as the product of the chunk's dimensions ($W \times H \times D$) [S1, unofficial].
    *   Estimate `maxLength` for output buffers. A common heuristic is $15 \times \text{voxelCount}$, assuming a maximum number of triangles per voxel [S1, unofficial].
    *   Allocate `outputVertices` and `outputTriangles` arrays of size `maxLength` using temporary job allocators [S1, unofficial].
    *   Initialize a `VertexCountCounter` with `Allocator.TempJob` [S1, unofficial].
3.  **Job Configuration**: Populate the `MarchingCubesJob` struct with the voxel data references, the `Isolevel` (default 0.5f), the counter, and the output arrays [S1, unofficial].
4.  **Scheduling**: The job is scheduled to the job system (e.g., Unity Job System). The scheduler wraps the job handle and data in a `JobHandleWithData<IMesherJob>` object and returns it to the caller [S1, unofficial].
5.  **Execution (Worker Side)**:
    *   The worker iterates over all voxels in the chunk.
    *   For each voxel, it evaluates the 8 corner values against the `Isolevel`.
    *   If a surface intersection is detected, the marching cubes algorithm determines the edge intersections and triangle topology.
    *   Vertices are appended to `OutputVertices`. The `VertexCountCounter` is atomically incremented to track the current write position.
    *   Triangle indices are appended to `OutputTriangles` [S1, unofficial].
6.  **Synchronization (Multi-Process Variant)**: In ground truth generation scenarios, a queue-based synchronization mechanism is used. Worker processes read object names, distribute them based on process ID, and write results to a shared queue. The main loop polls the queue with a timeout to aggregate results until all workers exit [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Core Job Scheduling Logic (Single-Threaded Job System Style)

struct MarchingCubesJob:
    VoxelData: VoxelDataVolume<byte>
    VoxelColors: VoxelDataVolume<Color32>
    Isolevel: float
    VertexCountCounter: NativeCounter
    OutputVertices: NativeArray<MeshingVertexData>
    OutputTriangles: NativeArray<ushort>

    def Execute(self, chunkCoordinate: int3):
        # 1. Get chunk data
        voxelData = self.VoxelData.GetChunk(chunkCoordinate)
        if voxelData is None:
            return

        # 2. Iterate voxels
        for x in range(chunkCoordinate.x * ChunkSize, (chunkCoordinate.x + 1) * ChunkSize):
            for y in range(chunkCoordinate.y * ChunkSize, (chunkCoordinate.y + 1) * ChunkSize):
                for z in range(chunkCoordinate.z * ChunkSize, (chunkCoordinate.z + 1) * ChunkSize):
                    
                    # 3. Evaluate voxel
                    corners = [
                        voxelData.Get(x, y, z),
                        voxelData.Get(x+1, y, z),
                        voxelData.Get(x, y+1, z),
                        voxelData.Get(x+1, y+1, z),
                        voxelData.Get(x, y, z+1),
                        voxelData.Get(x+1, y, z+1),
                        voxelData.Get(x, y+1, z+1),
                        voxelData.Get(x+1, y+1, z+1)
                    ]
                    
                    # 4. Marching Cubes Logic
                    # Determine case index based on sign of (corner - Isolevel)
                    caseIndex = 0
                    for i, val in enumerate(corners):
                        if val < self.Isolevel:
                            caseIndex |= (1 << i)
                    
                    if caseIndex == 0 or caseIndex == 255:
                        continue # No intersection
                    
                    # 5. Generate Triangles
                    # (Standard MC lookup table logic omitted for brevity, assumed implemented)
                    triangles = GetTrianglesFromCase(caseIndex, corners, chunkCoordinate, self.Isolevel)
                    
                    for tri in triangles:
                        # Append vertices
                        vertexIndex = self.VertexCountCounter.AddAndGet(1)
                        for v in tri.vertices:
                            self.OutputVertices[vertexIndex] = v
                            vertexIndex += 1
                        
                        # Append indices
                        self.OutputTriangles.Append(tri.indices)

# Multi-Process Ground Truth Scheduling Logic
def ScheduleGroundTruthGeneration(obj_names, num_of_process):
    # 1. Distribute work
    list_of_list_of_names = distribute_evenly(obj_names, num_of_process)
    
    # 2. Create workers
    workers = []
    for pid in range(P_start, P_end):
        p = Process(target=get_gt_from_intersectionpn, args=(queue, list_of_list_of_names[pid]))
        workers.append(p)
        p.start()
        
    # 3. Synchronize
    while True:
        try:
            item = queue.get(timeout=1.0)
            if item.success_flag:
                counter += 1
        except Empty:
            pass
            
        if all(p.exitcode is not None for p in workers) and queue.empty():
            break
```

### Parameters
*   **Isolevel**: The density threshold. Default: `0.5f`. Range: `[0, 1]` [S1, unofficial].
*   **ChunkSize**: Dimensions of the voxel chunk. Typical values depend on memory constraints and cache locality (e.g., $16^3$, $32^3$). Not given in the sources.
*   **maxLength**: Buffer size multiplier. Default: `15` times voxel count [S1, unofficial].

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If the estimated `maxLength` is insufficient, the job may write out of bounds. The heuristic `15 * voxelCount` is used but may fail for highly complex surfaces [S1, unofficial].
*   **Missing Chunks**: If a voxel chunk is not loaded or generated, the job must handle this gracefully by returning null or skipping the chunk [S1, unofficial].
*   **Atomic Contention**: High concurrency on the `VertexCountCounter` can lead to contention. Using efficient atomic operations is critical [S1, unofficial].
*   **Synchronization Deadlocks**: In multi-process setups, improper queue handling or timeout settings can lead to infinite loops if workers fail silently [S2, unofficial].

## Subtopics

### Multi-Process Ground Truth Generation
A variant of the scheduling method used for generating ground truth data for 3D models. It orchestrates parallel processing using multiple worker processes and a synchronization queue [S2, unofficial]. It distributes object names evenly among processes, executes intersection-based ground truth extraction, and aggregates results via a queue with timeout-based polling [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the processed chunks. Parallelization reduces wall-clock time by a factor proportional to the number of available cores/processors, subject to synchronization overhead [S1, S2, unofficial].
*   **Memory Complexity**: $O(N)$ for the output buffers. Temporary allocators are used for job data, which must be freed after job completion to avoid leaks [S1, unofficial].
*   **Performance**: Depends heavily on the efficiency of the underlying marching cubes implementation and the job system's thread pool management. The `15 * voxelCount` buffer size provides a balance between memory usage and overflow risk [S1, unofficial].
*   **Strengths**: Scalable to large volumes; decouples mesh generation from the main thread; supports concurrent ground truth generation [S1, S2, unofficial].
*   **Limitations**: Requires careful memory management (temporary allocators); buffer size estimation is heuristic; synchronization overhead in multi-process setups [S1, S2, unofficial].

## Background
The method is derived from implementations in voxel terrain generation systems, specifically the `Eldemarkki/Marching-Cubes-Terrain` repository, which utilizes Unity's Job System for parallel mesh generation [S1, unofficial]. The multi-process variant is associated with ground truth generation pipelines for neural 3D reconstruction, such as the `NDC` repository, which uses Python multiprocessing for data preparation [S2, unofficial].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
