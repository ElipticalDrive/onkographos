# GPU-accelerated UPBP with RTX BVH

Also called: Robust Volumetric Light Transport Simulation GPU, UPBP GPU implementation

## Overview
This method is a full GPU implementation of the Unified Points, Beams, and Paths (UPBP) algorithm that leverages NVIDIA RTX hardware for light and camera path generation, combined with new accelerated data structures and a constant-time Multiple Importance Sampling (MIS) weight algorithm [S1]. It solves the performance bottleneck of CPU-based UPBP implementations by offloading path tracing and density estimation queries to the GPU, achieving significant speedups over single-threaded and multi-threaded CPU baselines [S1]. Unlike its CPU sibling [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md), this implementation specifically adapts the Point-Point and Beam-Beam estimators to work with ray-tracing acceleration structures (BVH) rather than traditional hash grids or grids, using small-extent rays for point queries and AABBs for sphere containment checks [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, participating media properties (phase function, extinction coefficient), light source definitions, camera parameters, and UPBP configuration parameters (kernel widths, subpath lengths).
*   **Outputs:** Rendered image with unbiased volumetric lighting estimates.

**Data Structures:**
The implementation utilizes specific data structures for different estimators, stored on the GPU:
1.  **Point-Point 3D (P-P3D):** Stores an Axis-Aligned Bounding Box (AABB) representing a sphere centered on each photon point. The sphere radius equals the kernel width [S1]. This structure is queried to determine if a camera point lies within the sphere (indicating distance $\le$ radius) [S1].
2.  **Beam-Beam (B-B) and Point-Beam (P-B):** Uses a BVH-based structure accelerated by RTX hardware. The source notes that this structure spends more time building than LBVH but less time querying, resulting in better overall performance for these estimators [S1].
3.  **Hash Grids:** A GPU implementation of hash grids is also available for comparison, particularly for P-P3D, but is noted to perform poorly in specific scenarios [S1].

**Algorithm Steps:**
1.  **Light Subpath Generation:** Generate light subpaths on the GPU using RTX hardware acceleration. Cache data at light vertices required for Photon Density Estimation (PDE) or Beam-Photon Transport (BTP) [S1].
2.  **Camera Subpath Generation:** Generate camera subpaths on the GPU. This kernel accounts for most of the GPU running time, as it generates subpaths, queries the data structures, and evaluates estimators [S1].
3.  **Point Query Adaptation:** For Point-Point estimators, adapt the point query to the ray-tracing framework. Create a ray with a very small extent where the origin is the point and an arbitrary direction is chosen [S1].
4.  **Density Estimation Query:**
    *   For P-P3D: Check if the camera point is inside the AABB of the photon's sphere. If inside, the distance is $\le$ kernel radius [S1].
    *   For B-B/P-B: Query the BVH structure to find relevant beams/points [S1].
5.  **MIS Weight Evaluation:** Compute the MIS weight for the formed path. The weight is evaluated in constant time by splitting the path weight into three independent parts:
    *   One part for the light subpath.
    *   One part for the camera subpath.
    *   One quantity depending on both subpath ends [S1].
    The subpath weight is formulated as a recursive quantity evaluated as the subpath is traced [S1].
6.  **Path Weight Computation:** Upon forming a full path by combining two subpaths, compute the weight only from data cached at the light and camera vertices that are merged with PDE or connected with BTP [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Kernel: Trace Camera Subpath and Evaluate UPBP Estimators

def gpu_upbp_kernel(scene, camera, light_subpaths_data, data_structures):
    """
    Executes the camera subpath generation and UPBP estimator evaluation on GPU.
    """
    # 1. Generate Camera Subpath
    # Use RTX hardware for path generation
    camera_subpath = generate_camera_subpath_rtx(camera, scene)
    
    # 2. Cache Camera Vertex Data
    # Data needed for MIS weight and density estimation
    camera_vertex_data = cache_vertex_data(camera_subpath)
    
    # 3. Iterate through Light Subpaths to form Paths
    for light_subpath_data in light_subpaths_data:
        # 4. Combine Subpaths
        # Form full path by merging light and camera subpaths
        full_path = combine_subpaths(light_subpath_data, camera_subpath)
        
        # 5. Evaluate Estimators
        # Determine which estimator to use (P-P, P-B, B-B, etc.)
        estimator_type = determine_estimator(full_path)
        
        if estimator_type == "P-P3D":
            # Point-Point 3D Estimator
            # Adapt point query to ray query
            query_point = get_camera_point(full_path)
            query_ray = create_small_extent_ray(query_point, arbitrary_direction())
            
            # Query BVH/AABB structure
            # Check if query_point is inside the sphere AABB of any photon
            photons_in_sphere = query_aabb_structure(data_structures.p_p3d_structure, query_ray)
            
            # Calculate density estimate
            density_estimate = 0.0
            for photon in photons_in_sphere:
                # Distance check is implicit in AABB containment (radius = kernel width)
                density_estimate += evaluate_kernel(query_point, photon.position, photon.kernel_width)
            
            # 6. Compute MIS Weight
            # Split weight into three parts: light subpath, camera subpath, both ends
            mis_weight = compute_mis_weight_constant_time(
                light_subpath_data.subpath_weight,
                camera_vertex_data.subpath_weight,
                full_path.endpoints
            )
            
            # 7. Final Contribution
            contribution = density_estimate * mis_weight * full_path.path_weight
            accumulate_image(contribution)
            
        elif estimator_type == "B-B1D":
            # Beam-Beam 1D Estimator
            # Query BVH structure for beam intersections
            beams = query_bvh_structure(data_structures.b_b_structure, full_path)
            # Evaluate beam-beam estimator
            density_estimate = evaluate_beam_beam_estimator(full_path, beams)
            mis_weight = compute_mis_weight_constant_time(...)
            contribution = density_estimate * mis_weight * full_path.path_weight
            accumulate_image(contribution)
            
        else:
            # Handle other estimators (P-B, B-P, etc.) similarly
            pass

def create_small_extent_ray(point, direction):
    """
    Creates a ray with a very small extent for point queries in ray tracing.
    """
    origin = point
    # Arbitrary direction, e.g., normalized random vector or fixed axis
    if direction is None:
        direction = normalize(random_vector())
    # Small extent, e.g., epsilon
    t_max = 1e-6
    return Ray(origin, direction, t_max)

def compute_mis_weight_constant_time(light_weight, camera_weight, endpoints):
    """
    Computes MIS weight in constant time by splitting into three parts.
    """
    # Part 1: Light subpath weight
    w_light = light_weight
    
    # Part 2: Camera subpath weight
    w_camera = camera_weight
    
    # Part 3: Quantity depending on both subpath ends
    w_both = evaluate_endpoint_interaction(endpoints)
    
    # Combine weights (specific formula depends on UPBP's extended balance heuristic)
    # Assuming standard MIS combination: w = w_light / (w_light + w_camera + w_both)
    total_weight = w_light + w_camera + w_both
    if total_weight == 0:
        return 0.0
    return w_light / total_weight
```

**Parameters:**
*   **Hardware:** NVIDIA RTX 2070 GPU, AMD Ryzen 5 2600 CPU (6 cores, 3.4 GHz), 16 GB RAM [S1].
*   **Subpath Lengths:** Equal number of light and camera subpaths in experiments [S1].
*   **Kernel Widths:** Determined by scene parameters (see supplementary materials in source) [S1].
*   **Iterations:** Performance measured over 120 iterations [S1].

**Edge Cases and Pitfalls:**
*   **Small Light Sources:** Hash grids perform notoriously poorly on scenes with small light sources concentrated in a small part of the scene, causing non-uniform distribution of light samples. This significantly affects P-P3D estimators in scenes like San Miguel and Sun Temple [S1].
*   **Point Query in Ray Tracing:** Point queries must be adapted to ray queries using small-extent rays, which may introduce precision issues if not handled carefully [S1].
*   **Data Structure Construction:** The new BVH-based data structures spend more time building than LBVH, which may impact performance in dynamic scenes or if frequent rebuilds are required [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Speedup:** The GPU implementation achieves a speedup of 21 to 69 times over the single-threaded SmallUPBP and 4 to 15 times over the multi-threaded version [S1].
*   **Data Structure Performance:**
    *   **Beam-Beam (B-B1D):** The new GPU data structure spends more time building than LBVH but less time querying, resulting in better overall performance [S1].
    *   **Point-Point 3D (P-P3D):** Hash grids perform best on most scenes, but fail on scenes with small, concentrated light sources (San Miguel, Sun Temple) [S1].
    *   **Point-Point 2D (P-P2D):** The new data structure has the lowest query time but highest construction time, with no clear winner among data structures [S1].

**Measured Performance:**
*   **Baseline CPU Times (SmallUPBP):**
    *   Bathroom: ST 134.01s, MT 28.46s
    *   Bistro: ST 127.22s, MT 18.28s
    *   Mirror Balls: ST 58.33s, MT 9.97s
    *   San Miguel: ST 115.06s, MT 26.84s
    *   Still Life: ST 50.72s, MT 7.86s
    *   Sun Temple: ST 419.78s, MT 89.00s
    [S1]
*   **GPU Data Structure Times (ms, 20 iterations):**
    *   **B-B1D (San Miguel):** Grid (CPU) Build 36, Query 491; Ours (GPU) Build 20, Query 2; Hash Grid (GPU) Build 8, Query 625; LBVH (GPU) Build 10, Query 52 [S1].
    *   **P-P2D (Bathroom):** Hash Grid (CPU) Build 424, Query 28646; Ours (GPU) Build 415, Query 171; Hash Grid (GPU) Build 52, Query 1946; LBVH (GPU) Build 88, Query 833 [S1].

**Strengths and Limitations:**
*   **Strengths:** Significant performance improvement over CPU implementations; efficient constant-time MIS weight evaluation; effective use of RTX hardware for path generation; robust data structures for beam-based estimators.
*   **Limitations:** Poor performance of hash grids for P-P3D in scenes with small light sources; higher construction time for the new BVH-based data structures compared to LBVH; most GPU time spent in the camera kernel.

**When to Use:**
*   Use when high-performance volumetric rendering is required and NVIDIA RTX hardware is available.
*   Use for scenes with sparse and dense media where beam-based estimators are beneficial.
*   Avoid hash grids for P-P3D in scenes with small, concentrated light sources.

**When Not to Use:**
*   Avoid if the scene has small, concentrated light sources and P-P3D is the primary estimator, as hash grids will perform poorly.
*   Avoid if frequent data structure rebuilds are required, due to higher construction times.

## Background
The method originates from the paper "Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation" [S1]. It builds on the UPBP algorithm [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md) and the CPU-based SmallUPBP implementation [Vév15, KGH*14, Vév14]. The work adapts UPBP for GPU execution, leveraging NVIDIA RTX hardware for path generation and introducing new data structures and MIS weight algorithms to optimize performance. The implementation is publicly available [S1].

## Sources
[S1] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
