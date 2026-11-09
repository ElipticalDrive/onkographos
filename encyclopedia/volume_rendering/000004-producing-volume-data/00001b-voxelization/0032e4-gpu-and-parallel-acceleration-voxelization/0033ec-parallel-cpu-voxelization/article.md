# Parallel CPU Voxelization

Also called: Parallel Voxelization, ThreadPoolExecutor Voxelization

## Overview
Parallel CPU Voxelization is a multi-threaded method for converting polygonal meshes or point clouds into a discrete volumetric representation, serving as a fallback when GPU acceleration (CUDA) is unavailable or explicitly disabled [S1, unofficial]. It differs from sibling methods by leveraging Python's `ThreadPoolExecutor` to distribute voxel computation across available CPU cores, contrasting with the single-threaded or GPU-specific approaches of its alternatives [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A geometric source (mesh or point cloud) and a target resolution $(N_x, N_y, N_z)$ defining the voxel grid dimensions [S1, unofficial][S2, unofficial].
*   **Output:** A voxelized representation, which may be stored as a binary file (e.g., `.vox`), a Morton-ordered binary grid, or a mesh format (e.g., `.obj`) if solid voxelization is requested [S1, unofficial].

**Data Structures:**
The method partitions the voxel grid into sub-tasks. Each worker item contains:
*   `x_start`, `x_end`: The inclusive range of x-indices for this worker.
*   `y_start`, `y_end`: The inclusive range of y-indices.
*   `z_start`, `z_end`: The inclusive range of z-indices.
*   `i`, `j`, `k`: Indices associated with the worker's position in the task queue [S2, unofficial].

**Steps:**
1.  **Initialization:** Check for CUDA availability. If `cuda_ok` is false or the `-cpu` flag is set, proceed with the CPU path [S1, unofficial].
2.  **Task Generation:** Generate a list of worker items. Each item defines a bounding box $(x_{start}, x_{end}, y_{start}, y_{end}, z_{start}, z_{end})$ within the global grid [S2, unofficial].
3.  **Parallel Execution:** Submit tasks to a `ThreadPoolExecutor` with `max_workers` set to `os.cpu_count()` to utilize all available cores [S2, unofficial].
4.  **Voxel Computation:** For each worker, iterate through the specified voxel bounds. For each voxel, determine if it intersects with the input geometry.
    *   *Note:* The specific intersection logic (e.g., ray casting or plane projection) is implemented in the underlying C++ logic (`cpu_voxelizer.h`) called by the Python wrapper [S1, unofficial].
5.  **Aggregation:** Collect results from all workers and write the final voxel data to the specified output format [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
from concurrent.futures import ThreadPoolExecutor

def parallel_cpu_voxelization(mesh, resolution, output_path, force_cpu=True):
    """
    Voxelize a mesh using parallel CPU threads.
    
    Args:
        mesh: Input geometric data (vertices, faces).
        resolution: Tuple (Nx, Ny, Nz) defining grid size.
        output_path: Path for the output file.
        force_cpu: Boolean flag to force CPU execution.
    """
    # Step 1: Check CUDA availability (simulated)
    cuda_ok = check_cuda_availability()
    
    # Step 2: Decide execution path
    if not cuda_ok or force_cpu:
        # Generate worker tasks
        # The grid is partitioned into chunks. 
        # Each chunk is defined by x_start, x_end, y_start, y_end, z_start, z_end.
        worker_items = generate_worker_items(resolution)
        
        # Step 3: Execute with ThreadPoolExecutor
        max_workers = os.cpu_count()
        
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # Submit tasks to the executor
            # The actual voxelization logic is handled by the underlying C++ module
            futures = [
                executor.submit(process_voxel_chunk, mesh, item) 
                for item in worker_items
            ]
            
            # Wait for completion and aggregate results
            results = [f.result() for f in futures]
            
        # Step 4: Write output
        write_voxel_data(results, output_path)
    else:
        # Fallback to GPU if available and not forced
        gpu_voxelization(mesh, resolution, output_path)

def process_voxel_chunk(mesh, worker_item):
    """
    Process a specific chunk of the voxel grid.
    This function is executed by each worker thread.
    """
    x_start = worker_item['x_start']
    x_end = worker_item['x_end']
    y_start = worker_item['y_start']
    y_end = worker_item['y_end']
    z_start = worker_item['z_start']
    z_end = worker_item['z_end']
    
    # Initialize local voxel grid for this chunk
    local_voxels = initialize_voxel_grid(x_end - x_start + 1, 
                                        y_end - y_start + 1, 
                                        z_end - z_start + 1)
    
    # Iterate over voxels in this chunk
    for x in range(x_start, x_end + 1):
        for y in range(y_start, y_end + 1):
            for z in range(z_start, z_end + 1):
                # Determine if voxel (x,y,z) intersects with mesh
                # This calls the underlying C++ logic (cpu_voxelizer.h)
                is_filled = check_voxel_intersection(mesh, x, y, z)
                if is_filled:
                    local_voxels[x - x_start, y - y_start, z - z_start] = 1
                    
    return local_voxels
```

**Parameters:**
*   `max_workers`: Set to `os.cpu_count()` to utilize all available CPU cores [S2, unofficial].
*   `force_cpu`: Boolean flag to force CPU execution even if CUDA is available [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Small Models:** For very small models, the overhead of CPU thread creation and synchronization may make this method slower than GPU execution [S1, unofficial].
*   **Watertight Meshes:** Solid voxelization requires a watertight input mesh; non-watertight meshes may produce incorrect internal voxel values [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and triangles, but with significant constant overhead due to thread management and Python interpreter overhead [S2, unofficial].
*   **Memory Complexity:** Linear with respect to the voxel grid size $(N_x \times N_y \times N_z)$ [S1, unofficial].
*   **Performance:** Performance scales with the number of CPU cores but is generally slower than GPU-based methods for large resolutions due to memory bandwidth and parallelism limitations [S1, unofficial].
*   **Strengths:** No CUDA hardware required; suitable for environments where GPU access is restricted or unavailable [S1, unofficial].
*   **Limitations:** Slower than GPU methods; Python overhead can be significant for fine-grained tasks [S2, unofficial].

## Background
Parallel CPU Voxelization is part of the `cuda_voxelizer` project by Forceflow, which primarily focuses on GPU acceleration but includes a CPU fallback for compatibility [S1, unofficial]. It is also referenced in the `Point-Cloud-Voxelizer` project, which uses Python's `ThreadPoolExecutor` for parallel processing [S2, unofficial].

## Sources
[S1] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
[S2] MasonAndrewHarrison/Point-Cloud-Voxelizer, https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer, fetched 2026-10-27 (unofficial)
