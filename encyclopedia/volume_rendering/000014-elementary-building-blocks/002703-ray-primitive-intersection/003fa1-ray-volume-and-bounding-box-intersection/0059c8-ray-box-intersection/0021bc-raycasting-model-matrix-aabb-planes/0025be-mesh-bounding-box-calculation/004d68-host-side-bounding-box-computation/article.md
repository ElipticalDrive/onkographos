# Host-side Bounding Box Computation

Also called: ComputeBoundHost, UnionBoundHost

## Overview
Computes the axis-aligned bounding box (AABB) for each triangle in a mesh on the CPU host, producing separate arrays for minimum and maximum coordinates [S1, unofficial]. It serves as a prerequisite step for GPU-based voxelization by providing the spatial extent of each primitive before ray-primitive intersection or volume rasterization [G]. It differs from [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md), which computes a single global AABB for the entire mesh, by maintaining per-triangle bounds [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Pointer to an array of `float` values representing vertex positions. The layout is contiguous triples (x, y, z), indexed by triangle indices [S1, unofficial].
- `triangles`: Pointer to an array of `int` indices. Each triangle consists of 3 indices into the `vertices` array [S1, unofficial].
- `N`: The number of triangles in the mesh.
- `n_cores`: The number of CPU threads to use for parallel execution.

**Outputs:**
- `bbox0`: Output array of `float` of size at least $3 \times N$. Stores the minimum coordinates (min_x, min_y, min_z) for each triangle [S1, unofficial].
- `bbox1`: Output array of `float` of size at least $3 \times N$. Stores the maximum coordinates (max_x, max_y, max_z) for each triangle [S1, unofficial].

**Data Structure Layout:**
The output arrays use a Structure of Arrays (SoA) layout optimized for component-wise access [S1, unofficial]. For triangle index $i$ ($0 \le i < N$):
- `bbox0[i]` stores $\min_x$
- `bbox0[i + N]` stores $\min_y$
- `bbox0[i + 2N]` stores $\min_z$
- `bbox1[i]` stores $\max_x$
- `bbox1[i + N]` stores $\max_y$
- `bbox1[i + 2N]` stores $\max_z$ [S1, unofficial].

**Algorithm Steps:**
1. **Dispatch:** Check if `n_cores > 1` [S1, unofficial].
   - If true, call `ComputeBoundParallel` with `begin=0` and `end=N` [S1, unofficial].
   - If false, call `ComputeBoundSerial` with `begin=0` and `end=N` [S1, unofficial].
2. **Serial Execution (`ComputeBoundSerial`):**
   - Iterate `tri_idx` from `begin` to `end` [S1, unofficial].
   - Initialize `bbox0` and `bbox1` entries for `tri_idx` to extreme values:
     - For $c \in \{0, 1, 2\}$:
       - $c\_idx = tri\_idx + c \times N$
       - $bbox0[c\_idx] = \text{FLT\_MAX}$
       - $bbox1[c\_idx] = -\text{FLT\_MAX}$ [S1, unofficial].
   - For each vertex component $c \in \{0, 1, 2\}$ of the triangle:
     - Retrieve vertex index: $idx = triangles[tri\_idx + c \times N]$ [S1, unofficial].
     - Retrieve vertex pointer: $ptr = &vertices[3 \times idx]$ [S1, unofficial].
     - $val = ptr[c]$
     - $c\_idx = tri\_idx + c \times N$
     - Update bounds:
       - If $val < bbox0[c\_idx]$, set $bbox0[c\_idx] = val$ [S1, unofficial].
       - If $val > bbox1[c\_idx]$, set $bbox1[c\_idx] = val$ [S1, unofficial].
3. **Parallel Execution (`ComputeBoundParallel`):**
   - Calculate `thread_size = N / n_threads` (default $n\_threads = 4$) [S1, unofficial].
   - Create `n_threads - 1` threads, each processing a chunk of `thread_size` triangles starting from `begin` [S1, unofficial].
   - The last thread processes the remaining triangles from `begin + (n_threads - 1) \times thread_size` to `end` [S1, unofficial].
   - Each thread executes the logic of `ComputeBoundSerial` for its assigned range [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading

def compute_bound_host(vertices, triangles, N, n_cores=1):
    # Initialize output arrays of size 3*N
    bbox0 = [float('inf')] * (3 * N)
    bbox1 = [float('-inf')] * (3 * N)

    if n_cores > 1:
        compute_bound_parallel(vertices, triangles, N, bbox0, bbox1, n_cores)
    else:
        compute_bound_serial(vertices, triangles, N, 0, N, bbox0, bbox1)

    return bbox0, bbox1

def compute_bound_serial(vertices, triangles, N, begin, end, bbox0, bbox1):
    for tri_idx in range(begin, end):
        # Initialize bounds for this triangle
        for c in range(3):
            c_idx = tri_idx + c * N
            bbox0[c_idx] = float('inf')
            bbox1[c_idx] = float('-inf')

        # Process 3 vertices of the triangle
        for c in range(3):
            # Retrieve vertex index and pointer per evidence
            vert_idx = triangles[tri_idx + c * N]
            base_vert_idx = vert_idx * 3

            # Update bounds for component c
            val = vertices[base_vert_idx + c]
            c_idx = tri_idx + c * N

            if val < bbox0[c_idx]:
                bbox0[c_idx] = val
            if val > bbox1[c_idx]:
                bbox1[c_idx] = val

def compute_bound_parallel(vertices, triangles, N, bbox0, bbox1, n_cores):
    n_threads = n_cores
    thread_size = N // n_threads
    threads = []

    # Create n_threads - 1 threads
    for t in range(n_threads - 1):
        start_idx = t * thread_size
        end_idx = (t + 1) * thread_size
        thread = threading.Thread(
            target=compute_bound_serial,
            args=(vertices, triangles, N, start_idx, end_idx, bbox0, bbox1)
        )
        threads.append(thread)
        thread.start()

    # Last thread processes the remainder
    last_start = (n_threads - 1) * thread_size
    last_end = N
    thread = threading.Thread(
        target=compute_bound_serial,
        args=(vertices, triangles, N, last_start, last_end, bbox0, bbox1)
    )
    threads.append(thread)
    thread.start()

    for thread in threads:
        thread.join()
```

**Parameters:**
- `vertices`: Array of floats. Layout is contiguous triples (x,y,z) per vertex [S1, unofficial].
- `triangles`: Array of integers. Indices into `vertices` [S1, unofficial].
- `N`: Integer, number of triangles.
- `n_cores`: Integer, number of CPU threads. Default 4 in source implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `N < 1`, `UnionBoundHost` returns immediately without processing, avoiding division by zero or invalid memory access in subsequent logic if called directly, though `ComputeBoundHost` does not explicitly check `N < 1` before calling parallel/serial versions [S1, unofficial].
- `ComputeBoundSerial` loops `tri_idx` from `begin` to `end`, so if `begin >= end`, it does nothing, which safely handles the last thread in parallel execution when `N` is not evenly divisible by `n_cores` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of triangles. Each triangle is processed exactly once [G].
- **Memory Complexity:** $O(N)$ for the output arrays `bbox0` and `bbox1` [G].
- **Performance:** Parallel execution scales with `n_cores`. The source implementation uses a simple chunking strategy [S1, unofficial]. No specific hardware benchmarks are provided in the sources [S1, unofficial].
- **Strengths:** Simple, deterministic, and easy to parallelize. Provides per-triangle bounds necessary for efficient voxelization culling [S1, unofficial].
- **Limitations:** Does not compute a global mesh bound. Requires host-side memory allocation for output arrays [S1, unofficial].

## Background
This method is part of the host-side preprocessing pipeline for GPU-based voxelization, specifically from the `hpsoar/voxelizer` project [S1, unofficial]. It prepares the geometric data for the `voxelize_kernel` which runs on the GPU device [S1, unofficial]. The method is distinct from global bounding box computation, focusing on per-primitive bounds to enable efficient spatial partitioning during rasterization [S1, unofficial].

## Sources
[S1] hpsoar/voxelizer, https://github.com/hpsoar/voxelizer, fetched 2026-10-26 (unofficial)
