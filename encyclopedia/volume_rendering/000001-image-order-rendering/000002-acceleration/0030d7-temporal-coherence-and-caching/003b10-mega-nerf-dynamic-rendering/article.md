# Mega-NeRF-Dynamic Rendering
Also called: Dynamic Octree Caching, Temporal Coherence Rendering

## Overview
Mega-NeRF-Dynamic Rendering is an interactive rendering method for large-scale Neural Radiance Fields (NeRFs) that exploits temporal coherence in fly-through sequences by dynamically expanding and reusing a sparse voxel octree cache across frames [S1]. Unlike fixed-resolution caching methods such as Plenoctree, which limit image fidelity due to static data structures, this method dynamically subdivides the octree based on the current camera position, allowing for higher resolution and incremental refinement [S1]. It addresses the problem of high latency in large-scale NeRF fly-throughs by minimizing additional processing time beyond the initial training phase [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained Mega-NeRF model (a collection of self-contained MLPs), the previous frame's dynamic octree cache, and the current camera pose.
*   **Output:** A rendered image frame and an updated dynamic octree cache for the next frame.

**Data Structures:**
*   **Sparse Voxel Octree:** A hierarchical structure storing precomputed opacity and color (radiance) values. The tree is not fixed; it expands dynamically.
*   **Octree Weights:** Stored in the octree nodes to guide ray sampling, providing a high-quality estimate of scene geometry [S1].

**Steps:**
1.  **Precomputation:** Precompute a coarse cache of opacity and color into a sparse voxel octree [S1]. This initial step is computationally expensive (see Operating characteristics).
2.  **Dynamic Expansion:** During interactive visualization, dynamically subdivide the octree based on the current camera position [S1]. This contrasts with fixed octrees used in methods like Plenoctree [S1].
3.  **Initial View Generation:** Use the cached outputs from the expanded octree to produce an initial view of the current frame [S1].
4.  **Refinement:** Perform additional rounds of model sampling to refine the image quality. Store these new, refined values into the octree cache [S1].
5.  **Temporal Reuse:** For the next frame, reuse the expanded octree from the previous frame. Due to temporal coherence, only incremental work is required to maintain quality [S1].
6.  **Guided Ray Sampling:** Perform a final round of guided ray sampling in a single pass. Use the weights stored in the octree to place samples near surfaces of interest, skipping empty spaces [S1]. This replaces the traditional two-stage hierarchical sampling [S1].
7.  **Transmittance Accumulation:** Accumulate transmittance along the ray and terminate sampling after a certain threshold to accelerate the process [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame_mega_nerf_dynamic(prev_octree, camera_pose, nerf_model):
    # Step 2: Dynamic Expansion
    # Subdivide the octree based on current camera position
    current_octree = dynamic_expand_octree(prev_octree, camera_pose)
    
    # Step 3: Initial View
    # Render using cached outputs for speed
    initial_image = render_from_cache(current_octree, camera_pose)
    
    # Step 4: Refinement
    # Perform additional sampling to refine image and update cache
    refined_image, updated_octree = refine_image_and_cache(
        initial_image, 
        current_octree, 
        camera_pose, 
        nerf_model
    )
    
    # Step 6 & 7: Guided Sampling & Transmittance
    # Final pass using octree weights to skip empty space
    final_image = guided_ray_marching(
        updated_octree, 
        camera_pose, 
        nerf_model,
        transmittance_threshold=0.95 # Example threshold [S1]
    )
    
    return final_image, updated_octree
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The method requires significant memory for large scenes. Generating an 8-level octree can take 1-12 GB of memory [S1].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Rays are terminated near ground level to avoid querying underground regions, which improves efficiency [S1].

## Subtopics

### Dynamic vs. Fixed Octree
Mega-NeRF-Dynamic expands the octree based on camera position, whereas Plenoctree uses a fixed octree [S1]. Choose Mega-NeRF-Dynamic when high resolution and fly-through coherence are required [S1]. See [Plenoctree](link).

### Guided Sampling
Uses octree weights to place samples near surfaces, skipping empty space, unlike standard hierarchical sampling [S1]. See [Hierarchical Sampling](link).

## Operating characteristics
*   **Time Complexity:** Precomputation is heavy. Generating an 8-level octree took 1 hour of computation [S1]. Adding one level increased time to 10 hours [S1]. Interactive rendering is accelerated by reusing the octree [S1].
*   **Memory Complexity:** High. 1-12 GB for an 8-level octree [S1]. Adding a level increased size to 55 GB, exceeding the capacity of most GPUs [S1].
*   **Measured Performance:** Normal ray sampling takes over 2 minutes for a 720p frame [S1]. The dynamic method accelerates this to interactive speeds [S1].
*   **Strengths:** Preserves visual fidelity, minimizes additional processing time beyond training, and accelerates rendering for large-scale scenes [S1].
*   **Limitations:** High memory requirements for deep octrees; precomputation time is significant [S1].
*   **When to use:** Interactive fly-throughs of large-scale scenes where temporal coherence is high [S1].
*   **When not to use:** Static views or scenes with large, rapid changes where temporal coherence is low [S1].

## Background
Mega-NeRF-Dynamic Rendering was introduced in the paper "Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs" [S1]. It was developed to address the limitations of existing fast NeRF renderers, such as Plenoctree, which use fixed octrees that limit resolution [S1]. The method is particularly relevant for search-and-rescue use cases requiring high fidelity and low latency [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
