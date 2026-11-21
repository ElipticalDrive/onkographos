# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job Scheduling, Chunk-based Marching Cubes Job Scheduling

## Overview
Marching Cubes Job Scheduling is a parallel execution strategy for isosurface extraction that decomposes a 3D volume into spatial chunks (sub-volumes) and assigns each chunk to an independent processing unit (job or worker process) for concurrent mesh generation [S1, S2, unofficial][S3]. It solves the scalability problem of processing large volumetric datasets by distributing the computational load of voxel classification and triangle generation across multiple CPU cores or GPU threads [S1, S2, unofficial][S3]. Unlike monolithic extraction methods, it enables incremental processing and memory management by handling one chunk at a time, though it requires synchronization mechanisms to aggregate results from independent outputs [S1, S2, unofficial][S3].

## How it works
The method orchestrates the parallel processing of 3D models or volumes by mapping spatial data regions to independent computational tasks. The specific implementation details vary depending on whether the target is a GPU compute pipeline (using job schedulers like Unity's Job System) or a multi-process CPU environment.

### Inputs and Outputs
*   **Input**: A 3D volume dataset (scalar or vectorial), typically stored in a structured format such as a voxel grid. The volume is partitioned into chunks of fixed dimensions (e.g., $ChunkSize.x \times ChunkSize.y \times ChunkSize.z$) [S1, S2, unofficial][S3].
*   **Output**: A collection of triangle meshes (vertices and indices) representing the isosurface for each chunk. In multi-process scenarios, these may be serialized as intermediate files or passed via queues [S2, unofficial][S3].

### Data Structures
The data structures depend on the execution environment:

**1. GPU/Job System Layout (Unity C# Example)**
*   **`MarchingCubesJob` Struct**: Contains references to the input data and output buffers [S1, unofficial].
    *   `VoxelData`: `VoxelDataVolume<byte>` (or similar), storing the scalar field values for the chunk [S1, unofficial].
    *   `VoxelColors`: `VoxelDataVolume<Color32>` (optional), storing color/opacity data [S1, unofficial].
    *   `Isolevel`: `float`, the threshold for surface extraction (default 0.5) [S1, unofficial].
    *   `VertexCountCounter`: `NativeCounter`, an atomic counter to track the number of generated vertices/triangles for dynamic buffer allocation [S1, unofficial].
    *   `OutputVertices`: `NativeArray<MeshingVertexData>`, a pre-allocated buffer for vertex positions, normals, and colors [S1, unofficial].
    *   `OutputTriangles`: `NativeArray<ushort>`, a pre-allocated buffer for triangle indices [S1, unofficial].
*   **Buffer Sizing**: The maximum length of the output buffers is calculated as $MaxLength = 15 \times VoxelCount$, where $VoxelCount$ is the number of voxels in the chunk. This assumes a maximum of 5 triangles per voxel, with 3 vertices per triangle, and potentially storing additional data per vertex [S1, unofficial].

**2. Multi-Process Layout (Python Example)**
*   **`obj_names`**: List of dataset identifiers to process [S2, unofficial].
*   **`Queue`**: A synchronization primitive for workers to report status or pass data [S2, unofficial].
*   **`list_of_list_of_names`**: A partitioned list where each sublist contains the names of objects assigned to a specific worker process [S2, unofficial].

### Steps
The general workflow for chunk-based job scheduling is as follows:

1.  **Partitioning**: Divide the global volume into spatial chunks. Calculate the total number of voxels per chunk: $VoxelCount = ChunkSize.x \times ChunkSize.y \times ChunkSize.z$ [S1, unofficial].
2.  **Job Initialization**:
    *   Allocate output buffers with sufficient capacity ($15 \times VoxelCount$) [S1, unofficial].
    *   Initialize an atomic counter (`VertexCountCounter`) to 0 [S1, unofficial].
    *   Populate the `MarchingCubesJob` struct with references to the chunk's voxel data, the isovalue, the counter, and the output buffers [S1, unofficial].
3.  **Scheduling**:
    *   **GPU/Job System**: Schedule the job using the platform's job scheduler (e.g., `job.Schedule()`). The scheduler maps threads to voxels within the chunk [S1, unofficial].
    *   **Multi-Process**: Distribute the workload. Calculate the number of processes ($N_{proc}$) and assign subsets of the data (`obj_names`) to each worker based on an index modulo $N_{proc}$ [S2, unofficial]. Launch worker processes targeting a function like `get_gt_from_intersectionpn` [S2, unofficial].
4.  **Execution (Per Chunk/Worker)**:
    *   Each thread/process iterates over its assigned voxels.
    *   Compute the case index $C$ for each voxel using the parent method's logic [Parent].
    *   Look up edges and vertices in the `edgeTable` and `triangleTable` [Parent].
    *   Write generated triangles to the output buffers. Use atomic operations (GPU) or thread-local buffers followed by merging (CPU) to handle variable output sizes [S1, S2, unofficial][S3].
5.  **Synchronization and Aggregation**:
    *   **GPU**: Wait for the job handle to complete. Read the final vertex count from the atomic counter to determine the valid size of the output arrays [S1, unofficial].
    *   **Multi-Process**: Loop until all workers exit and the queue is empty. Check worker exit codes and process any returned data or status flags with a timeout [S2, unofficial].
6.  **Post-Processing (Optional)**:
    *   Filter disconnected components based on geometric criteria (e.g., volume) to reduce noise [S3].
    *   Merge material surfaces if processing vectorial volumes [S4].

### Pseudocode (GPU Job System Style)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the scheduling logic, not the voxel processing itself
def schedule_marching_cubes_job(chunk_coordinate, voxel_world):
    # 1. Retrieve chunk data
    chunk_size = voxel_world.world_settings.chunk_size
    voxel_count = chunk_size.x * chunk_size.y * chunk_size.z
    
    # 2. Allocate buffers
    # Max triangles per voxel is 5, 3 vertices per triangle. 
    # Assuming MeshingVertexData stores 3 floats for position + others.
    # OutputTriangles stores 3 ushort indices per triangle.
    max_length = 15 * voxel_count 
    
    output_vertices = NativeArray<MeshingVertexData>(max_length, Allocator.TempJob)
    output_triangles = NativeArray<ushort>(max_length, Allocator.TempJob)
    vertex_count_counter = NativeCounter(Allocator.TempJob)
    
    # 3. Get voxel data for this chunk
    voxel_data = voxel_world.get_voxel_data_chunk(chunk_coordinate)
    voxel_colors = voxel_world.get_voxel_color_chunk(chunk_coordinate)
    
    if voxel_data is None or voxel_colors is None:
        return None

    # 4. Initialize Job Struct
    job = MarchingCubesJob(
        VoxelData=voxel_data,
        VoxelColors=voxel_colors,
        Isolevel=0.5f,
        VertexCountCounter=vertex_count_counter,
        OutputVertices=output_vertices,
        OutputTriangles=output_triangles
    )
    
    # 5. Schedule Job
    job_handle = job.schedule()
    
    # 6. Return handle and data for later retrieval
    return JobHandleWithData<IMesherJob>(job_handle, job)

def process_multi_process_workers(obj_list, num_processes):
    # 1. Partition data
    list_of_list_of_names = []
    for i in range(num_processes):
        list_of_list_of_names.append([])
        
    for idx, name in enumerate(obj_list):
        list_of_list_of_names[idx % num_processes].append(name)
        
    # 2. Create Queue and Workers
    q = Queue()
    workers = []
    for pid in range(num_processes):
        p = Process(target=get_gt_from_intersectionpn, args=(q, list_of_list_of_names[pid]))
        workers.append(p)
        p.start()
        
    # 3. Monitor and Aggregate
    success_counter = 0
    while True:
        try:
            item = q.get(timeout=1.0)
            success_counter += item.success_flag
        except Empty:
            pass
            
        # Check if all workers exited
        if all(p.exitcode is not None for p in workers) and q.empty():
            break
            
    return success_counter
```

### Parameters
*   **Isolevel**: The density threshold for surface extraction. Default value is 0.5f [S1, unofficial].
*   **ChunkSize**: The dimensions of each spatial chunk. Determines the granularity of parallelism and memory usage per job [S1, unofficial].
*   **NumProcesses**: The number of worker processes or threads to spawn [S2, unofficial][S3].

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If the number of triangles exceeds the pre-allocated size ($15 \times VoxelCount$), data corruption occurs. The factor 15 is a theoretical upper bound (5 triangles/voxel * 3 vertices/triangle), but if vertex data is larger, the buffer size must be adjusted [S1, unofficial].
*   **Atomic Contention**: On GPUs, high contention on the `VertexCountCounter` can reduce performance. Histogram-based prefix-sum approaches avoid this but require two passes [Parent].
*   **Boundary Artifacts**: Chunks processed independently may produce disconnected triangles at chunk boundaries. These must be merged or filtered in post-processing [S3].
*   **Empty Chunks**: If a chunk contains no voxels crossing the isosurface, the job should return early to save resources [S1, unofficial].

## Subtopics

### Multi-Process Ground Truth Generation
A strategy using multiple worker processes and a queue for synchronization to generate ground truth data from 3D models [S2, unofficial]. It distributes object names across workers based on an index modulo the number of processes, allowing parallel processing of large datasets [S2, unofficial]. Workers report status via a queue, and the main process aggregates results until all workers exit [S2, unofficial].

### Tandem Algorithm with Dumping
An enhancement to the tandem algorithm that reduces the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration [S3]. It filters small components based on geometric criteria (e.g., volume), reducing 18,111 components to 328 in a 1626x7028x2000 dataset [S3]. This approach drastically reduces memory consumption and extraction time (5 minutes with 56 processors vs. 3 hours sequentially) [S3].

### Surface Rendering for Vectorial Volumes
A technique adapted for vectorial volumes by treating each classified material distribution as an independent scalar volume [S4]. It extracts surfaces for each material using standard Marching Cubes on these binary-like volumes (values in [0, 1]) and renders them as multiple objects in the same scene to merge materials in the final image [S4].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the total number of voxels, distributed across $P$ processors. Theoretical speedup is linear with $P$, limited by synchronization overhead and load imbalance [S3].
*   **Memory Complexity**: $O(V_{chunk})$ per job, where $V_{chunk}$ is the number of voxels in a chunk. This allows processing volumes larger than available RAM by processing chunks sequentially or in parallel with limited memory [S1, unofficial].
*   **Measured Performance**: On a 1626x7028x2000 dataset (40 GB), the parallel tandem approach extracted surfaces in 5 minutes using 56 processors, compared to more than 3 hours for the sequential algorithm [S3].
*   **Strengths**: Scalability to large datasets, memory efficiency through chunking, and compatibility with both CPU and GPU parallel architectures [S1, S2, unofficial][S3].
*   **Limitations**: Requires careful management of output buffers and synchronization. Boundary artifacts between chunks may require post-processing [S3]. Atomic counters can become a bottleneck on GPUs [Parent].
*   **When to Use**: Large volumetric datasets that do not fit in memory or require interactive exploration of simplified surfaces [S3]. When hardware supports parallel execution (multi-core CPU or GPU) [S1, S2, unofficial][S3].
*   **When Not to Use**: Small datasets where the overhead of job scheduling outweighs the benefits. When strict topological continuity across chunk boundaries is required without post-processing [S3].

## Background
The concept of parallelizing Marching Cubes has evolved from early CPU-based multi-threading to GPU compute shaders and distributed multi-process systems. The tandem algorithm, originally developed for seismic data analysis, introduced component-based dumping to handle large datasets [S3]. The use of job schedulers (like Unity's Job System) for Marching Cubes allows for efficient CPU/GPU parallelism in real-time applications like voxel terrain generation [S1, unofficial]. Multi-process approaches are often used in scientific computing for generating ground truth data or processing massive geophysical datasets [S2, unofficial][S3]. Surface rendering for vectorial volumes extends these techniques to handle multi-material data by decomposing the problem into independent scalar extractions [S4].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S4] A Multi-Volume Visualization Framework for Spatial Aligned Volumes after 3D/3D Image Registration, https://doi.org/10.1109/ICISE.2009.77, fetched 2026-10-26
