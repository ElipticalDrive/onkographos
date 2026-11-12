# Parallel Splatting

Also called: Parallel Projection, Tsplat parallel projection

## Overview
Parallel Splatting is a distributed volume rendering technique that applies the **sort-last** parallelization paradigm to the splatting pipeline [S2, unofficial]. It solves the scalability problem of rendering large volumetric datasets by partitioning the workload across multiple processing nodes or cores, which render partial images that are subsequently composited into a final result [S2, unofficial]. It is distinguished from its siblings by its explicit reliance on a master-node compositing stage and subvolume-based data distribution, contrasting with [Ray-Driven Splatting](1156-ray-driven-splatting.md) which often employs sort-first or hybrid approaches, and [Hybrid Sort-First/Sort-Last Parallel Rendering](9056-hybrid-sort-first-sort-last-parallel-rendering.md) which combines both paradigms [S2, unofficial].

## How it works
The method distributes volumetric data to processing units, renders them in parallel to partial images, and composites these images in depth-sorted order [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A volumetric dataset (voxels), camera parameters, and a set of processing nodes/threads.
*   **Output:** A final 2D image.

**Data Structures:**
*   **Volume Data:** Partitioned into slices (axis-aligned planes) or blocks [S2, unofficial].
*   **Partial Images:** Rendered buffers containing the projection of assigned subvolumes.
*   **Hierarchical Structures:** Optional k-d trees used to facilitate compositing and occlusion culling [S2, unofficial].

**Steps:**
1.  **Data Partitioning:** Distribute the volume data to processing nodes. This is done either by assigning axis-aligned slices or 3D blocks [S2, unofficial]. Alternatively, tasks are defined by specific subvolumes assigned to each processor [S2, unofficial].
2.  **Parallel Projection:** Each node renders its assigned subvolume in parallel. For each voxel/splat in the subvolume:
    *   Compute the projection onto the image plane.
    *   Determine the bounding box of the projected splat.
    *   If the splat is valid (within bounds), add it to the partial image buffer [S1, unofficial].
    *   *Note:* Standard implementations splat all voxels regardless of emptiness or occlusion [S2, unofficial].
3.  **Partial Image Generation:** Each node produces a partial image representing the contribution of its assigned subvolume [S2, unofficial].
4.  **Optimization (Optional):** Pass only non-empty parts of the partial images to reduce communication overhead [S2, unofficial].
5.  **Compositing:** Composite the partial images into the final image. This is performed by a master node [S2, unofficial]. The compositing is done in depth-sort order [S2, unofficial].
6.  **Parallel Compositing (Optional):** The final compositing stage can be parallelized using screen-space partitioning [S2, unofficial]. Hierarchical data structures like k-d trees may be used to accelerate this step and enable occlusion culling [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_splatting(volume_data, camera, num_nodes):
    # Step 1: Partition volume data into subvolumes (slices or blocks)
    subvolumes = partition_volume(volume_data, num_nodes)
    
    # Step 2: Render subvolumes in parallel
    partial_images = []
    for i in range(num_nodes):
        subvol = subvolumes[i]
        # Render assigned subvolume to a partial image
        img = render_subvolume(subvol, camera)
        partial_images.append(img)
        
    # Step 3: Composite partial images (Sort-Last)
    # In a distributed setting, this is often done by a master node
    final_image = composite_partial_images(partial_images, depth_sorted=True)
    
    return final_image

def render_subvolume(subvolume, camera):
    # Initialize empty partial image buffer
    buffer = create_empty_buffer(camera.width, camera.height)
    
    # Iterate over voxels/splats in the subvolume
    for voxel in subvolume:
        # Project voxel to screen space
        projected = project(voxel, camera)
        
        # Check if valid and within bounds
        if is_valid(projected):
            # Add contribution to buffer
            # Note: Standard parallel splatting does not check occlusion here
            add_to_buffer(buffer, projected)
            
    return buffer

def composite_partial_images(partial_images, depth_sorted=True):
    # Sort partial images by depth if required
    if depth_sorted:
        partial_images = sort_by_depth(partial_images)
        
    # Composite images
    # This can be parallelized using screen-space partitioning
    final_image = create_empty_image()
    for img in partial_images:
        blend(final_image, img)
        
    return final_image
```

**Parameters:**
*   **Partitioning Strategy:** Slices (axis-aligned planes) or Blocks [S2, unofficial].
*   **Compositing Order:** Depth-sorted [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion Ignorance:** Standard parallel splatting implementations splat all voxels, including empty or occluded ones, which leads to redundant computation [S2, unofficial].
*   **Load Balancing:** Implementations that account for visibility and occlusion are considerably more challenging for load-balancing [S2, unofficial].

## Subtopics

### Parallel Splatting with Visibility/Occlusion
A parallel splatting algorithm that accounts for visibility and occlusion during rendering [S2, unofficial]. It is more challenging to implement due to the difficulty of load-balancing when the amount of work depends on dynamic visibility information [S2, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is proportional to the number of voxels and processing nodes. Memory complexity depends on the partitioning strategy and the size of partial images.
*   **Performance:** Speed-ups are achieved by parallelizing the rendering stage and optimizing communication by passing only non-empty parts of partial images [S2, unofficial].
*   **Strengths:** Scalable to large datasets via distributed memory architectures; leverages existing splatting efficiency [S2, unofficial].
*   **Limitations:** Standard implementations lack occlusion culling, leading to wasted computation on empty/occluded voxels [S2, unofficial]. Load balancing is difficult when visibility-aware optimizations are used [S2, unofficial].
*   **When to use:** Large volumetric datasets requiring distributed rendering where exact occlusion culling is not critical or can be approximated.
*   **When not to use:** Scenes with high occlusion where splatting all voxels is prohibitively expensive, or when strict load balancing is required without complex visibility tracking.

## Background
Parallel splatting has been researched as a scalable solution for volume rendering [S2, unofficial]. It utilizes the sort-last paradigm, where nodes render partial images composited by a master node [S2, unofficial]. Machiraju and Yagel reported an early parallel implementation where tasks were defined by subvolumes assigned to processors [S2, unofficial]. Huang presented a variant that accounts for visibility and occlusion, noting the increased complexity for load-balancing [S2, unofficial]. The approach is often compared to parallel shearwarp implementations, which divide data after the shear operation [S2, unofficial].

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
