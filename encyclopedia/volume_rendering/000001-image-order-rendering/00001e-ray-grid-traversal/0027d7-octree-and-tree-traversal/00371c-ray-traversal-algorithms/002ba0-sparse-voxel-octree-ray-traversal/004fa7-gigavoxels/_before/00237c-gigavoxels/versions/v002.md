# GigaVoxels

Also called: Dynamic Sparse Octree, Ray-Guided Streaming

## Overview
GigaVoxels is a dynamic sparse octree approach for efficient, detailed voxel rendering that achieves real-time to interactive rates on volumes exceeding GPU memory capacities by using ray-guided streaming [S1, unofficial]. It solves the problem of rendering billions of voxels by ensuring the volume is only used at the resolution needed to produce the final image, providing memory and speed gains while offering inherent anti-aliasing [S1, unofficial]. It differs from its parent, [Sparse Voxel Octree Ray Traversal](933-sparse-voxel-octree-ray-traversal.md), by continuously adapting resolution based on the current view and occlusion information, making discontinuities disappear and ensuring smooth temporal transitions to different subdivision levels [S1, unofficial].

## How it works
**Inputs and Outputs:**
* Input: A volumetric dataset (e.g., 3D scans, hypertextured meshes, or fractals) with resolutions such as $8192^3$ or $16384^3$ [S1, unofficial].
* Output: A rendered image at interactive rates (20-90 fps) [S1, unofficial].

**Data Structures:**
The method combines an adaptive version of the structure from [LHN05a] with regular constant-sized memory blocks from [LKS*06] [S1, unofficial]. This hybrid structure supports efficient hardware-based filtering and exploits "sparse-twice" data structures (dense clusters in free space) [S1, unofficial]. The hierarchy is built by iteratively downsampling and averaging neighbors, creating a mipmapping-like structure [S1, unofficial].

**Parameters:**
* Resolution: Examples include $8192^3$ for 3D scans and $16384^3$ for hypertextured meshes [S1, unofficial].
* Quality/Performance Trade-off: The system allows a trade-off between quality and performance, exploiting temporal coherence [S1, unofficial].
* Opacity Threshold: Rays stop when a certain opacity level is reached [S1, unofficial].

**Algorithm Steps:**
1. Ray Casting: Cast rays from the eye through the pixel extent [S1, unofficial].
2. Ray-Guided Streaming: Guide data production and streaming directly based on information extracted during rendering [S1, unofficial]. This involves loading only the necessary octree nodes into GPU memory based on the ray's path [S1, unofficial].
3. Adaptive Traversal: Traverse the sparse octree [S1, unofficial]. The resolution adapts continuously to the current view and occlusion information [S1, unofficial].
4. Mipmapping Filtering: When traversing to larger steps (proportional to distance from the observer), read from the mipmap hierarchy [S1, unofficial]. Reading from a mipmap delivers the integrated value of a small volume, approximating the cone integral defined by the eye and pixel extent [S1, unofficial].
5. Early Termination: Stop rays when a certain opacity level is reached or when traversing empty space [S1, unofficial].
6. Procedural Enrichment: Create additional details through procedural methods to enrich the dataset [S1, unofficial].

**Pseudocode:**
```python
def render_gigavoxels(camera, scene, opacity_threshold=0.95):
    height, width = camera.resolution
    image = [[vec3(0.0) for _ in range(width)] for _ in range(height)]
    
    for py in range(height):
        for px in range(width):
            ray = camera.generate_ray(px, py)
            accumulated_color = vec3(0.0)
            accumulated_opacity = 0.0
            ray_terminated = False
            node = scene.get_root_node()
            
            while not ray_terminated and node is not None:
                if node.is_empty():
                    ray.origin += ray.direction * node.size
                    node = None
                    continue
                
                next_node = find_intersecting_child(node, ray)
                if next_node is None:
                    ray.origin += ray.direction * node.size
                    node = None
                    continue
                
                distance = length(ray.origin - camera.position)
                step_size = next_node.size
                
                if distance > threshold:
                    volume_value = next_node.read_mipmap(ray.direction, ray.origin)
                else:
                    volume_value = next_node.read_voxel(ray.direction, ray.origin)
                
                alpha = volume_value.alpha
                color = volume_value.color
                weight = alpha * (1.0 - accumulated_opacity)
                accumulated_color += color * weight
                accumulated_opacity += weight
                
                if accumulated_opacity > opacity_threshold:
                    ray_terminated = True
                    break
                
                ray.origin += ray.direction * step_size
                node = next_node
            
            image[py][px] = accumulated_color
    return image
```
*Note: `find_intersecting_child`, `read_mipmap`, `read_voxel`, `vec3`, and `length` are standard vector/math operations or octree traversal helpers [G].*

**Edge Cases and Pitfalls:**
* Aliasing: Appears because pixel value should be defined by integration along a cone, not just ray accumulation [S1, unofficial]. Mipmapping addresses this by smoothing results and allowing larger step sizes proportional to distance [S1, unofficial].
* Memory Budget: The method respects the limited GPU memory budget by streaming only necessary data [S1, unofficial].
* Temporal Coherence: The system exploits temporal coherence to maintain quality and performance [S1, unofficial].

## Operating characteristics
* Performance: 20-90 fps on current generation hardware [S1, unofficial].
* Memory: Respects limited GPU memory budget by streaming data [S1, unofficial].
* Scalability: Handles volumes exceeding GPU memory capacities, with resolutions up to $16384^3$ or theoretically infinite for fractals [S1, unofficial].
* Strengths: Real-time to interactive rates, inherent anti-aliasing, adaptive resolution, efficient handling of "sparse-twice" data structures [S1, unofficial].
* Limitations: Increases memory consumption due to mipmap hierarchy and makes tree updates more challenging [S1, unofficial].
* When to use: When rendering large volumetric datasets (billions of voxels) on hardware with limited GPU memory, especially for computer graphics scenes with dense clusters in free space [S1, unofficial].
* When not to use: When view-independent structures are sufficient, or when procedural content generation is not required [S1, unofficial].

## Background
GigaVoxels was developed by Sylvain Lefebvre, Elmar Eisemann, and others at INRIA Sophia-Antipolis and MPI Informatik / Saarland University [S1, unofficial]. It was published in the I3D 2009 conference proceedings [S1, unofficial]. The approach was developed in parallel to the work of Gobbetti et al. [GMAG08] [S1, unofficial]. It builds on the structure from [LHN05a] and [LKS*06] [S1, unofficial].

## Sources
[S1] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
