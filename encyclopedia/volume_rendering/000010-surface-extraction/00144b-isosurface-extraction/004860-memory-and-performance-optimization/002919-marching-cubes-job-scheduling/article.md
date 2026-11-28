# Marching Cubes Job Scheduling


## Overview
Marching Cubes Job Scheduling decomposes a 3D volume into spatial chunks and assigns each chunk to an independent computational task (GPU job or CPU worker process) for concurrent isosurface extraction [S1, S2, unofficial][S3]. It addresses the memory and computational limits of processing large volumetric datasets by distributing voxel classification and triangle generation across parallel hardware [S1, S2, unofficial][S3]. The method requires synchronization to aggregate independent chunk outputs into a final mesh or dataset [S1, S2, unofficial][S3].

## How it works
The method orchestrates parallel processing by mapping spatial data regions to independent tasks, with implementation details varying between GPU compute pipelines and multi-process CPU environments [S1, S2, unofficial].

### Inputs and Outputs
*   **Input**: A 3D volume dataset partitioned into fixed-dimension chunks (e.g., $ChunkSize.x \times ChunkSize.y \times ChunkSize.z$) [S1, unofficial].
*   **Output**: A collection of triangle meshes (vertices and indices) per chunk, which may be serialized or passed via queues in multi-process setups [S1, S2, unofficial].

### Data Structures
The data structures depend on the execution environment:

**1. GPU/Job System Layout**
*   **`MarchingCubesJob` Struct**: Contains references to input data and output buffers [S1, unofficial].
    *   `VoxelData`: `VoxelDataVolume<byte>`, storing scalar field values for the chunk [S1, unofficial].
    *   `VoxelColors`: `VoxelDataVolume<Color32>`, storing color/opacity data [S1, unofficial].
    *   `Isolevel`: `float`, the threshold for surface extraction (default 0.5) [S1, unofficial].
    *   `VertexCountCounter`: `NativeCounter`, an atomic counter tracking generated vertices/triangles for dynamic buffer allocation [S1, unofficial].
    *   `OutputVertices`: `NativeArray`, a pre-allocated buffer for vertex data [S1, unofficial].
    *   `OutputTriangles`: `NativeArray`, a pre-allocated buffer for triangle indices [S1, unofficial].
*   **Buffer Sizing**: The maximum buffer length is calculated as $MaxLength = 15 \times VoxelCount$, where $VoxelCount$ is the product of chunk dimensions [S1, unofficial].

**2. Multi-Process Layout**
*   **`obj_names`**: List of dataset identifiers to process [S2, unofficial].
*   **`Queue`**: A synchronization primitive for workers to report status or pass data [S2, unofficial].
*   **`list_of_list_of_names`**: A partitioned list where each sublist contains dataset names assigned to a specific worker process [S2, unofficial].

### Steps
The general workflow for chunk-based job scheduling is as follows:

1.  **Partitioning**: Divide the global volume into spatial chunks. Calculate $VoxelCount = ChunkSize.x \times ChunkSize.y \times ChunkSize.z$ [S1, unofficial].
2.  **Job Initialization**: Allocate output buffers with capacity $15 \times VoxelCount$ [S1, unofficial]. Initialize an atomic counter (`VertexCountCounter`) to zero [S1, unofficial]. Populate the job struct with chunk voxel data, the isovalue, the counter, and output buffers [S1, unofficial].
3.  **Scheduling**: 
    *   **GPU/Job System**: Schedule the job using the platform's scheduler (e.g., `job.Schedule()`), which maps threads to voxels within the chunk [S1, unofficial].
    *   **Multi-Process**: Distribute the workload by calculating $N_{proc}$ and assigning `obj_names` to workers based on `idx % N_proc` [S2, unofficial]. Launch worker processes targeting a handler like `get_gt_from_intersectionpn` [S2, unofficial].
4.  **Execution (Per Chunk/Worker)**: Each thread/process iterates over assigned voxels, computes the case index $C$ using the parent method's logic [Parent], looks up edges and vertices in `edgeTable` and `triangleTable` [Parent], and writes generated triangles to output buffers using atomics or thread-local merging [S1, S2, unofficial].
5.  **Synchronization and Aggregation**: 
    *   **GPU**: Wait for the job handle to complete and read the final vertex count from the atomic counter to determine valid buffer size [S1, unofficial].
    *   **Multi-Process**: Loop until all workers exit and the queue is empty, checking exit codes and processing status flags with a timeout [S2, unofficial].
6.  **Post-Processing (Optional)**: Filter disconnected components based on geometric criteria (e.g., volume) to reduce noise [S3]. Merge material surfaces if processing vectorial volumes [S4].

### Pseudocode (GPU Job System Style)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def schedule_marching_cubes_job(chunk_coordinate, voxel_world):
    chunk_size = voxel_world.world_settings.chunk_size
    voxel_count = chunk_size.x * chunk_size.y * chunk_size.z
    max_length = 15 * voxel_count
    output_vertices = NativeArray(max_length, Allocator.TempJob)
    output_triangles = NativeArray(max_length, Allocator.TempJob)
    vertex_count_counter = NativeCounter(Allocator.TempJob)
    voxel_data = voxel_world.get_voxel_data_chunk(chunk_coordinate)
    voxel_colors = voxel_world.get_voxel_color_chunk(chunk_coordinate)
    if voxel_data is None or voxel_colors is None:
        return None
    job = MarchingCubesJob(
        VoxelData=voxel_data,
        VoxelColors=voxel_colors,
        Isolevel=0.5,
        VertexCountCounter=vertex_count_counter,
        OutputVertices=output_vertices,
        OutputTriangles=output_triangles
    )
    job_handle = job.schedule()
    return JobHandleWithData(job_handle, job)

def process_multi_process_workers(obj_list, num_processes):
    list_of_list_of_names = [[] for _ in range(num_processes)]
    for idx, name in enumerate(obj_list):
        list_of_list_of_names[idx % num_processes].append(name)
    q = Queue()
    workers = []
    for pid in range(num_processes):
        p = Process(target=get_gt_from_intersectionpn, args=(q, list_of_list_of_names[pid]))
        workers.append(p)
        p.start()
    success_counter = 0
    while True:
        try:
            item = q.get(timeout=1.0)
            success_counter += item.success_flag
        except Empty:
            pass
        if all(p.exitcode is not None for p in workers) and q.empty():
            break
    return success_counter
```

### Parameters
*   **Isolevel**: The density threshold for surface extraction. Default value is 0.5 [S1, unofficial].
*   **ChunkSize**: The dimensions of each spatial chunk. Determines parallelism granularity and per-job memory usage [S1, unofficial].
*   **NumProcesses**: The number of worker processes or threads to spawn [S2, unofficial].

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If the number of triangles exceeds the pre-allocated size ($15 \times VoxelCount$), data corruption occurs [S1, unofficial].
*   **Atomic Contention**: On GPUs, high contention on the `VertexCountCounter` can reduce performance [Parent].
*   **Boundary Artifacts**: Chunks processed independently may produce disconnected triangles at chunk boundaries [S3].
*   **Empty Chunks**: If a chunk contains no voxels crossing the isosurface, the job should return early to save resources [G].

## Subtopics

### Multi-Process Ground Truth Generation
A strategy using multiple worker processes and a queue for synchronization to generate ground truth data from 3D models [S2, unofficial]. It distributes object names across workers based on an index modulo the number of processes, allowing parallel processing of large datasets [S2, unofficial]. Workers report status via a queue, and the main process aggregates results until all workers exit [S2, unofficial].

### Tandem Algorithm with Dumping
An enhancement to the tandem algorithm that reduces the number of generated triangles through a dumping mechanism, specifically handling disconnected components to allow interactive exploration [S3]. It filters small components based on geometric criteria (e.g., volume), reducing 18,111 components to 328 in a 1626x7028x2000 dataset [S3]. This approach drastically reduces memory consumption and extraction time (5 minutes with 56 processors vs. 3 hours sequentially) [S3].

### Surface Rendering for Vectorial Volumes
A technique adapted for vectorial volumes by treating each classified material distribution as an independent scalar volume [S4]. It extracts surfaces for each material using standard Marching Cubes on these binary-like volumes (values in [0, 1]) and renders them as multiple objects in the same scene to merge materials in the final image [S4].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the total number of voxels, distributed across processors [G].
*   **Memory Complexity**: $O(V_{chunk})$ per job, where $V_{chunk}$ is the number of voxels in a chunk. This allows processing volumes larger than available RAM by handling chunks with limited memory [S1, unofficial].
*   **Measured Performance**: On a 1626x7028x2000 dataset (40 GB), the parallel tandem approach extracted surfaces in 5 minutes using 56 processors, compared to more than 3 hours for the sequential algorithm [S3].
*   **Strengths**: Scalability to large datasets and memory efficiency through chunking [S1, S2, unofficial][S3].
*   **Limitations**: Requires synchronization to aggregate outputs and careful buffer management [S1, S2, unofficial]. Atomic counters can become a bottleneck on GPUs [Parent].
*   **When to Use**: Large volumetric datasets that require interactive exploration of simplified surfaces [S3]. Hardware supporting parallel execution [S1, S2, unofficial][S3].
*   **When Not to Use**: Small datasets where the overhead of job scheduling outweighs the benefits [G].

## Background
The concept of parallelizing Marching Cubes has evolved from early CPU-based multi-threading to GPU compute shaders and distributed multi-process systems. The tandem algorithm, originally developed for seismic data analysis, introduced component-based dumping to handle large datasets [S3]. The use of job schedulers (like Unity's Job System) for Marching Cubes allows for efficient CPU/GPU parallelism in real-time applications like voxel terrain generation [S1, unofficial]. Multi-process approaches are often used in scientific computing for generating ground truth data or processing massive geophysical datasets [S2, unofficial][S3]. Surface rendering for vectorial volumes extends these techniques to handle multi-material data by decomposing the problem into independent scalar extractions [S4].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S4] A Multi-Volume Visualization Framework for Spatial Aligned Volumes after 3D/3D Image Registration, https://doi.org/10.1109/ICISE.2009.77, fetched 2026-10-26
