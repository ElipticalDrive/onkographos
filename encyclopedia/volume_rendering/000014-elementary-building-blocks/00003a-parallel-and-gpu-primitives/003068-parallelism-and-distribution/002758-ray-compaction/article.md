# Ray Compaction


## Overview
Ray Compaction is a parallel primitive used in volume rendering to filter a list of active rays, removing those that have terminated (e.g., reached the volume boundary or accumulated sufficient opacity) and compacting the remaining "alive" rays into a dense, contiguous buffer. It solves the problem of sparse ray lists causing inefficient memory access and underutilized GPU occupancy in subsequent ray-marching steps. It sets itself apart from siblings like [Parallel Ray Casting with Segment Lists](2565-parallel-ray-casting-with-segment-lists.md) by operating directly on per-ray state arrays using atomic counters rather than hierarchical segment structures, and from [Ray Buffer Reordering](4073-ray-buffer-reordering.md) by focusing on the initial compaction logic rather than the double-buffering swap mechanism.

## How it works
**Inputs and Outputs:**
- Input: `rays_alive_old` (array of ray state structures/objects), `rays_t_old` (array of ray traversal parameters, e.g., distance traveled or current t-value).
- Output: `rays_alive` (compact array of alive rays), `rays_t` (compact array of corresponding t-values).
- Auxiliary: `alive_counter` (global atomic integer, initialized to 0).

**Data Structures:**
- `rays_alive_old`: Array of size $N$ (total rays). Each element contains ray metadata (origin, direction, current opacity, etc.).
- `rays_t_old`: Array of size $N$ of floats. A value $< 0$ indicates a terminated ray.
- `rays_alive`: Array of size $N$ (pre-allocated to worst-case size).
- `rays_t`: Array of size $N$ of floats.

**Steps:**
1. Initialize `alive_counter` to 0.
2. Launch a GPU kernel with one thread per ray $n$ ($0 \le n < N$).
3. Each thread $n$ checks if `rays_t_old[n] >= 0`.
4. If the ray is alive:
   a. Atomically increment `alive_counter` to obtain a unique new index `idx`.
   b. Write `rays_alive_old[n]` to `rays_alive[idx]`.
   c. Write `rays_t_old[n]` to `rays_t[idx]`.
5. If the ray is terminated, do nothing.
6. After the kernel, the number of alive rays is `alive_counter`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel launch: grid(N), block(256)
def compact_rays_kernel(rays_alive_old, rays_t_old, rays_alive, rays_t, alive_counter):
    n = get_global_thread_id()
    if n < len(rays_t_old):
        # Check if ray is still active
        if rays_t_old[n] >= 0:
            # Get unique index for this alive ray
            idx = atomic_add(alive_counter, 1)
            # Compact data into new buffers
            rays_alive[idx] = rays_alive_old[n]
            rays_t[idx] = rays_t_old[n]
```

**Parameters:**
- Block size: Typically 256 or 512 threads per block [S1, unofficial].
- `alive_counter`: Must be a global atomic variable.

**Edge Cases and Pitfalls:**
- **Precision:** `rays_t_old` comparison with 0 must handle floating-point noise. If termination is defined by opacity accumulation, the threshold check should be explicit.
- **Boundaries:** The output arrays `rays_alive` and `rays_t` must be pre-allocated to size $N$ to handle the worst case where all rays survive.
- **Atomic Contention:** High contention on `alive_counter` can reduce performance if the ratio of alive rays is high.

## Subtopics

### [Ray Buffer Reordering](4073-ray-buffer-reordering.md)
Uses double-buffering with atomic counters to maintain dense lists across iterations.

### [In-Kernel Stream Compaction (InK-Compact)](6991-in-kernel-stream-compaction-ink-compact.md)
Computes output indices within a single kernel to avoid separate prefix-sum and scatter launches.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of input rays. Each ray is processed once.
- **Memory Complexity:** $O(N)$ for input and output buffers.
- **Performance:** Performance is heavily dependent on the ratio of alive rays. High atomic contention on `alive_counter` can become a bottleneck.
- **Strengths:** Simple implementation, low overhead for small to medium ray counts, no sorting required.
- **Limitations:** Not suitable for very large $N$ where atomic contention is severe; requires pre-allocation of full-size output buffers.
- **When to use:** When the number of rays is moderate and the termination rate is variable but not extremely high.

## Background
Originated in CUDA-based volume rendering implementations to handle dynamic ray lists efficiently. The method is described in unofficial sources from the megvii-research/cv-master-ex repository [S1, unofficial].

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
