# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job Dispatch, Chunk-based MC Job Scheduling

## Overview
Marching Cubes Job Scheduling is a method for orchestrating the parallel execution of isosurface extraction on a GPU or multi-core CPU by decomposing the volume into independent processing units (chunks or jobs). It solves the problem of managing variable output sizes and synchronization overhead when extracting meshes from large voxel grids. Unlike monolithic extraction approaches, this method schedules discrete jobs that process specific spatial regions (e.g., voxel chunks) independently, leveraging precomputed lookup tables defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md). It sets itself apart by explicitly managing the lifecycle of job data structures (allocations, scheduling, and handle wrapping) to enable asynchronous execution and memory safety, distinct from immediate-mode CPU rendering [2463-cpu-immediate-mode-marching-cubes.md] or pure atomic-counter GPU pipelines [4082-gpu-marching-cubes-with-atomic-counter-emission.md].

## How it works
The method operates by creating discrete job structures that encapsulate the necessary input data, configuration, and output buffers for a specific spatial region of the volume. The scheduler manages the allocation of these buffers, the dispatch of the computation, and the retrieval of results.

### Inputs and Outputs
*   **Input**: A 3D voxel grid divided into chunks. Each chunk contains a `VoxelDataVolume<byte>` (density values) and optionally a `VoxelDataVolume<Color32>` (color values).
*   **Output**: A contiguous mesh consisting of `outputVertices` (positions, normals, colors) and `outputTriangles` (indices).
*   **Configuration**: An `Isolevel` threshold for surface extraction.

### Data Structures
The core data structure is the `MarchingCubesJob` struct, which holds the state for a single processing unit.

**`MarchingCubesJob` Layout:**
*   `VoxelData`: `VoxelDataVolume<byte>` - The density data for the chunk.
*   `VoxelColors`: `VoxelDataVolume<Color32>` - The color data for the chunk (optional).
*   `Isolevel`: `float` - The threshold for extraction.
*   `VertexCountCounter`: `NativeCounter` - An atomic counter to track the number of vertices/triangles generated.
*   `OutputVertices`: `NativeArray<MeshingVertexData>` - Pre-allocated buffer for vertex data.
*   `OutputTriangles`: `NativeArray<ushort>` - Pre-allocated buffer for triangle indices.

**Buffer Sizing:**
The output buffers are pre-allocated with a maximum capacity to avoid reallocation during the job.
*   `voxelCount`: The total number of voxels in the chunk, calculated as $W \times H \times D$, where $W, H, D$ are the chunk dimensions.
*   `maxLength`: The maximum number of triangles/vertices the chunk can produce. A common upper bound is $15 \times \text{voxelCount}$, assuming a worst-case scenario where each voxel generates a constant number of triangles [S1, unofficial].

### Algorithm Steps

1.  **Chunk Retrieval**:
    *   Request the voxel data chunk (`VoxelDataVolume<byte>`) and color chunk (`VoxelDataVolume<Color32>`) for a given `chunkCoordinate`.
    *   If either chunk fails to load or is null, return null or an empty job handle [S1, unofficial].

2.  **Buffer Allocation**:
    *   Allocate a `NativeCounter` (`vertexCountCounter`) using a temporary allocator (e.g., `Allocator.TempJob`) to track the number of generated primitives [S1, unofficial].
    *   Calculate `voxelCount` based on the chunk dimensions.
    *   Calculate `maxLength` as $15 \times \text{voxelCount}$ [S1, unofficial].
    *   Allocate `outputVertices` as a `NativeArray` of `MeshingVertexData` with size `maxLength`.
    *   Allocate `outputTriangles` as a `NativeArray` of `ushort` with size `maxLength`.
    *   All arrays should use the same allocator type as the counter to ensure proper lifecycle management [S1, unofficial].

3.  **Job Initialization**:
    *   Instantiate the `MarchingCubesJob` struct.
    *   Assign the retrieved `VoxelData` and `VoxelColors`.
    *   Set the `Isolevel` parameter (default $0.5$ if not specified) [S1, unofficial].
    *   Assign the `vertexCountCounter`, `outputVertices`, and `outputTriangles` to the job struct.

4.  **Job Scheduling**:
    *   Schedule the `MarchingCubesJob` for execution on the target hardware (GPU compute shader or CPU job system) using the job system's schedule method (e.g., `job.Schedule()`).
    *   The job system returns a `JobHandle` representing the asynchronous task [S1, unofficial].

5.  **Result Wrapping**:
    *   Wrap the `JobHandle` and the associated job data (or references to the output buffers) in a result object (e.g., `JobHandleWithData<IMesherJob>`).
    *   Return this handle to the caller, allowing them to wait for completion and access the generated mesh data later [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MarchingCubesJobScheduler:
    def create_mesh(self, chunk_coordinate, voxel_world):
        """
        Creates and schedules a marching cubes job for a specific chunk.
        
        Args:
            chunk_coordinate: int3, the spatial coordinates of the chunk.
            voxel_world: VoxelWorld, the world object containing chunk data.
            
        Returns:
            JobHandleWithData, a handle to the asynchronous job.
        """
        # 1. Retrieve chunk data
        voxel_data = voxel_world.chunk_store.get_voxel_data(chunk_coordinate)
        voxel_colors = voxel_world.chunk_store.get_voxel_colors(chunk_coordinate)
        
        if voxel_data is None or voxel_colors is None:
            return None

        # 2. Calculate buffer sizes
        chunk_size = voxel_world.world_settings.chunk_size
        voxel_count = chunk_size.x * chunk_size.y * chunk_size.z
        # Max triangles per voxel is bounded; 15 is a safe upper bound for MC
        max_length = 15 * voxel_count

        # 3. Allocate buffers and counter
        # Using temporary allocator for job system compatibility
        vertex_count_counter = NativeCounter(allocator=Allocator.TempJob)
        output_vertices = NativeArray(MeshingVertexData, size=max_length, allocator=Allocator.TempJob)
        output_triangles = NativeArray(ushort, size=max_length, allocator=Allocator.TempJob)

        # 4. Initialize Job Struct
        job = MarchingCubesJob(
            voxel_data=voxel_data,
            voxel_colors=voxel_colors,
            isolevel=0.5,  # Default isosurface threshold
            vertex_count_counter=vertex_count_counter,
            output_vertices=output_vertices,
            output_triangles=output_triangles
        )

        # 5. Schedule Job
        job_handle = job.schedule()

        # 6. Wrap and Return
        return JobHandleWithData(IMesherJob, job_handle, job)
```

### Parameters
*   **Isolevel**: The density threshold for surface extraction. Typically ranges from $0.0$ to $1.0$. Default value is $0.5$ [S1, unofficial].
*   **Chunk Size**: The dimensions of the voxel chunk. Affects memory allocation and parallelism granularity.

### Edge Cases and Pitfalls
*   **Buffer Overflow**: If the actual number of triangles exceeds `maxLength`, the output will be truncated. The factor of $15$ is a heuristic; in degenerate cases or with complex topologies, this may need adjustment [S1, unofficial].
*   **Memory Leaks**: Temporary allocators (`Allocator.TempJob`) must be properly disposed of after the job completes and results are read, otherwise memory leaks occur in long-running applications [S1, unofficial].
*   **Boundary Handling**: Threads processing voxels at the edge of a chunk must check for neighbor existence to avoid out-of-bounds access when sampling the 8 corners of the cube, as per the parent article's principles [4136-gpu-acceleration-and-parallel-pipelines.md].

## Subtopics

### Multi-Process Ground Truth Generation
A variant of job scheduling used for generating ground truth data for 3D models using multiple worker processes. It orchestrates parallel processing by distributing a list of object names (`obj_names`) across worker processes based on a modulo index (`idx % num_of_process`). Each worker reads a subset of objects, processes them (e.g., generating point clouds and KNN features), and synchronizes via a queue (`Queue q`) with a timeout mechanism to ensure all workers complete and the queue is empty before termination [S2, unofficial].

### Tandem Algorithm with Dumping
A strategy that enhances the tandem algorithm by reducing the number of generated triangles through a "dumping" mechanism, specifically handling disconnected components to allow interactive exploration. It drastically reduces triangle count compared to brute-force approaches. On a $1626 \times 7028 \times 2000$ dataset (40 GB), the parallel tandem approach with dumping extracted surfaces in 5 minutes using 56 processors, compared to over 3 hours for the sequential version. It filters 18,111 initial disconnected components down to 328 based on geometric criteria (volume), making it suitable for seismic cube analysis in hydrocarbon exploration [S3].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of voxels in the chunk, $O(N)$, where $N$ is the number of voxels. The parallel nature allows for near-linear speedup with the number of available cores/GPU threads, limited by synchronization overhead and memory bandwidth.
*   **Memory Complexity**: $O(N)$ for the output buffers, where $N$ is the number of voxels. The pre-allocation of `maxLength` buffers ensures constant-time memory access but may lead to over-allocation if the actual triangle count is significantly lower than the worst-case bound.
*   **Performance**: The tandem algorithm variant with dumping achieved a 36x speedup (5 minutes vs. 3 hours) on a 40 GB dataset using 56 processors [S3].
*   **Strengths**: Asynchronous execution, memory safety through explicit allocation management, and scalability via chunking.
*   **Limitations**: Pre-allocation of buffers can waste memory if the mesh density is low. The choice of chunk size affects load balancing; too small increases overhead, too large reduces parallelism.
*   **When to Use**: Large-scale volume rendering where real-time or near-real-time feedback is needed, and the volume can be decomposed into independent spatial regions.
*   **When Not to Use**: Small volumes where the overhead of job scheduling outweighs the benefits, or when memory is extremely constrained and the worst-case triangle count is prohibitive.

## Background
The concept of job scheduling for Marching Cubes emerged from the need to handle large datasets in real-time applications, such as voxel-based terrain generation and geophysical data analysis. The specific implementation details described in [S1, unofficial] are derived from the `Eldemarkki/Marching-Cubes-Terrain` project, which utilizes Unity's Job System for parallel mesh generation. The tandem algorithm with dumping, described in [S3], was developed for seismic data interpretation, addressing the challenge of processing terabyte-scale datasets by combining parallel extraction with component-based simplification. The multi-process ground truth generation method [S2, unofficial] reflects broader trends in parallel computing for 3D data processing, where task distribution and synchronization are key.

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
