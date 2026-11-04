# Marching Cubes Job Scheduling

Also called: Parallel Marching Cubes Job Scheduling

## Overview
This method orchestrates the parallel execution of Marching Cubes surface extraction on a GPU by decomposing the 3D voxel volume into independent chunks and scheduling compute jobs for each chunk. It solves the problem of managing asynchronous GPU workloads and dynamic output buffering in a chunk-based terrain or voxel system. Unlike monolithic approaches, it isolates mesh generation per chunk, allowing for incremental updates and memory efficiency. It builds on [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md) by providing the specific job scheduling infrastructure required to drive those pipelines in a chunked environment.

## How it works
The method operates by retrieving voxel data for a specific chunk coordinate, allocating temporary buffers for the mesh output, and scheduling a compute job that processes the chunk's voxels.

**Inputs and Outputs:**
*   **Input:** A 3D chunk coordinate $(C_x, C_y, C_z)$ defining the spatial region to process.
*   **Input:** A voxel data volume (scalar field) and optionally a color volume.
*   **Output:** A `JobHandle` representing the asynchronous completion of the mesh generation for that chunk.
*   **Output:** Implicitly, a triangle mesh is generated in the output buffers, which are later consumed by the rendering pipeline.

**Data Structures:**
The core scheduling unit is the `MarchingCubesJob` struct. Based on the implementation in [S1, unofficial], it contains the following fields:
1.  `VoxelData`: A reference to the scalar voxel volume (`VoxelDataVolume<byte>` or similar).
2.  `VoxelColors`: A reference to the color voxel volume (`VoxelDataVolume<Color32>`), if color is enabled.
3.  `Isolevel`: A `float` threshold for surface extraction.
4.  `VertexCountCounter`: An atomic counter (`NativeCounter`) to track the number of vertices generated.
5.  `OutputVertices`: A `NativeArray` of vertex data structures (`MeshingVertexData`).
6.  `OutputTriangles`: A `NativeArray` of triangle indices (`ushort`).

**Steps:**
1.  **Chunk Data Retrieval:** Attempt to retrieve the `VoxelData` and `VoxelColors` chunks corresponding to the input `chunkCoordinate`. If either chunk is not loaded or fails to retrieve, the method returns `null` or an empty handle, indicating no work is to be done [S1, unofficial].
2.  **Counter Allocation:** Allocate a `NativeCounter` (e.g., `vertexCountCounter`) using a temporary allocator. This counter will be used by the compute job to atomically increment the vertex count as triangles are generated [S1, unofficial].
3.  **Buffer Sizing:**
    *   Calculate `voxelCount` as the product of the chunk dimensions: $N_x \times N_y \times N_z$.
    *   Calculate `maxLength` as $15 \times \text{voxelCount}$. This upper bound assumes the worst-case scenario where every voxel generates the maximum number of triangles (5 triangles $\times$ 3 vertices = 15 vertices) [S1, unofficial].
    *   Allocate `outputVertices` as a `NativeArray` of size `maxLength`.
    *   Allocate `outputTriangles` as a `NativeArray` of size `maxLength` (assuming 3 indices per triangle, this capacity supports up to $5 \times \text{voxelCount}$ triangles) [S1, unofficial].
4.  **Job Initialization:** Populate the `MarchingCubesJob` struct with the retrieved voxel data, the `Isolevel` parameter, the allocated counter, and the output buffers.
5.  **Job Scheduling:** Schedule the `MarchingCubesJob` on the GPU compute queue. The job processes all voxels in the chunk, classifies cubes, looks up triangle configurations, and writes vertices/triangles to the output buffers using atomic operations for index management.
6.  **Handle Return:** Wrap the resulting `JobHandle` and job data into a return structure (e.g., `JobHandleWithData<IMesherJob>`) and return it to the caller [S1, unofficial].

**Parameters:**
*   `Isolevel`: The density threshold for surface extraction. The default value is typically 0.5f, often constrained to the range [0, 1] in editor interfaces [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def schedule_marching_cubes_job(chunk_coordinate, voxel_world):
    """
    Schedules a Marching Cubes job for a specific chunk.
    
    Args:
        chunk_coordinate (int3): The (x, y, z) coordinate of the chunk.
        voxel_world (VoxelWorld): The world object containing voxel data and settings.
        
    Returns:
        JobHandleWithData: A handle to the scheduled job, or null if data is missing.
    """
    # Step 1: Retrieve chunk data
    voxel_data = voxel_world.chunk_store.get_voxel_data(chunk_coordinate)
    voxel_colors = voxel_world.chunk_store.get_voxel_colors(chunk_coordinate)
    
    if voxel_data is None or voxel_colors is None:
        return None

    # Step 2: Allocate atomic counter
    vertex_count_counter = NativeCounter(allocator=Allocator.TempJob)

    # Step 3: Calculate buffer sizes
    chunk_size = voxel_world.world_settings.chunk_size
    voxel_count = chunk_size.x * chunk_size.y * chunk_size.z
    max_length = 15 * voxel_count  # Max vertices per chunk
    
    # Allocate output buffers
    output_vertices = NativeArray(size=max_length, allocator=Allocator.TempJob)
    output_triangles = NativeArray(size=max_length, allocator=Allocator.TempJob)

    # Step 4: Initialize job struct
    job = MarchingCubesJob(
        VoxelData=voxel_data,
        VoxelColors=voxel_colors,
        Isolevel=0.5f,  # Default isovalue
        VertexCountCounter=vertex_count_counter,
        OutputVertices=output_vertices,
        OutputTriangles=output_triangles
    )

    # Step 5: Schedule the job
    job_handle = job.schedule()

    # Step 6: Return wrapped handle
    return JobHandleWithData(job_handle, job)
```

## Subtopics

### Multi-Process Ground Truth Generation
This variant orchestrates parallel processing of 3D models to generate ground truth data using multiple worker processes and a queue for synchronization [S2, unofficial]. It reads object names from a list, distributes them among worker processes based on an even distribution strategy, and processes intersections to generate ground truth inputs. Workers communicate via a queue, with the main process polling for results and checking worker exit codes to ensure completion [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the chunk, $O(N_x N_y N_z)$, as each voxel is processed independently.
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the temporary output buffers. The factor of 15 ensures capacity for the worst-case triangle count.
*   **Performance:** Highly scalable on GPU hardware due to the embarrassingly parallel nature of voxel processing. Performance is limited by memory bandwidth when reading voxel data and writing to the output buffers.
*   **Strengths:** Enables incremental updates (only re-meshing changed chunks), efficient memory usage (chunk-based), and easy integration with asynchronous rendering pipelines.
*   **Limitations:** Requires careful management of temporary memory allocations (e.g., `Allocator.TempJob` in Unity) to avoid leaks. The fixed-size buffer allocation can lead to significant memory overhead if the actual triangle count is much lower than the worst-case bound.
*   **When to use:** Ideal for dynamic voxel terrains or large-scale implicit surfaces where only a portion of the volume changes frequently.
*   **When not to use:** Not suitable for static, small volumes where the overhead of job scheduling and buffer management outweighs the benefits of parallelism.

## Background
The method is derived from implementations in voxel terrain systems, specifically the `Eldemarkki/Marching-Cubes-Terrain` project [S1, unofficial]. It leverages Unity's Job System and Burst Compiler to achieve high-performance parallel mesh generation. The approach aligns with general GPU acceleration techniques for Marching Cubes, utilizing atomic counters for dynamic output buffering [4082-gpu-marching-cubes-with-atomic-counter-emission].

## Sources
[S1] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S2] czq142857/NDC, https://github.com/czq142857/NDC, fetched 2026-10-26 (unofficial)
