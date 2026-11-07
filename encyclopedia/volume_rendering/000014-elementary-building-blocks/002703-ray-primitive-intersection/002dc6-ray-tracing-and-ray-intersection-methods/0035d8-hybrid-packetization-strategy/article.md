# Hybrid Packetization Strategy

Also called: Hybrid Intersector, Hybrid Packetization, Hybrid Packetization for Triangle Intersection, Hybrid Ray-Triangle Intersection

## Overview
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when rendering scenes with sparse acceleration structures that contain many primitives per leaf [S1, S2, S3, unofficial]. It addresses the inefficiency of standard packetization, where assigning different rays to different threads causes a loss of coherence if some rays traverse full cells while others pass through empty ones [S1, S2, S3, unofficial]. Unlike standard packetization which processes a fixed group of rays in parallel, this method dynamically distributes the workload of a single ray with high intersection candidates across multiple threads [S2, S3, unofficial].

## How it works
The method operates within a GPU warp (a SIMD unit of 32 threads) during the ray-triangle intersection phase [S1, S2, S3, unofficial]. It selects between standard warp-level intersection and a parallel processing mode based on the workload distribution of the rays in the current cell.

### Inputs and Outputs
- **Inputs**: A set of rays $\mathcal{R}$ currently traversing a leaf cell, and the list of triangle primitives $\mathcal{P}$ contained in that leaf.
- **Outputs**: For each ray, the closest intersection distance $t$ and the primitive index, or a flag indicating no intersection.

### Data Structures
- **Shared Memory Buffer**: Stores entries for rays with high workload. Each entry contains:
    - `ray_index`: The index of the ray within the warp or global context.
    - `first_candidate_index`: The index of the first intersection candidate (primitive) for that ray.
    - Note: The indices of the remaining primitives are consecutive, so only the first needs to be stored explicitly [S1, S2, unofficial].
- **Shared Memory Arrays**: Used to store intermediate intersection distances for reduction [S1, S2, unofficial].

### Parameters
- **High Workload Threshold ($H$)**: 16 intersection candidates [S1, S2, unofficial]. Defined as the number of primitives a ray must be tested against to be considered "high workload".
- **Fallback Threshold ($F$)**: 20 active rays with high workload [S1, S2, unofficial]. If the number of rays exceeding $H$ is greater than $F$, the method falls back to standard intersection.
- **SIMD Width ($W$)**: 32 threads per warp [S1, S2, S3, unofficial].
- **Parallel Pairing**: Rays are processed in pairs (2 rays per parallel unit) to balance overhead and parallelism [S1, S2, unofficial].

### Steps
1. **Check Uniformity**: Determine if all rays in the warp have intersection candidates in the current leaf cell.
    - If **yes**, proceed with standard ray-triangle intersection [S1, S2, S3, unofficial].
    - If **no**, proceed to step 2.
2. **Count High Workload Rays**: Count the number of rays that have at least $H$ (16) intersection candidates.
3. **Decide Mode**:
    - If the count of high workload rays is greater than $F$ (20), proceed with the **standard intersection routine**. This is because standard packetization maps better to hardware (e.g., Fermi GPUs) that support more Boolean operations at the warp level [S1, S2, S3, unofficial].
    - If the count is $\le 20$, proceed with the **hybrid parallel processing** mode [S1, S2, S3, unofficial].
4. **Hybrid Processing Mode**:
    - **Buffering**: Insert rays with high workload into the shared memory buffer. Use an atomic increment on a shared counter to assign buffer slots. Store only the `ray_index` and `first_candidate_index` [S1, S2, unofficial].
    - **Parallel Execution**: Iterate through the buffer entries, taking two rays at a time. Perform intersection tests for these two rays in parallel across the warp threads [S1, S2, unofficial].
    - **Reduction**: Combine the results by performing a reduction on the arrays containing intersection distances, which are shared between threads [S1, S2, unofficial].
    - **Recording**: The thread responsible for tracing a specific ray records the closest intersection distance (if any) in thread-local memory [S1, S2, unofficial].
5. **Remaining Tests**: After all high-workload rays are processed, perform the remaining intersection tests one by one for all rays as usual [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_packetization_intersect(warp_rays, leaf_primitives, warp_size=32):
    """
    Performs ray-triangle intersection using the Hybrid Packetization Strategy.
    
    Args:
        warp_rays: List of ray objects for the current warp.
        leaf_primitives: List of triangle primitives in the current leaf cell.
        
    Returns:
        List of intersection results (distance, primitive_index) for each ray.
    """
    H = 16  # High workload threshold
    F = 20  # Fallback threshold
    
    # Step 1: Check if all rays have candidates
    # Determine which rays have intersection candidates in this leaf
    ray_has_candidates = [len(get_candidates(ray, leaf_primitives)) > 0 for ray in warp_rays]
    
    if all(ray_has_candidates):
        # Proceed with standard ray-triangle intersection
        return standard_intersection(warp_rays, leaf_primitives)
    
    # Step 2: Count high workload rays
    high_workload_rays = []
    for i, ray in enumerate(warp_rays):
        candidates = get_candidates(ray, leaf_primitives)
        if len(candidates) >= H:
            high_workload_rays.append((i, candidates))
            
    count_high_workload = len(high_workload_rays)
    
    # Step 3: Decide mode
    if count_high_workload > F:
        # Fallback to standard intersection
        return standard_intersection(warp_rays, leaf_primitives)
    
    # Step 4: Hybrid Processing Mode
    # Initialize shared memory buffer and counters
    # Note: In actual CUDA, this uses __shared__ memory and atomicAdd
    buffer = []
    shared_counter = 0
    
    # Insert high workload rays into buffer
    for ray_idx, candidates in high_workload_rays:
        # Atomic increment to get slot index
        slot_idx = atomic_add(shared_counter, 1)
        buffer.append({
            'ray_index': ray_idx,
            'first_candidate_index': 0, # Index in the candidates list
            'candidates': candidates
        })
        
    # Process buffer in pairs
    num_pairs = len(buffer) // 2
    for p in range(num_pairs):
        ray1_data = buffer[2 * p]
        ray2_data = buffer[2 * p + 1]
        
        # Perform 32 intersection tests for the pair of rays in parallel
        # This assumes the warp threads are organized to handle the pair
        # Each thread might handle a subset of primitives or the pair is split
        # across threads. The source says "perform 32 intersection tests for a pair of rays in parallel".
        # This implies the pair is processed by the whole warp, likely with each thread
        # handling a portion of the primitive list or the pair being processed by subsets.
        # Given the "2 rays at a time" and "32 tests", it suggests a specific mapping.
        # We will assume a generic parallel reduction approach.
        
        dist1 = compute_intersection(ray1_data['candidates'])
        dist2 = compute_intersection(ray2_data['candidates'])
        
        # Reduction on shared memory arrays
        # This step aggregates the closest distance for each ray in the pair
        closest1 = reduce_min(dist1)
        closest2 = reduce_min(dist2)
        
        # Record results in thread-local memory for the responsible threads
        # The thread responsible for tracing ray1_data['ray_index'] stores closest1
        # The thread responsible for tracing ray2_data['ray_index'] stores closest2
        store_result(ray1_data['ray_index'], closest1)
        store_result(ray2_data['ray_index'], closest2)
        
    # Handle odd ray if buffer size is odd
    if len(buffer) % 2 == 1:
        last_ray_data = buffer[-1]
        dist = compute_intersection(last_ray_data['candidates'])
        closest = reduce_min(dist)
        store_result(last_ray_data['ray_index'], closest)
        
    # Step 5: Remaining tests
    # Perform remaining intersection tests one by one for all rays
    # This covers rays that were not high workload or were processed in standard mode
    # However, the source says "When all rays with high workload have been processed we perform the remaining intersection tests one by one for all rays as usual."
    # This implies the standard intersection is called for the non-high-workload rays, or the whole set if the hybrid mode was only for high-workload.
    # The pseudocode above handles high-workload rays. The remaining rays (low workload) need to be processed.
    
    low_workload_rays = [ray for i, ray in enumerate(warp_rays) if i not in [r['ray_index'] for r in buffer]]
    
    for ray in low_workload_rays:
        candidates = get_candidates(ray, leaf_primitives)
        if len(candidates) > 0:
            dist = compute_intersection(candidates)
            closest = reduce_min(dist)
            store_result(ray.index, closest)
        else:
            store_result(ray.index, INF)
            
    return get_all_results()

def get_candidates(ray, primitives):
    """Returns list of primitives that are potential intersection candidates for the ray."""
    # Implementation depends on the acceleration structure traversal logic
    # For this method, it returns the primitives in the current leaf cell
    return primitives

def compute_intersection(candidates):
    """Computes intersection distances for a ray against a list of primitives."""
    # Returns a list of distances, one per candidate
    # Uses Möller-Trumbore or similar algorithm
    distances = []
    for prim in candidates:
        t = ray_triangle_intersect(ray, prim)
        distances.append(t)
    return distances

def reduce_min(distances):
    """Finds the minimum positive distance in the list."""
    min_t = INF
    for t in distances:
        if t > 0 and t < min_t:
            min_t = t
    return min_t

def store_result(ray_index, distance):
    """Stores the result in thread-local memory."""
    # Implementation detail: maps ray_index to thread-local storage
    pass

def standard_intersection(rays, primitives):
    """Standard warp-level ray-triangle intersection."""
    # Uses hardware-optimized warp-level primitives
    pass
```

### Edge Cases and Pitfalls
- **Loss of Coherence**: SIMD efficiency is tied to ray coherence. If some rays traverse a full cell while others pass through an empty one, coherence is lost because parallelization assigns different rays to each thread [S1, S2, S3, unofficial].
- **Overhead**: Processing more than 2 rays at a time introduces too much overhead [S1, S2, unofficial].
- **Sparse Grids**: The method is specifically designed for sparse two-level grids with many primitives per leaf [S1, S2, S3, unofficial]. It may not be optimal for dense structures.
- **Parameter Tuning**: The threshold $F=20$ was derived by benchmarking the speed of the implementation with all possible numbers of active rays [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the number of primitives per leaf and the number of rays. The hybrid mode aims to reduce the effective workload per thread by parallelizing across rays with high candidate counts.
- **Memory Complexity**: Requires shared memory for the buffer and reduction arrays. The buffer size is proportional to the number of high-workload rays (up to 20).
- **Performance**: Measured on NVIDIA GeForce GTX 285, GTX 280, and GTX 470 [S1, unofficial]. The runtime grows with the number of primitive references, which is scene-dependent [S2, unofficial].
- **Strengths**: Improves SIMD utilization in sparse grids with many primitives per leaf [S1, S2, S3, unofficial].
- **Limitations**: Inefficient for dense grids or when ray coherence is low [S1, S2, S3, unofficial]. The fallback to standard intersection when many rays have high workload may negate benefits in those cases [S1, S2, unofficial].

## Background
The Hybrid Packetization Strategy was introduced in the context of "Two-Level Grids for Ray Tracing on GPUs" by J. Kalojanov, M. Billeter, and P. Slusallek [S1, S2, S3, unofficial]. It was developed to address the specific challenges of rendering with two-level grids, which are sparse and contain many primitives per leaf [S1, S2, S3, unofficial]. The method was tested on NVIDIA Fermi architecture GPUs (GTX 280, GTX 285, GTX 470) [S1, S2, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://webhome.cs.uvic.ca/~blob/downloads/tyler/paper1227_final.pdf, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S3] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
