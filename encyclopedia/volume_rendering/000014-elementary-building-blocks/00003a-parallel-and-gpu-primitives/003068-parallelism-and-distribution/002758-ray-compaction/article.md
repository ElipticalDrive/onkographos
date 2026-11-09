# Ray Compaction


## Overview
Ray Compaction is a parallel primitive used in volume rendering to filter a list of active rays, removing those that have terminated (e.g., reached the far plane or accumulated sufficient opacity) and compacting the remaining "alive" rays into a dense, contiguous buffer. It solves the problem of sparse ray lists, ensuring that subsequent processing stages (such as ray marching or feature sampling) operate only on valid rays with predictable memory access patterns. It sets itself apart from siblings like [Ray Buffer Reordering](4073-ray-buffer-reordering.md) by often serving as the foundational atomic-indexing step, whereas [In-Kernel Stream Compaction (InK-Compact)](6991-in-kernel-stream-compaction-ink-compact.md) offers a more complex, single-kernel alternative that avoids separate prefix-sum launches.

## How it works
**Inputs and Outputs:**
- Input: `rays_alive_old` (array of ray state/position data), `rays_t_old` (array of ray termination flags or distances), `N` (number of input rays).
- Output: `rays_alive` (compacted ray data), `rays_t` (compacted flags), `alive_count` (total number of surviving rays).

**Data Structures:**
- `rays_alive_old`: Array of size $N$. Layout depends on ray representation (e.g., origin, direction, current $t$).
- `rays_t_old`: Array of size $N$. Typically stores a scalar termination value. Rays are considered terminated if `rays_t_old[n] < 0` or exceeds a threshold.
- `rays_alive`: Array of size $N$ (worst case).
- `rays_t`: Array of size $N$.
- `alive_counter`: Global atomic integer, initialized to 0.

**Steps:**
1. Initialize `alive_counter` to 0.
2. Launch a GPU kernel with one thread per input ray $n$ ($0 \le n < N$).
3. Each thread $n$ checks if its ray is alive: `if rays_t_old[n] >= 0`.
4. If alive:
   a. Atomically increment `alive_counter` to obtain a unique new index: `index = atomicAdd(&alive_counter, 1)`.
   b. Copy the ray data: `rays_alive[index] = rays_alive_old[n]`.
   c. Copy the termination flag: `rays_t[index] = rays_t_old[n]`.
5. After kernel completion, read `alive_counter` to determine the number of valid rays in the output buffers.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_compaction_kernel(rays_alive_old, rays_t_old, rays_alive, rays_t, alive_counter):
    n = get_thread_id()
    if n < len(rays_t_old):
        # Check if ray is still active (not terminated)
        # Note: Termination condition depends on specific implementation; 
        # sources indicate rays_t_old[n] < 0 marks termination [S1, unofficial].
        if rays_t_old[n] >= 0:
            # Atomically assign a new compacted index
            index = atomic_add(alive_counter, 1)
            
            # Copy ray state data
            rays_alive[index] = rays_alive_old[n]
            
            # Copy termination flag
            rays_t[index] = rays_t_old[n]

def compact_rays(rays_alive_old, rays_t_old, N):
    # Initialize output buffers
    rays_alive = [None] * N
    rays_t = [None] * N
    
    # Initialize atomic counter
    alive_counter = 0
    
    # Launch kernel
    # In practice, this is a CUDA/HIP kernel launch
    # grid = (N + block_size - 1) // block_size
    # ray_compaction_kernel<<<grid, block_size>>>(rays_alive_old, rays_t_old, rays_alive, rays_t, alive_counter)
    
    # Wait for completion and retrieve count
    # alive_count = get_atomic_value(alive_counter)
    
    return rays_alive, rays_t, alive_count
```

**Parameters:**
- `N`: Number of input rays. Typical values depend on resolution (e.g., $W \times H$ for primary rays).
- Termination Threshold: Defined by the application logic (e.g., `rays_t_old[n] < 0`).

**Edge Cases and Pitfalls:**
- **Precision:** Atomic operations on integers are precise. However, if `rays_t` stores floating-point termination distances, ensure the comparison logic (`>= 0`) aligns with the termination definition.
- **Boundaries:** The output buffers must be sized to $N$ to handle the case where all rays survive.
- **Degenerate Input:** If all rays terminate, `alive_count` is 0, and subsequent steps must handle empty buffers.

## Subtopics

### [Ray Buffer Reordering](4073-ray-buffer-reordering.md)
Uses double-buffering and atomic counters to reorder rays without prior sorting. Choose when maintaining strict input-output order is required without global reductions.

### [In-Kernel Stream Compaction (InK-Compact)](6991-in-kernel-stream-compaction-ink-compact.md)
Computes output indices entirely within a single kernel using bit-decomposition scans. Choose when minimizing kernel launch overhead and synchronization is critical.

## Operating characteristics
- **Time Complexity:** $O(N)$ work, where $N$ is the number of input rays. The atomic add operation is the bottleneck, potentially causing contention.
- **Memory Complexity:** $O(N)$ for input and output buffers.
- **Performance:** Performance depends heavily on the ratio of alive rays. High contention on `alive_counter` can reduce throughput. Coalesced memory access for `rays_alive_old` and `rays_t_old` is crucial.
- **Strengths:** Simple to implement; low memory overhead compared to sorting-based methods.
- **Limitations:** Atomic contention can limit scalability on high-core-count GPUs. Does not preserve original order if multiple threads write to the same index (though atomic add ensures unique indices, the order of assignment is non-deterministic relative to input order unless additional sorting is applied).
- **When to Use:** When ray lists are moderately sparse and simplicity is preferred.
- **When Not to Use:** When strict input order preservation is required without additional sorting steps, or when extreme sparsity makes sorting-based approaches more efficient.

## Background
Originating from parallel volume rendering implementations on GPUs, Ray Compaction is a standard technique for managing active ray lists. It is commonly found in CUDA-based volume renderers where efficient memory usage and processing of only active rays are critical. The method leverages atomic operations, a fundamental GPU primitive, to achieve thread-safe compaction.

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
