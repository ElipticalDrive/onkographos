# Hybrid Packetization Strategy

Also called: Hybrid Intersector, Hybrid Packetization, Hybrid Packetization for Triangle Intersection, Hybrid Ray-Triangle Intersection

## Overview
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when standard packetization is inefficient due to sparse acceleration structures with high primitive density per leaf [S1, S2, S3, unofficial]. It addresses the problem of lost coherence in warp-level computations, where some rays traverse full cells while others pass through empty ones, by dynamically distributing the workload of a single ray across multiple threads [S1, S2, S3, unofficial]. Unlike standard packetization, which processes multiple rays in parallel, this method exploits the property that two-level grids contain many primitives per leaf by splitting a single high-workload ray's intersection tests among threads, thereby maintaining parallel efficiency [S1, S2, S3, unofficial].

## How it works
The method operates within a GPU warp (a SIMD unit of 32 threads) during the ray-triangle intersection phase of ray traversal [S1, S2, S3, unofficial]. It dynamically selects between standard warp-level intersection and a parallel processing mode based on the workload distribution among the rays in the warp [S2, S3, unofficial].

### Inputs and Outputs
*   **Inputs**: A set of rays forming a warp (typically 32 rays); an acceleration structure (specifically a two-level grid) containing triangle primitives; for each ray, a list of intersection candidates (primitive indices) [S1, S2, S3, unofficial].
*   **Outputs**: The closest intersection distance and primitive index for each ray in the warp, stored in thread-local memory [S1, S2, S3, unofficial].

### Data Structures
*   **Shared Memory Buffer**: A buffer used to store rays with high workload.
    *   **Layout**: Each entry stores the **ray index** and the **first intersection candidate index** [S1, S2, S3, unofficial].
    *   **Note**: The indices of the remaining primitives for a given ray are consecutive in memory, so only the first candidate needs to be stored explicitly [S1, S2, S3, unofficial].
*   **Shared Memory Arrays**: Arrays used to store intersection distances during the reduction phase [S1, S2, S3, unofficial].

### Parameters
*   **High Workload Threshold**: 16 intersection candidates [S1, S2, S3, unofficial]. Rays with at least 16 candidates are considered high workload [S2, S3, unofficial]. This value was chosen because the SIMD width of 32 was too large for the available primitives per cell, and processing more than 2 rays at a time introduced excessive overhead [S1, S2, S3, unofficial].
*   **Fallback Threshold**: 20 active rays with high workload [S1, S2, S3, unofficial]. If more than 20 rays in the warp have high workload, the method falls back to the standard intersection routine [S2, S3, unofficial]. This threshold was derived by benchmarking the implementation speed against the number of active rays [S1, S2, S3, unofficial].
*   **Warp Size**: 32 threads [S1, S2, S3, unofficial].

### Steps
1.  **Check Uniformity**: Determine if all rays in the warp have intersection candidates [S1, S2, S3, unofficial].
    *   If **yes**, proceed with the standard ray-triangle intersection routine [S1, S2, S3, unofficial].
    *   If **no**, proceed to Step 2 [S1, S2, S3, unofficial].
2.  **Check High Workload Count**: Count the number of rays with high workload (≥16 intersection candidates) [S2, S3, unofficial].
    *   If the count is **greater than 20**, proceed with the standard intersection routine [S1, S2, S3, unofficial]. This fallback is used because standard packetization maps better to certain GPU architectures (e.g., NVIDIA Fermi) which support more Boolean operations at the warp level in hardware [S2, S3, unofficial].
    *   If the count is **≤ 20** (and not all rays have candidates), proceed with the Hybrid Packetization strategy [S1, S2, S3, unofficial].
3.  **Insert High Workload Rays**: For each ray identified as high workload:
    *   Use an atomic increment on a shared counter to obtain a unique index in the shared memory buffer [S1, S2, S3, unofficial].
    *   Store the **ray index** and the **first intersection candidate index** into the buffer at that position [S1, S2, S3, unofficial].
4.  **Process High Workload Rays**:
    *   Iterate through the entries in the shared memory buffer, taking **two** entries at a time [S1, S2, S3, unofficial].
    *   For each pair of rays, perform 32 intersection tests in parallel using the threads in the warp [S1, S2, S3, unofficial].
    *   Combine the results by performing a reduction on the arrays containing the intersection distances, which are stored in shared memory [S1, S2, S3, unofficial].
    *   The thread responsible for tracing a specific ray records the closest intersection (if any) in its thread-local memory [S1, S2, S3, unofficial].
5.  **Process Remaining Rays**: After all high-workload rays have been processed, perform the remaining intersection tests one by one for all rays in the warp as usual [S1, S2, S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_packetization_intersect(warp_rays, acceleration_structure):
    """
    warp_rays: List of 32 ray objects. Each ray has a list of candidate primitive indices.
    acceleration_structure: Two-level grid structure providing primitive access.
    Returns: List of closest intersection results for each ray.
    """
    # Step 1: Check if all rays have candidates
    all_have_candidates = all(len(ray.candidates) > 0 for ray in warp_rays)
    if all_have_candidates:
        return standard_ray_triangle_intersection(warp_rays, acceleration_structure)

    # Step 2: Count high workload rays
    high_workload_threshold = 16
    high_workload_rays = [ray for ray in warp_rays if len(ray.candidates) >= high_workload_threshold]
    num_high_workload = len(high_workload_rays)

    fallback_threshold = 20
    if num_high_workload > fallback_threshold:
        return standard_ray_triangle_intersection(warp_rays, acceleration_structure)

    # Step 3: Insert high workload rays into shared memory buffer
    # Shared memory buffer stores (ray_index, first_candidate_index)
    shared_buffer = []
    shared_counter = 0
    
    # Note: In actual CUDA, this uses atomicAdd on shared memory
    for ray in high_workload_rays:
        idx = atomic_add(&shared_counter, 1)
        shared_buffer.append((ray.id, ray.candidates[0]))

    # Step 4: Process high workload rays in pairs
    # Iterate taking two at a time
    for i in range(0, len(shared_buffer), 2):
        if i + 1 < len(shared_buffer):
            ray1_id, ray1_first_candidate = shared_buffer[i]
            ray2_id, ray2_first_candidate = shared_buffer[i + 1]
            
            # Perform 32 intersection tests for the pair of rays in parallel
            # This assumes the warp threads are mapped to the intersection tests
            # The primitives are accessed from the acceleration structure
            # Candidates are consecutive, so we iterate through them
            results = perform_parallel_intersection_tests(ray1_id, ray1_first_candidate, 
                                                        ray2_id, ray2_first_candidate, 
                                                        acceleration_structure)
            
            # Step 4b: Reduction on shared memory arrays
            # Combine results by performing reduction on intersection distances
            closest_dist1, closest_prim1 = reduce_intersection_results(results[0])
            closest_dist2, closest_prim2 = reduce_intersection_results(results[1])
            
            # Record closest intersection in thread local memory
            # The thread responsible for tracing ray1 stores result1
            # The thread responsible for tracing ray2 stores result2
            store_result(ray1_id, closest_dist1, closest_prim1)
            store_result(ray2_id, closest_dist2, closest_prim2)
        else:
            # Handle odd number of high workload rays if necessary
            # The source specifies taking two at a time, implying even processing
            # If odd, the last ray might be processed separately or ignored in this loop
            # Based on "taking two at a time", we assume even distribution or handle remainder
            pass

    # Step 5: Process remaining rays one by one
    # Rays not in high_workload_rays or already processed
    remaining_rays = [ray for ray in warp_rays if ray not in high_workload_rays]
    for ray in remaining_rays:
        result = perform_single_ray_intersection(ray, acceleration_structure)
        store_result(ray.id, result.distance, result.primitive_id)

    return get_all_results()
```

### Edge Cases and Pitfalls
*   **Coherence Loss**: SIMD efficiency is tied to ray coherence. If some rays traverse a full cell while others pass through an empty one, coherence in computations is lost [S1, S2, S3, unofficial].
*   **Overhead**: Processing more than 2 rays at a time introduces too much overhead [S1, S2, S3, unofficial].
*   **Fallback Necessity**: If too many rays (more than 20) have high workload, the hybrid strategy is less efficient than standard packetization, particularly on architectures like Fermi that support hardware Boolean operations [S2, S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Not explicitly given in the sources. The runtime grows with the number of primitive references, which is scene-dependent [S2, S3, unofficial].
*   **Memory Complexity**: Uses shared memory for the buffer and reduction arrays. The buffer size depends on the number of high-workload rays (up to 20) [S1, S2, S3, unofficial].
*   **Measured Performance**: Tested on NVIDIA GeForce GTX 285, GTX 280, and GTX 470 [S1, S2, S3, unofficial]. Performance data is provided for various scenes (Thai, Soda, Venice, Conference, Dragon, Fairy) with triangle counts ranging from 174K to 10M [S1, S2, S3, unofficial].
*   **Strengths**: Improves SIMD utilization in sparse grids with high primitive density per leaf [S1, S2, S3, unofficial].
*   **Limitations**: Requires benchmarking to determine optimal thresholds (20) for specific hardware [S1, S2, S3, unofficial]. Performance degrades if ray coherence is low [S1, S2, S3, unofficial].
*   **When to Use**: When using two-level grids with sparse top-level cells and high primitive density per leaf, and when standard packetization is inefficient due to workload imbalance [S1, S2, S3, unofficial].
*   **When Not to Use**: When more than 20 rays in a warp have high workload, or on architectures where standard packetization is more efficient due to hardware Boolean operations [S2, S3, unofficial].

## Background
The Hybrid Packetization Strategy was introduced by J. Kalojanov, M. Billeter, and P. Slusallek in the paper "Two-Level Grids for Ray Tracing on GPUs" [S1, S2, S3, unofficial]. It was developed to optimize ray-triangle intersection on GPUs, specifically targeting NVIDIA Fermi architecture characteristics [S2, S3, unofficial]. The method is part of a broader ray traversal system using two-level grids, which are sparse acceleration structures [S1, S2, S3, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://webhome.cs.uvic.ca/~blob/downloads/tyler/paper1227_final.pdf, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S3] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
