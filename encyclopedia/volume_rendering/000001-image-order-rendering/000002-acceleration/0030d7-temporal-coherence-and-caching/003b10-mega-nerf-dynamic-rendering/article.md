# Mega-NeRF-Dynamic Rendering
Also called: Dynamic Octree Caching, Temporal Coherence Rendering

## Overview
Mega-NeRF-Dynamic is an interactive rendering method for large-scale Neural Radiance Fields (NeRFs) that exploits temporal coherence in fly-throughs by dynamically expanding and reusing a sparse voxel octree cache across frames [S1]. Unlike sibling methods such as [Plenoctree](link) which rely on a fixed, precomputed octree that limits resolution, Mega-NeRF-Dynamic dynamically subdivides the octree based on the current camera position, allowing for higher resolution and incremental refinement [S1]. It addresses the problem of high latency in large-scale NeRF rendering by reducing per-frame work through state caching and guided sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained Mega-NeRF model (comprising spatial MLPs and feature modules), a camera trajectory (fly-through), and the octree cache from the previous frame (or a coarse initial cache for the first frame) [S1].
*   **Outputs:** A rendered image frame and an updated sparse voxel octree cache containing refined opacity and color data [S1].

**Data Structures:**
*   **Sparse Voxel Octree:** A hierarchical structure storing precomputed model outputs (opacity and color/spherical harmonic coefficients) [S1].
    *   **Layout:** The octree is dynamically expanded. The source notes that generating an 8-level octree requires 1–12 GB of memory, with higher levels scaling significantly (e.g., adding one level increases size to 55 GB) [S1].
    *   **Weights:** The octree stores weights used to guide ray sampling, estimating scene geometry to place samples near surfaces of interest [S1].

**Algorithm Steps:**
1.  **Initialization (Precomputation):** Precompute a coarse cache of opacity and color into a sparse voxel octree [S1].
2.  **Initial View Generation:** For the current frame, use the cached outputs from the previous frame's expanded octree to produce an initial view [S1].
3.  **Dynamic Expansion & Refinement:** Dynamically subdivide the octree based on the current camera position [S1]. Perform additional rounds of model sampling to refine the image in regions of interest and store these new values into the cache [S1].
4.  **Guided Ray Sampling:** Perform a final round of guided ray sampling in a single pass [S1]. Unlike standard hierarchical sampling which uses uniform coarse intervals followed by fine sampling, this method uses the weights stored in the octree to skip empty spaces and place a small number of samples near surfaces of interest [S1].
5.  **Transmittance Accumulation:** Accumulate transmittance along the ray [S1]. End sampling after a certain threshold is reached to accelerate the process [S1].
6.  **Cache Update:** The refined octree from the current frame is reused for the next frame, requiring only incremental work to maintain quality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_frame_mega_nerf_dynamic(camera, prev_octree, nerf_model, step_size=0.5, transmittance_threshold=0.001):
    """
    Renders a single frame using dynamic octree caching and guided sampling.
    
    Args:
        camera: Current camera pose and intrinsics.
        prev_octree: Sparse voxel octree from the previous frame.
        nerf_model: The trained Mega-NeRF model.
        step_size: Distance between samples along the ray.
        transmittance_threshold: Early termination threshold for transmittance.
        
    Returns:
        image: Rendered RGBA image buffer.
        current_octree: Updated octree for the next frame.
    """
    # Step 1: Initialize octree for current frame based on previous state
    # If first frame, use coarse precomputed cache; otherwise, reuse and expand
    current_octree = prev_octree.copy() if prev_octree else build_coarse_cache(nerf_model)
    
    # Step 2: Dynamic Expansion
    # Subdivide octree nodes based on camera proximity and current view requirements
    current_octree = dynamic_subdivide(current_octree, camera)
    
    # Step 3: Initial View & Refinement
    # Use cached outputs for a quick initial view
    initial_image = render_from_cache(current_octree, camera)
    
    # Perform additional rounds of model sampling to refine the image
    # Store new values into the cache
    refined_octree = refine_octree(current_octree, camera, nerf_model)
    
    # Step 4 & 5: Guided Ray Sampling & Transmittance Accumulation
    image = empty_image_buffer(camera.resolution)
    for y in range(camera.resolution[1]):
        for x in range(camera.resolution[0]):
            ray_origin, ray_direction = generate_ray(camera, x, y)
            accumulated_color = [0.0, 0.0, 0.0]
            transmittance = 1.0
            t = 0.0
            
            # Traverse octree and sample along ray
            while t < camera.far_plane and transmittance > transmittance_threshold:
                pos = ray_origin + ray_direction * t
                node = current_octree.query_node(pos)
                
                if node is not None:
                    density, color = nerf_model.query(pos)
                    # Apply octree weight for guided sampling if available
                    weight = node.get_weight() if hasattr(node, 'get_weight') else 1.0
                    alpha = 1.0 - math.exp(-density * weight * step_size)
                    accumulated_color[0] += alpha * transmittance * color[0]
                    accumulated_color[1] += alpha * transmittance * color[1]
                    accumulated_color[2] += alpha * transmittance * color[2]
                    transmittance *= (1.0 - alpha)
                t += step_size
                
            # Return RGBA: color + accumulated alpha (1 - transmittance)
            image[y, x] = accumulated_color + [1.0 - transmittance]
            
    return image, refined_octree

def build_coarse_cache(nerf_model):
    # Precompute coarse cache of opacity and color into a sparse voxel octree
    # Not given in the sources.
    pass

def dynamic_subdivide(octree, camera):
    # Dynamically subdivide the octree based on the current camera position
    # Not given in the sources.
    pass

def render_from_cache(octree, camera):
    # Use cached outputs to produce an initial view
    # Not given in the sources.
    pass

def refine_octree(octree, camera, nerf_model):
    # Perform additional rounds of model sampling to refine the image
    # Store new values into the cache
    # Not given in the sources.
    pass

def empty_image_buffer(resolution):
    # Initialize an empty buffer for the image
    return [[None for _ in range(resolution[0])] for _ in range(resolution[1])]

def generate_ray(camera, x, y):
    # Generate a ray from camera origin through pixel (x, y)
    # Standard pinhole camera projection
    # Not given in the sources.
    pass
```

**Parameters:**
*   **Octree Depth:** The source notes that an 8-level octree is used for scenes, with higher levels increasing memory usage drastically (55 GB for one additional level) [S1].
*   **Transmittance Threshold:** Sampling ends after a certain threshold is reached to accelerate the process [S1].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Generating high-level octrees is memory-intensive. Adding a single level can increase size to 55 GB, exceeding the capacity of many GPUs [S1].
*   **Precomputation Time:** Generating the initial coarse cache takes significant time (1 hour for 8 levels) [S1].
*   **Camera Motion:** The method relies on temporal coherence. Large, sudden camera movements may reduce the effectiveness of cache reuse, requiring more incremental work [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation is expensive (1 hour for 8-level octree) [S1]. Interactive rendering is accelerated by reusing the octree, requiring only incremental work per frame [S1].
*   **Memory Complexity:** High. An 8-level octree requires 1–12 GB of memory. Adding a level increases this to 55 GB [S1].
*   **Performance:** Reduces rendering time from over 2 minutes per 720p frame (with normal ray sampling) to something more manageable for interactive fly-throughs [S1].
*   **Strengths:** High resolution due to dynamic expansion; efficient reuse of computation across frames; guided sampling reduces unnecessary queries [S1].
*   **Limitations:** High memory footprint for deep octrees; precomputation overhead [S1].
*   **When to use:** Large-scale NeRF fly-throughs where visual fidelity and resolution are prioritized, and hardware has sufficient memory [S1].
*   **When not to use:** Scenes with rapid, non-sequential camera motion or hardware with limited memory [S1].

## Background
Mega-NeRF-Dynamic was introduced in the paper "Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs" [S1]. It was developed to address the limitations of existing fast NeRF renderers like Plenoctree, which use fixed octrees that limit resolution [S1]. The method is designed for search-and-rescue use cases, aiming to preserve visual fidelity while minimizing processing time beyond training [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
