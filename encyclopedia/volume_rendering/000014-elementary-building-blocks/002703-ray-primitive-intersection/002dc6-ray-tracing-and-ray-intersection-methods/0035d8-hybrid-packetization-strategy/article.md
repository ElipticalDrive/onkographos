# Hybrid Packetization Strategy

Also called: Hybrid Intersector, Hybrid Packetization, Hybrid Packetization for Triangle Intersection, Hybrid Ray-Triangle Intersection

## Overview
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when standard packetization becomes inefficient due to sparse acceleration structures with high primitive density per leaf [S1, S2, S3, unofficial]. It solves the problem of lost computational coherence in warp-level parallelism by dynamically switching between standard warp-level intersection and a specialized parallel processing mode that distributes the workload of a single ray across multiple threads [S2, S3, unofficial]. It differs from standard packetization, which assigns one ray per thread, by allowing a single ray with many intersection candidates to be processed by multiple threads within a warp, thereby maintaining high occupancy even when ray coherence is low [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A warp of 32 rays, a set of intersection candidates (primitive indices) for each ray, and a shared memory buffer.
*   **Output:** The closest intersection distance $t$ for each ray, stored in thread-local memory.

**Data Structures**
*   **Shared Memory Buffer:** Stores entries for rays with "high workload." Each entry contains:
    *   `ray_index`: The index of the ray within the warp (0-31).
    *   `first_candidate_index`: The index of the first primitive in the candidate list for that ray.
    *   *Note:* The indices of the remaining primitives for a given ray are assumed to be consecutive in memory [S1, S2, unofficial].
*   **Shared Memory Arrays:** Arrays to store intermediate intersection distances for reduction operations [S1, S2, unofficial].

**Parameters**
*   **High Workload Threshold:** 16 intersection candidates [S1, S2, unofficial].
*   **Standard Fallback Threshold:** 20 active rays with high workload [S1, S2, unofficial].
*   **Warp Size:** 32 threads [S1, S2, S3, unofficial].
*   **Parallel Pairing:** Rays are processed in pairs (2 rays per 32-thread warp iteration) [S1, S2, unofficial].

**Steps**
1.  **Check Uniformity:** Determine if all 32 rays in the warp have intersection candidates.
    *   If **yes**, proceed with standard ray-triangle intersection for the entire warp [S1, S2, S3, unofficial].
    *   If **no**, proceed to Step 2.
2.  **Check High Workload Count:** Count the number of rays with a high workload (≥16 intersection candidates).
    *   If the count is **> 20**, proceed with the standard intersection routine. This is because standard packetization maps better to hardware (e.g., Fermi GPUs) that support many Boolean operations at the warp level [S1, S2, S3, unofficial].
    *   If the count is **≤ 20** (and > 0), proceed with the Hybrid Packetization strategy [S1, S2, S3, unofficial].
3.  **Buffer High Workload Rays:**
    *   For each ray with high workload, insert its `ray_index` and `first_candidate_index` into the shared memory buffer.
    *   Use an atomic increment on a shared counter to determine the insertion position [S1, S2, unofficial].
4.  **Process Buffered Rays:**
    *   Iterate through the entries in the shared memory buffer, taking two entries at a time [S1, S2, unofficial].
    *   For each pair of rays, perform 32 intersection tests in parallel using the warp's threads [S1, S2, unofficial].
    *   *Note:* Since the remaining primitive indices are consecutive, the threads can compute the indices for the subsequent primitives based on the `first_candidate_index` and the thread's lane ID or a loop counter [S1, S2, unofficial].
5.  **Reduce Results:**
    *   Perform a reduction operation on the arrays containing the intersection distances, which are shared between threads [S1, S2, unofficial].
    *   Identify the minimum $t$ for each ray [S1, S2, unofficial].
6.  **Store Closest Intersection:**
    *   The thread responsible for tracing the specific ray (typically determined by `ray_index`) records the closest intersection distance in its thread-local memory [S1, S2, unofficial].
7.  **Process Remaining Rays:**
    *   After all high-workload rays in the buffer have been processed, perform the remaining intersection tests for the other rays in the warp one by one as usual [S1, S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_packetization_intersect(warp_rays, candidates):
    """
    warp_rays: List of 32 ray objects (origin, direction)
    candidates: List of lists, where candidates[i] is a list of primitive indices for ray i
    """
    warp_size = 32
    high_workload_threshold = 16
    fallback_threshold = 20
    
    # Step 1: Check if all rays have candidates
    all_have_candidates = all(len(cands) > 0 for cands in candidates)
    
    if all_have_candidates:
        # Proceed with standard ray-triangle intersection
        return standard_warp_intersection(warp_rays, candidates)
    
    # Step 2: Count high workload rays
    high_workload_rays = []
    for i in range(warp_size):
        if len(candidates[i]) >= high_workload_threshold:
            high_workload_rays.append(i)
            
    count_high_workload = len(high_workload_rays)
    
    # Step 3: Decide strategy
    if count_high_workload > fallback_threshold:
        # Standard intersection maps better to hardware (e.g., Fermi)
        return standard_warp_intersection(warp_rays, candidates)
    
    # Step 4: Hybrid Strategy
    # Initialize shared memory buffer and counter
    # Note: In real CUDA, this would be __shared__ variables
    shared_buffer = []
    shared_counter = 0
    
    # Insert high workload rays into buffer
    for ray_idx in high_workload_rays:
        # Atomic increment on shared counter
        pos = atomic_add(&shared_counter, 1)
        shared_buffer.append({
            'ray_index': ray_idx,
            'first_candidate_index': candidates[ray_idx][0]
        })
        
    # Step 5: Process buffered rays in pairs
    # Iterate through buffer taking two at a time
    num_pairs = len(shared_buffer) // 2
    for p in range(num_pairs):
        ray1_idx = shared_buffer[2*p]['ray_index']
        ray2_idx = shared_buffer[2*p+1]['ray_index']
        
        # Perform 32 intersection tests for the pair of rays
        # Each thread in the warp handles one primitive test
        # The primitives are accessed based on the first_candidate_index
        # and the thread's lane ID or a loop counter to get consecutive indices
        
        # Example logic for thread t (0-31):
        # For ray1: primitive index = first_candidate_index + t
        # For ray2: primitive index = first_candidate_index + t
        # Compute intersection t1, t2 for each ray
        
        # Store results in shared memory arrays for reduction
        # ... (implementation details for intersection and storage)
        
        # Reduction on shared memory arrays to find min t for each ray
        min_t_ray1 = reduce_min(shared_distances_for_ray1)
        min_t_ray2 = reduce_min(shared_distances_for_ray2)
        
        # Step 6: Store closest intersection in thread-local memory
        # Thread responsible for ray1 (e.g., thread 0 or specific mapping)
        if thread_id == ray1_idx: # Or appropriate mapping
            local_result[ray1_idx] = min_t_ray1
            
        if thread_id == ray2_idx:
            local_result[ray2_idx] = min_t_ray2
            
    # Handle odd ray if buffer size is odd
    if len(shared_buffer) % 2 == 1:
        # Process last ray similarly
        pass
        
    # Step 7: Process remaining rays one by one
    for i in range(warp_size):
        if i not in high_workload_rays:
            # Standard intersection for this ray
            local_result[i] = intersect_ray_triangle(warp_rays[i], candidates[i])
            
    return local_result

def standard_warp_intersection(warp_rays, candidates):
    # Standard implementation where each thread handles one ray
    # and all rays have candidates
    results = []
    for i in range(32):
        results.append(intersect_ray_triangle(warp_rays[i], candidates[i]))
    return results
```

**Edge Cases and Pitfalls**
*   **Coherence Loss:** SIMD efficiency is tied to ray coherence. If some rays traverse a full cell while others pass through an empty one, coherence is lost because parallelization assigns different rays to each thread [S1, S2, unofficial].
*   **Overhead:** Processing more than 2 rays at a time introduces too much overhead, which is why the strategy processes rays in pairs [S1, S2, unofficial].
*   **Parameter Tuning:** The threshold of 20 active rays was derived by benchmarking the speed of the implementation with all possible numbers of active rays [S1, S2, unofficial]. The threshold of 16 candidates was chosen because the SIMD width of 32 was too big and there were not many cells with enough primitives [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of intersection candidates and the number of high-workload rays. The hybrid strategy aims to reduce the effective time by improving SIMD utilization in cases where standard packetization would suffer from low occupancy [S1, S2, unofficial].
*   **Memory Complexity:** Requires additional shared memory for the buffer and reduction arrays. The buffer size is proportional to the number of high-workload rays (up to 20) [S1, S2, unofficial].
*   **Measured Performance:** Tested on NVIDIA GeForce GTX 285, GTX 280, and GTX 470 [S1, S2, unofficial]. Performance scales with the number of primitive references, which is scene-dependent [S2, unofficial]. Table 2 in [S1, unofficial] provides build statistics and render times for various scenes (e.g., Thai, Soda, Venice) on a GTX 285.
*   **Strengths:** Improves SIMD utilization for sparse grids with many primitives per leaf [S1, S2, unofficial]. Effective when standard packetization is inefficient due to lack of ray coherence [S1, S2, unofficial].
*   **Limitations:** Requires careful tuning of thresholds (16 candidates, 20 rays) which may vary across architectures [S1, S2, unofficial]. The strategy is specifically optimized for GPU architectures like Fermi [S1, S2, unofficial].
*   **When to Use:** Use when rendering scenes with sparse acceleration structures (like two-level grids) that have high primitive density per leaf and when ray coherence is low [S1, S2, unofficial].
*   **When Not to Use:** Use standard packetization if all rays have candidates or if the number of high-workload rays exceeds 20 [S1, S2, unofficial].

## Background
*   **Origin:** The method was introduced in the paper "Two-Level Grids for Ray Tracing on GPUs" by J. Kalojanov, M. Billeter, and P. Slusallek [S1, S2, S3, unofficial].
*   **History:** The paper presents a two-level grid structure for ray tracing on GPUs. The hybrid packetization strategy was developed to address the inefficiencies of standard packetization in this specific context [S1, S2, S3, unofficial].
*   **Usage:** Used in GPU ray tracers utilizing two-level grids, particularly on NVIDIA GPUs (GTX 280, 285, 470) [S1, S2, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://webhome.cs.uvic.ca/~blob/downloads/tyler/paper1227_final.pdf, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S3] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
