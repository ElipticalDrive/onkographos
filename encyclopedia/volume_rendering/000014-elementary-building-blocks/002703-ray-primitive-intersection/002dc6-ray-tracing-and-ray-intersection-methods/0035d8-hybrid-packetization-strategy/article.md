# Hybrid Packetization Strategy

Also called: Hybrid Intersector, Hybrid Packetization, Hybrid Ray-Triangle Intersection

## Overview
A strategy to improve SIMD utilization by distributing work for a single ray to many threads when rays have high workload or when standard packetization is inefficient [S1, S2, unofficial]. It addresses the loss of parallel efficiency in sparse grids with many primitives per leaf, where standard warp-level packetization fails due to divergent traversal paths [S1, S2, unofficial]. It sets itself apart by dynamically switching between standard warp-level intersection and a shared-memory-based parallel processing mode based on workload thresholds [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: A warp of 32 rays, an acceleration structure (two-level grid) with leaf nodes containing lists of triangle primitives.
- Output: The closest intersection distance for each ray in the warp.

**Data Structures**
- **Shared Memory Buffer**: Stores indices for rays with high workload.
  - Fields: `ray_index` (uint32), `first_candidate_index` (uint32).
  - Size: Dynamic, up to 32 entries (one per thread in the warp).
- **Shared Distance Arrays**: Arrays to store intersection distances for intermediate reduction.
- **Thread Local Memory**: Stores the final closest intersection distance for the thread's assigned ray.

**Parameters**
- **High Workload Threshold**: 16 intersection candidates [S1, S2, unofficial].
- **Fallback Threshold**: 20 active rays with high workload [S1, S2, unofficial].
- **Warp Size**: 32 threads [S1, S2, unofficial].

**Steps**
1. **Check Uniformity**: Determine if all 32 rays in the warp have intersection candidates.
   - If yes, proceed with standard ray-triangle intersection [S1, S2, unofficial].
2. **Check High Workload Count**: If not all rays have candidates, count the number of rays with $\ge 16$ intersection candidates.
   - If count $> 20$, proceed with standard intersection routine (optimized for architectures like Fermi that support more Boolean operations) [S1, S2, unofficial].
   - If count $\le 20$ and $> 0$, proceed with hybrid packetization [S1, S2, unofficial].
3. **Buffer Insertion**: For each ray with high workload:
   - Use an atomic increment on a shared counter to get a buffer slot index [S1, S2, unofficial].
   - Store the ray's global index and the index of its first intersection candidate in the shared buffer [S1, S2, unofficial].
   - Note: Remaining primitive indices are assumed consecutive, so only the first is stored [S1, S2, unofficial].
4. **Parallel Processing**: Iterate through the buffer entries, taking two at a time.
   - Perform 32 intersection tests for a pair of rays in parallel using the warp's threads [S1, S2, unofficial].
   - Store intersection distances in shared memory arrays [S1, S2, unofficial].
5. **Reduction**: Perform a reduction on the shared distance arrays to find the minimum distance for each ray [S1, S2, unofficial].
6. **Record Result**: The thread responsible for tracing the ray records the closest intersection distance in thread local memory [S1, S2, unofficial].
7. **Remaining Tests**: Process remaining intersection tests one by one for all rays as usual after high workload rays are processed [S1, S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_packetization_intersect(warp_rays, grid, leaf_primitives):
    """
    warp_rays: List of 32 Ray objects
    grid: Two-level grid structure
    leaf_primitives: List of triangles in the current leaf node
    """
    # Step 1: Check if all rays have candidates
    all_have_candidates = all(ray.has_candidates() for ray in warp_rays)
    if all_have_candidates:
        return standard_ray_triangle_intersection(warp_rays, leaf_primitives)

    # Step 2: Count high workload rays
    high_workload_threshold = 16
    fallback_threshold = 20
    high_workload_rays = [ray for ray in warp_rays if ray.candidate_count() >= high_workload_threshold]
    num_high_workload = len(high_workload_rays)

    if num_high_workload > fallback_threshold:
        return standard_ray_triangle_intersection(warp_rays, leaf_primitives)

    if num_high_workload == 0:
        # No high workload rays, process remaining as usual
        return process_remaining_intersection_tests(warp_rays, leaf_primitives)

    # Step 3: Buffer Insertion
    # Shared memory buffer: array of (ray_index, first_candidate_index)
    # Shared counter for buffer insertion
    shared_buffer = [None] * 32
    shared_counter = 0
    
    for ray in high_workload_rays:
        # Atomic increment to get slot
        slot_index = atomic_add(shared_counter, 1)
        shared_buffer[slot_index] = (ray.global_index, ray.first_candidate_index)

    # Step 4 & 5: Parallel Processing and Reduction
    # Iterate buffer entries two at a time
    processed_rays = set()
    for i in range(0, num_high_workload, 2):
        if i + 1 < num_high_workload:
            ray1_idx, ray1_first = shared_buffer[i]
            ray2_idx, ray2_first = shared_buffer[i + 1]
            # Perform 32 intersection tests for the pair
            distances = perform_parallel_intersections(ray1_idx, ray1_first, ray2_idx, ray2_first, leaf_primitives)
            # Reduction on distances
            min_dist1 = reduce_min(distances[0])
            min_dist2 = reduce_min(distances[1])
            # Record results
            warp_rays[ray1_idx].closest_distance = min_dist1
            warp_rays[ray2_idx].closest_distance = min_dist2
            processed_rays.add(ray1_idx)
            processed_rays.add(ray2_idx)
        else:
            # Odd ray out
            ray_idx, ray_first = shared_buffer[i]
            distances = perform_parallel_intersections(ray_idx, ray_first, None, None, leaf_primitives)
            min_dist = reduce_min(distances[0])
            warp_rays[ray_idx].closest_distance = min_dist
            processed_rays.add(ray_idx)

    # Step 7: Remaining tests
    for ray in warp_rays:
        if ray.global_index not in processed_rays:
            ray.closest_distance = process_remaining_intersection_tests([ray], leaf_primitives)[0]

    return [ray.closest_distance for ray in warp_rays]
```

**Edge Cases and Pitfalls**
- **Precision**: Intersection distance calculations must handle floating-point precision issues, especially when comparing distances from different rays [G].
- **Boundaries**: Ensure buffer indices do not exceed shared memory limits [G].
- **Degenerate Input**: Handle cases where rays have no intersection candidates gracefully [G].

## Operating characteristics
- **Time Complexity**: Depends on the number of primitives per leaf and the number of high-workload rays. Standard packetization is $O(N)$ where $N$ is the number of primitives. Hybrid strategy adds overhead for buffer management and reduction but improves SIMD utilization for sparse grids [S1, S2, unofficial].
- **Memory Complexity**: Additional shared memory for the buffer and distance arrays. Buffer size is proportional to the number of high-workload rays (up to 32) [S1, S2, unofficial].
- **Measured Performance**: Benchmarked on GTX 280 (Fermi architecture). Performance improves for scenes with sparse grids and many primitives per leaf [S1, S2, unofficial].
- **Strengths**: Improves SIMD utilization in scenarios where standard packetization is inefficient due to divergent ray paths [S1, S2, unofficial].
- **Limitations**: Overhead from buffer management and reduction can negate benefits if too many rays have high workload [S1, S2, unofficial].
- **When to Use**: Sparse grids with many primitives per leaf, where standard packetization leads to poor SIMD utilization [S1, S2, unofficial].
- **When Not to Use**: Dense grids or scenes with low primitive counts per leaf, where standard packetization is more efficient [S1, S2, unofficial].

## Background
Originated in the paper "Two-Level Grids for Ray Tracing on GPUs" by J. Kalojanov, M. Billeter, and P. Slusallek [S1, S2, unofficial]. Developed to address inefficiencies in ray-triangle intersection on GPUs, particularly Fermi architecture, by leveraging shared memory and atomic operations to handle divergent ray workloads [S1, S2, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
