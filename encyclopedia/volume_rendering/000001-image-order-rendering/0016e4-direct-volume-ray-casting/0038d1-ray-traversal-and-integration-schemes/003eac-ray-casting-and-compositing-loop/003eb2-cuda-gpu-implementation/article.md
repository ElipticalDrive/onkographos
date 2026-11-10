# CUDA GPU Implementation

Also called: CUDA-based ray casting, GPU-accelerated direct volume rendering

## Overview
This method implements direct volume ray casting on NVIDIA GPUs using the CUDA programming model to achieve nearly interactive performance [S1, unofficial]. It distinguishes itself from CPU-based implementations by leveraging massive parallel shader execution to offload the computationally intensive tasks of ray-bounding box intersection, resampling, and compositing [S1, unofficial]. Unlike sibling methods that may couple traversal with specific data structures, this approach decouples ray traversal from the volume data structure, allowing for flexible integration of various spatial search structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (scalar field), transfer function, camera parameters (origin, direction), and a spatial search data structure (dynamic redundant octree).
*   **Output:** Rendered image (color and opacity per pixel).

**Data Structures:**
*   **Thread Mapping:** One CUDA thread processes a single ray [S1, unofficial].
*   **Spatial Search:** A dynamic redundant octree as described by Guennebaud et al. [16]. This structure is chosen for its efficiency in GPU applications, supporting dynamic updates and intuitive memory footprint control [S1, unofficial].
*   **Recursion Stack:** A small stack stored in local memory to manage the recursion of the adaptive ray traversal algorithm [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Each thread initializes its ray parameters (origin, direction) based on the pixel coordinates and camera setup.
2.  **Traversal Strategy Selection:** The implementation supports two strategies:
    *   **Constant Step Size:** Fixed interval sampling along the ray.
    *   **Adaptive Algorithm:** Variable step size based on local data density or gradient magnitude. This requires recursion management via the local memory stack [S1, unofficial].
3.  **Ray Traversal Loop:**
    *   While the ray has not exited the volume bounding box and opacity is below the threshold:
        *   Determine the current sample position.
        *   If using the adaptive strategy, push current state to the local stack if branching is required, or pop if backtracking [S1, unofficial].
        *   Query the dynamic redundant octree to find the current voxel or neighbor samples [S1, unofficial].
        *   Perform resampling (e.g., trilinear interpolation) to obtain scalar value and gradient at the sample point.
        *   Apply the transfer function to compute color and opacity.
        *   Update ray position based on the step size (constant or adaptive).
4.  **Compositing:** Perform front-to-back alpha compositing using the standard volume rendering integral approximation:
    $$ C = \sum_{i=1}^{n} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $c_i$ is the color and $\alpha_i$ is the opacity at sample $i$.
5.  **Output:** Write the final accumulated color to the frame buffer.

**Parameters:**
*   **Step Size:** Constant step size is a parameter. Adaptive step size depends on the specific heuristic implemented in the adaptive algorithm (details of the heuristic are not provided in the source).
*   **Stack Size:** Small stack in local memory; exact size is implementation-dependent [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Recursion Depth:** The adaptive algorithm requires recursion. The local memory stack must be sized to handle the maximum expected recursion depth to avoid overflow [S1, unofficial].
*   **Memory Footprint:** The dynamic redundant octree's memory footprint must be balanced against performance. Too large a footprint may exceed GPU memory limits; too small may reduce traversal efficiency [S1, unofficial].
*   **Precision:** Floating-point precision in ray position updates and octree queries can affect accuracy, especially for long rays or high-resolution volumes.

## Subtopics

### [Slab-based Raycasting](11383-slab-based-raycasting.md)
Slab-based raycasting is a CUDA GPU optimization for direct volume rendering that caches sampled voxels in shared memory as "slabs" to improve performance when volume data is accessed multiple times, such as for gradient calculation in Phong lighting. It sets itself apart from basic raycasting by trading increased shared memory overhead and reduced early ray termination granularity for cache hits on repeated voxel access. Unlike hardware texture caches, which benefit from large viewports where adjacent rays hit the same voxels, the slab cache is most efficient when the data set resolution is high compared to the viewport size.

### [GPU-accelerated ray casting](12256-gpu-accelerated-ray-casting.md)
GPU-accelerated ray casting is a direct volume rendering technique that offloads the computationally intensive tasks of ray-bounding box intersection, resampling, and compositing to the graphics processing unit (GPU) to achieve real-time interactive performance. It distinguishes itself from CPU-based implementations by leveraging parallel shader execution, which significantly reduces running time despite the underlying computational complexity of interpolation remaining constant for a given viewpoint.

## Operating characteristics
*   **Performance:** Achieves nearly interactive performance [S1, unofficial].
*   **Complexity:** The computational complexity of interpolation remains constant for a given viewpoint, but the parallel execution on the GPU significantly reduces running time compared to CPU-based implementations [S1, unofficial].
*   **Scalability:** Scales with the number of GPU cores. Each thread processes a single ray, allowing for fine-grained parallelism.
*   **Strengths:** Decoupled traversal allows for flexible data structure integration. Dynamic redundant octree supports dynamic updates. Local memory stack enables adaptive traversal without global memory contention.
*   **Limitations:** Requires CUDA-capable hardware. Adaptive traversal adds complexity due to recursion management. Memory footprint of the octree must be carefully managed.
*   **When to use:** When real-time or near-real-time performance is required for volume rendering on NVIDIA GPUs, and the volume data structure benefits from dynamic updates or complex traversal strategies.

## Background
This implementation is part of the Volume MLS Ray Casting framework [S1, unofficial]. It was developed to leverage the parallel processing capabilities of NVIDIA GPUs using the CUDA programming model [34]. The use of the dynamic redundant octree is based on the work of Guennebaud et al. [16]. The approach aims to bridge the gap between offline quality and interactive speed for volume visualization.

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
