# Parallel CPU Voxelization
Also called: Parallel Voxelization, ThreadPoolExecutor Voxelization

## Overview
A multi-threaded CPU-based voxelization method used as a fallback when no CUDA device is found or when forced via the `-cpu` flag [S1, unofficial]. It utilizes Python's `ThreadPoolExecutor` to parallelize the voxelization process across multiple CPU cores [S2, unofficial]. It is chosen when GPU acceleration is unavailable or when the overhead of GPU invocation is not worth it for very small models [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh file (e.g., `.ply`) or point cloud data, and a target resolution $S$ (e.g., 256) [S1, unofficial].
*   **Output:** A voxel model file (e.g., `.vox` or `.obj`) [S1, unofficial].

**Data Structures:**
The method partitions the voxel grid into sub-tasks. Each worker item contains:
*   Voxel bounds: $(x_{start}, x_{end}, y_{start}, y_{end}, z_{start}, z_{end})$ [S2, unofficial].
*   Indices: $(i, j, k)$ corresponding to the grid coordinates [S2, unofficial].

**Steps:**
1.  **Initialization:** Check if a CUDA device is available (`cuda_ok`). If `cuda_ok` is false or the `-cpu` flag is set, execute the CPU voxelization path [S1, unofficial].
2.  **Task Generation:** Generate a list of worker items. Each item defines a bounding box $(x_{start}, x_{end}, y_{start}, y_{end}, z_{start}, z_{end})$ and associated indices $(i, j, k)$ [S2, unofficial].
3.  **Parallel Execution:** Submit tasks to a `ThreadPoolExecutor`. The number of workers is set to `os.cpu_count()` to utilize all available CPU cores [S2, unofficial].
4.  **Local Processing:** For each worker, process points within the specified range. This involves filtering points by $x, y, z$ ranges and computing local averages or occupancy [S2, unofficial].
    *   Get points in $x$ range: `points_in_x_range` [S2, unofficial].
    *   Get points in $y$ range: `points_in_y_range` [S2, unofficial].
    *   Get points in $z$ range: `points_in_z_range` [S2, unofficial].
    *   Compute intersection or average based on these filtered sets [S2, unofficial].
5.  **Aggregation:** Combine results from all workers into the final voxel grid [S2, unofficial].

**Parameters:**
*   `max_workers`: Set to `os.cpu_count()` [S2, unofficial].
*   `-solid`: Experimental flag to use solid voxelization instead of face voxelization. Requires a watertight input mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
from concurrent.futures import ThreadPoolExecutor

def parallel_cpu_voxelization(mesh, resolution, force_cpu=False):
    # Step 1: Check CUDA availability
    cuda_ok = check_cuda_availability() # [G] Placeholder for CUDA detection logic
    
    if not cuda_ok or force_cpu:
        # Step 2: Generate worker items
        # Each item contains voxel bounds and indices
        worker_items = generate_worker_items(resolution)
        
        # Step 3: Define worker function
        def process_worker(item):
            x_start, x_end = item['x_start'], item['x_end']
            y_start, y_end = item['y_start'], item['y_end']
            z_start, z_end = item['z_start'], item['z_end']
            
            # Filter points based on bounds
            points_x = get_points_in_range(mesh.points, 'x', x_start, x_end)
            points_y = get_points_in_range(mesh.points, 'y', y_start, y_end)
            points_z = get_points_in_range(mesh.points, 'z', z_start, z_end)
            
            # Compute local average or occupancy
            # [S2, unofficial] Logic for local_average_faster implied
            result = compute_local_average(points_x, points_y, points_z)
            return result

        # Step 4: Execute with ThreadPoolExecutor
        max_workers = os.cpu_count()
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            results = list(executor.map(process_worker, worker_items))
        
        # Step 5: Aggregate results
        final_voxels = aggregate_results(results)
        return final_voxels
    else:
        raise RuntimeError("CUDA device available; use GPU path instead.")
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and input primitives, parallelized across $N$ cores where $N = \text{os.cpu_count()}$ [S2, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid size and the number of input points/primitives [S2, unofficial].
*   **Strengths:** Works on any CPU; useful for small models where GPU overhead is significant [S1, unofficial].
*   **Limitations:** Slower than GPU-based methods for large models; limited by CPU core count and memory bandwidth [S1, unofficial].
*   **When to use:** When CUDA is unavailable, incompatible, or for small models [S1, unofficial].

## Background
This method is part of the `cuda_voxelizer` tool suite by Forceflow [S1, unofficial]. It serves as a CPU fallback for the primary CUDA-based voxelization engine [S1, unofficial]. The parallelization strategy using `ThreadPoolExecutor` is documented in the `Point-Cloud-Voxelizer` repository by MasonAndrewHarrison [S2, unofficial].

## Sources
[S1] cuda_voxelizer, https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
[S2] Point-Cloud-Voxelizer, https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer, fetched 2026-10-27 (unofficial)
