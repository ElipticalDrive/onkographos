# Hybrid Packetization Strategy

Also called: Hybrid Intersector, Hybrid Packetization, Hybrid Ray-Triangle Intersection

## Overview
A strategy to improve SIMD utilization by distributing work for a single ray to many threads when rays have high workload or when standard packetization is inefficient [S1, unofficial]. It solves the problem of low occupancy in warp-level ray-triangle intersection on sparse grids with many primitives per leaf [S1, unofficial]. It sets itself apart from standard packetization by dynamically switching between warp-level parallelism and intra-ray parallelism based on workload thresholds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A warp of 32 rays, a list of intersection candidates (primitive indices) for each ray.
*   Output: The closest intersection distance for each ray in the warp.

**Data Structures:**
*   **Shared Memory Buffer:** Stores ray indices and the starting index of their intersection candidate lists.
    *   Layout: Array of structs or interleaved integers.
    *   Fields: `ray_id` (uint32), `candidate_start_index` (uint32).
    *   Size: Dynamic, up to 32 entries.
*   **Shared Memory Counter:** Atomic integer tracking the number of high-workload rays inserted into the buffer.
*   **Thread Local Memory:** Stores the current closest intersection distance for the active thread's ray.

**Parameters:**
*   Warp size: 32 threads [S1, unofficial].
*   High workload threshold: 16 intersection candidates per ray [S1, unofficial].
*   Fallback threshold: 20 active rays with high workload [S1, unofficial].

**Steps:**
1.  **Check Uniformity:** Determine if all 32 rays in the warp have intersection candidates.
    *   If yes, proceed with standard ray-triangle intersection [S1, unofficial].
2.  **Check High Workload Count:** If not all rays have candidates, count the number of rays with $\ge 16$ intersection candidates.
    *   If count $> 20$, proceed with standard intersection routine (optimized for hardware Boolean operations, e.g., Fermi) [S1, unofficial].
    *   If count $\le 20$ and $> 0$, proceed with hybrid strategy [S1, unofficial].
3.  **Buffer Insertion:** For each ray with high workload:
    *   Atomically increment a shared counter to get a buffer slot index.
    *   Store the ray's index and the index of its first intersection candidate in the shared memory buffer [S1, unofficial].
    *   Note: Remaining primitive indices are assumed consecutive, so only the start index is stored [S1, unofficial].
4.  **Parallel Processing Loop:** Iterate through the buffer entries in pairs (two rays at a time).
    *   For each pair, perform 32 intersection tests in parallel across the warp [S1, unofficial].
    *   This involves each thread computing intersections for the two rays' candidate lists.
5.  **Reduction:** Perform a reduction on the arrays of intersection distances stored in shared memory to find the minimum distance for each of the two rays [S1, unofficial].
6.  **Result Recording:** The thread responsible for tracing each ray records the closest intersection (if any) in thread local memory [S1, unofficial].
7.  **Remaining Rays:** After processing all high-workload rays, perform remaining intersection tests one by one for all other rays in the warp as usual [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_packetization_intersect(warp_rays, candidates_per_ray):
    """
    warp_rays: list of 32 ray objects
    candidates_per_ray: list of 32 lists of primitive indices
    """
    # Step 1: Check if all rays have candidates
    all_have_candidates = all(len(c) > 0 for c in candidates_per_ray)
    if all_have_candidates:
        return standard_ray_triangle_intersect(warp_rays, candidates_per_ray)

    # Step 2: Count high workload rays
    high_workload_threshold = 16
    high_workload_rays = []
    for i, c in enumerate(candidates_per_ray):
        if len(c) >= high_workload_threshold:
            high_workload_rays.append(i)

    fallback_threshold = 20
    if len(high_workload_rays) > fallback_threshold:
        return standard_ray_triangle_intersect(warp_rays, candidates_per_ray)

    # Step 3: Buffer insertion
    # Shared memory structures assumed to be initialized
    shared_buffer = []
    shared_counter = 0
    
    # Atomic increment simulation for shared memory
    for ray_idx in high_workload_rays:
        slot = atomic_add(shared_counter, 1)
        shared_buffer[slot] = {
            'ray_id': ray_idx,
            'start_index': 0 # Assuming consecutive, store start index
        }
    
    num_high_workload = len(high_workload_rays)
    
    # Step 4 & 5: Process in pairs
    results = {i: infinity for i in range(32)}
    
    for i in range(0, num_high_workload, 2):
        if i + 1 < num_high_workload:
            ray1_idx = high_workload_rays[i]
            ray2_idx = high_workload_rays[i+1]
            # Perform 32 intersection tests for the pair
            dist1, dist2 = parallel_intersect_pair(ray1_idx, ray2_idx, candidates_per_ray)
            # Reduction to find closest
            results[ray1_idx] = dist1
            results[ray2_idx] = dist2
        else:
            # Handle odd ray out if necessary, though logic implies pairs
            ray_idx = high_workload_rays[i]
            dist = parallel_intersect_single(ray_idx, candidates_per_ray)
            results[ray_idx] = dist

    # Step 7: Process remaining rays
    for i in range(32):
        if i not in high_workload_rays:
            results[i] = standard_single_ray_intersect(warp_rays[i], candidates_per_ray[i])

    return results
```

**Edge Cases and Pitfalls:**
*   Precision: Standard ray-triangle intersection precision issues apply.
*   Boundaries: The assumption that primitive indices are consecutive is critical for the buffer size optimization [S1, unofficial]. If not consecutive, the buffer must store full lists, increasing memory pressure.
*   Degenerate Input: If no rays have candidates, standard logic handles it. If exactly 20 rays have high workload, the fallback is used, which may be suboptimal but safe [S1, unofficial].

## Subtopics

### Standard Packetization
The baseline method where each thread handles one ray. Less efficient for sparse grids with high primitive density per leaf. [Standard Packetization](5617-ray-tracing-and-ray-intersection-methods.md)

### Warp-Level Intersection
The fallback method used when high workload rays exceed the threshold, leveraging hardware Boolean operations. [Warp-Level Intersection](5617-ray-tracing-and-ray-intersection-methods.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of intersection candidates. The hybrid strategy aims to balance the overhead of buffer management with the gain from parallelizing high-workload rays.
*   **Memory Complexity:** Requires shared memory for the buffer and counter. The buffer size is limited to the number of high-workload rays (max 20 in the hybrid path).
*   **Measured Performance:** Tested on GTX 280 (Fermi architecture). Performance improves on sparse grids with many primitives per leaf [S1, unofficial].
*   **Strengths:** Improves SIMD utilization when rays have uneven workloads.
*   **Limitations:** Overhead from atomic operations and buffer management can negate benefits if not tuned correctly. The thresholds (16, 20) are specific to the tested architecture and scene characteristics [S1, unofficial].
*   **When to use:** When rendering sparse grids with high primitive density per leaf and ray coherence is low [S1, unofficial].
*   **When not to use:** When all rays have uniform low workload or when the number of high-workload rays exceeds the fallback threshold [S1, unofficial].

## Background
Originated in the paper "Two-Level Grids for Ray Tracing on GPUs" by J. Kalojanov, M. Billeter, and P. Slusallek [S1, unofficial]. The method was developed to address the inefficiencies of standard packetization on GPU architectures like Fermi, which have specific hardware capabilities for warp-level Boolean operations [S1, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
