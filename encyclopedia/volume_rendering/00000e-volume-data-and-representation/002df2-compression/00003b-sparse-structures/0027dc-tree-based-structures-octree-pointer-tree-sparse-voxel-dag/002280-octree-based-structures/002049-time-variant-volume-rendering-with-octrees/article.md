# Time-Variant Volume Rendering with Octrees

Also called: Multi-timestep rendering, Time-Variant Volume Rendering with Octree Compression

## Overview
This method enables the interactive rendering of time-variant volume datasets by storing multiple timesteps in limited CPU main memory using octree compression, thereby overcoming GPU memory capacity and bus bandwidth limitations [S1, S2]. It sets itself apart from standard sparse structures by allowing full sequences of medium-sized time-variant data to reside in main memory for interactive exploration [S1, S2]. Unlike baseline renderers that process full uncompressed volumes, this approach permits residency of large datasets on commodity hardware [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of $T$ timesteps of 3D volume data, where each timestep is a regular grid (e.g., $302^3$) [S1, S2].
*   **Output:** A compressed octree representation of the entire sequence stored in CPU memory, and rendered frames at interactive frame rates [S1, S2].

**Data Structures:**
The method relies on the **Octree** structure defined in [Octree-based structures](2792-octree-based-structures.md). Each node in the octree represents a sub-volume. The compression is applied to the entire sequence of timesteps to fit them into main memory [S1, S2]. The specific encoding of differences between timesteps is detailed in [Octree-Difference Encoding for Time-Varying Volume Rendering](5850-octree-difference-encoding-for-time-varying-volume-rendering.md).

**Steps:**
1.  **Compression:** Compress the full sequence of $T$ timesteps into a single octree structure [S1, S2].
2.  **Memory Residency:** Store the compressed octree in CPU main memory, allowing all $T$ timesteps to reside simultaneously [S1, S2].
3.  **Rendering:** Perform interactive isosurface ray tracing on the compressed octree [S1, S2].
4.  **Time Step Selection:** Select a specific timestep $t$ from the compressed structure for rendering [S1, S2].

**Formulas:**
*   Not given in the sources.

**Parameters:**
*   **Isovalue:** The threshold value for isosurface extraction. Example: 42 for the heptane simulation [S1, S2].
*   **Resolution:** The rendering resolution. Examples: $512^2$ and $1024^2$ pixel frames [S1, S2].
*   **Time Step Interval:** The interval between rendered timesteps. Examples: 25, 50, 75, 100, 125, 150 [S1, S2].

**Edge Cases and Pitfalls:**
*   **GPU Memory Limitations:** Without compression, GPU memory restricts the number of timesteps that can be stored in-core [S1, S2].
*   **Bus Bandwidth:** Streaming uncompressed textures from CPU to GPU is inefficient due to bus bandwidth limitations [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_time_variant_volume(compressed_octree, timestep_index, isovalue, resolution):
    """Renders a specific timestep from a compressed time-variant octree."""
    # 1. Access the octree root for the requested timestep.
    octree_root = compressed_octree.get_timestep_root(timestep_index)
    
    # 2. Initialize image buffer.
    image = [0.0] * (resolution * resolution)
    
    # 3. Cast rays for each pixel and traverse the octree.
    for y in range(resolution):
        for x in range(resolution):
            # Standard ray generation for a pixel [G]
            ray_o = [x / resolution - 0.5, y / resolution - 0.5, -1.0]
            ray_d = [0.0, 0.0, 1.0]
            # Traverse octree for isosurface extraction [G]
            color = trace_octree(octree_root, ray_o, ray_d, isovalue)
            image[y * resolution + x] = color
    return image

def trace_octree(node, origin, direction, isovalue):
    """Recursive octree traversal for ray intersection [G]."""
    if node is None or not box_intersects_ray(node.bounding_box, origin, direction):
        return 0.0
    if node.is_leaf():
        return sample_voxel(node.data, origin, direction, isovalue)
    # Traverse children in front-to-back order
    for child in node.children:
        color = trace_octree(child, origin, direction, isovalue)
        if color > 0.0:
            return color
    return 0.0

def box_intersects_ray(box, origin, direction):
    """Standard axis-aligned bounding box ray intersection test [G]."""
    t_min = (box.min - origin) / direction
    t_max = (box.max - origin) / direction
    t1 = min(t_min); t2 = max(t_min)
    t3 = min(t_max); t4 = max(t_max)
    if t1 > t3: t1, t3 = t3, t1
    if t2 > t4: t2, t4 = t4, t2
    if max(t1, t2) <= min(t3, t4):
        return True
    return False

def sample_voxel(data, origin, direction, isovalue):
    """Standard voxel sampling for isosurface extraction [G]."""
    # Evaluate density/value along ray segment within voxel
    # Return color or alpha if isovalue is crossed
    return 0.0
```

## Subtopics

### [Octree-Difference Encoding for Time-Varying Volume Rendering](5850-octree-difference-encoding-for-time-varying-volume-rendering.md)
This method compresses time-varying volume data using quantization and octree-difference encoding to improve rendering rates. It sets itself apart from standard sparse structures by explicitly exploiting temporal coherence: identical subtrees at consecutive time steps are merged, and the rendering engine is restructured to perform selective rendering based on this merged tree representation. It is chosen when reducing I/O costs and inter-frame delay is critical for large time-variant datasets.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The compressed size is significantly smaller than the uncompressed size. For the Utah CSAFE heptane simulation, the uncompressed size is 4.11 GB, while the compressed octree size is 678 MB [S1, S2]. This allows the entire dataset to fit in limited CPU main memory [S1, S2].
*   **Time:** Rendering time depends on the complexity of the octree and the resolution. The method scales well with the number of threads [S1, S2].

**Measured Performance:**
*   **Hardware:**
    *   Core Duo 2.16 GHz laptop [S1, S2].
    *   NUMA 16-core 2.4 GHz Opteron workstation [S1, S2].
*   **Dataset:** Utah CSAFE heptane simulation, $302^3$ volume, 153 timesteps [S1, S2].
*   **Frame Rates (Core Duo 2.16 GHz, $512^2$):**
    *   Timestep 25: 17.0 fps [S1, S2]
    *   Timestep 50: 11.1 fps [S1, S2]
    *   Timestep 75: 5.7 fps [S1, S2]
    *   Timestep 100: 4.1 fps [S1, S2]
    *   Timestep 125: 3.5 fps [S1, S2]
    *   Timestep 150: 3.2 fps [S1, S2]
*   **Frame Rates (NUMA 16-core 2.4 GHz, $512^2$):**
    *   Timestep 25: 87.1 fps [S1, S2]
    *   Timestep 50: 60.3 fps [S1, S2]
    *   Timestep 75: 36.7 fps [S1, S2]
    *   Timestep 100: 26.6 fps [S1, S2]
    *   Timestep 125: 28.0 fps [S1, S2]
    *   Timestep 150: 23.1 fps [S1, S2]
*   **Frame Rates (NUMA 16-core 2.4 GHz, $1024^2$):**
    *   Timestep 25: 29.2 fps [S1, S2]
    *   Timestep 50: 18.0 fps [S1, S2]
    *   Timestep 75: 9.6 fps [S1, S2]
    *   Timestep 100: 6.6 fps [S1, S2]
    *   Timestep 125: 7.1 fps [S1, S2]
    *   Timestep 150: 6.3 fps [S1, S2]

**Scaling:**
The technique scales well with the number of threads. On a 2.4 GHz Opteron NUMA workstation, scalability was tested on 1, 2, 4, 8, 12, and 16 threads. The slight change in slope at 8 threads corresponds to the use of local NUMA memory by two cores instead of one. The octree technique scales as well as the hierarchical grid with uncompressed data [S1, S2].

**Strengths and Limitations:**
*   **Strengths:** Overcomes GPU memory and bus bandwidth limitations; allows interactive rendering of large time-variant datasets on commodity hardware; reduces I/O costs and inter-frame delay [S1, S2].
*   **Limitations:** Requires CPU main memory for storage; performance degrades with higher resolutions and more complex timesteps [S1, S2].

**When to Use:**
*   When GPU memory is insufficient to store all timesteps of a time-variant volume.
*   When bus bandwidth limits the streaming of uncompressed data.
*   When interactive frame rates are required for large datasets.

**When Not to Use:**
*   When GPU memory is sufficient to store all timesteps uncompressed.
*   When the dataset is too small to benefit from compression overhead.

## Background
**Origin:**
The method was introduced in the paper "Interactive Isosurface Ray Tracing of Large Octree Volumes" by Wald et al. [S1, S2].

**History and Development:**
The method was developed to address the limitations of GPU volume rendering for time-variant volumes. It leverages octree compression to store multiple timesteps in CPU main memory, enabling interactive rendering on commodity hardware [S1, S2]. The underlying implementation utilizes a dynamically load-balanced, tile-based parallelization scheme [S1, S2].

**Where it is Used:**
*   Visualization of large scientific datasets, such as the Utah CSAFE heptane simulation [S1, S2].
*   Interactive exploration of time-variant volume data on machines with limited main memory [S1, S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
