# GPU-based Ray-casting for Unstructured Grids
Also called: GPU Ray-Casting for Unstructured Grids, Ours, Per-pixel intersection list method

## Overview
This method is a GPU-based ray-casting algorithm for volume rendering unstructured grid data that guarantees accurate results for non-convex meshes by employing per-pixel intersection lists [S1, unofficial]. It solves the difficulty of mapping irregular unstructured grid data efficiently to GPU hardware by leveraging coherent sample placement and flat array storage, avoiding the cell-ordering bottlenecks of direct ray-casting [S1, unofficial]. It sets itself apart from siblings like [CUDA Volume Ray Casting](2048-cuda-volume-ray-casting.md) by explicitly handling non-convexity through per-pixel intersection lists, and differs from [VOGUE Architecture](3149-vogue-architecture.md) by relying on a ray-casting approach rather than a voxelization-based one [G]. It builds on [Hardware and Platform Specific](2517-hardware-and-platform-specific.md) principles by optimizing work distribution for modern GPU architectures [G].

## How it works
**Inputs and Outputs:** Unstructured grid dataset (mesh topology and cell data); rendering parameters (camera, transfer functions) [G]. Outputs: Rendered image (pixel colors) [G].

**Data Structures:**
1. **Intersection Lists:** A flat array representing intersection lists for all faces, replacing linked lists to ensure coalesced memory access on the GPU [S1, unofficial].
2. **Thread Block Mapping:** Rays are grouped into thread blocks based on an extended image-tile-based work distribution method, increasing ray-coherency within a block [S1, unofficial].

**Numbered Steps:**
1. **Intersection List Construction:** Represent the intersection lists of all pixels as a flat array instead of using linked lists, because linked lists lead to irregular memory access which is inappropriate for the GPU [S1, unofficial].
2. **Work Distribution:** Group rays from an image-tile (a square image having a certain size) to improve ray-coherency in a ray group. Use a 2D thread block having the same size as an image-tile, where each thread handles one ray [S1, unofficial].
3. **Ray Casting:** Perform ray-casting on the GPU where a GPU thread handles each ray [S2, unofficial]. Each thread retrieves the intersection list for its assigned pixel from the flat array [S1, unofficial].
4. **On-Demand VDFI Calculation:** Take an on-demand View-Dependent Face Information (VDFI) computation strategy during the ray-casting process, calculating VDFI whenever a ray meets a face, rather than using thread-local buffers or preprocessing all faces [S1, unofficial].
5. **Compositing:** Composite the optical properties (color, opacity) along the ray path using the intersection data and write the final color to the output image buffer [G].

**Formulas:**
Volume rendering compositing follows the standard front-to-back alpha blending equation:
$$C_{out} = C_{in} + (1 - O_{in}) \cdot C_{cell} \cdot O_{cell}$$
$$O_{out} = O_{in} + (1 - O_{in}) \cdot O_{cell}$$
where $C_{in}$ and $O_{in}$ are the accumulated color and opacity before the current sample, $C_{cell}$ and $O_{cell}$ are the color and opacity of the current grid cell/face, and $C_{out}$ and $O_{out}$ are the updated accumulators [G].

**Parameters:**
* **Image Tile Size:** Determines the block size for thread grouping. Not given in the sources. [G]
* **Sorting Algorithm:** Used for visibility sorting. Not given in the sources. Standard distance-based sorting is assumed [G].

**Edge Cases and Pitfalls:**
* **Warp Divergence from Buffers:** Using thread-local VDFI buffers causes warp divergence: since GPU threads run in warps (e.g., 32 threads), if one or more threads in a warp fail to find the required VDFI in the buffer, the work of threads in the warp is serialized, invalidating the benefit of buffer hits for other threads and making buffer handling overhead [S1, unofficial].
* **Duplicate Computations:** The on-demand calculation strategy allows duplicate computations, but achieves better performance even though it allows duplicate computations [S1, unofficial].
* **Non-Convex Meshes:** The per-pixel intersection list concept guarantees accurate results for non-convex meshes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_kernel(pixel_x, pixel_y, intersection_lists, grid_data, transfer_function, camera):
    # 1. Retrieve intersection list for this pixel from the flat array
    ray_origin = camera.get_origin(pixel_x, pixel_y)
    ray_direction = camera.get_direction(pixel_x, pixel_y)
    intersections = intersection_lists[pixel_y * camera.width + pixel_x]

    # 2. Sort intersections by distance along the ray (front-to-back)
    intersections.sort(key=lambda inter: inter.distance)

    # 3. On-demand integration
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0

    for intersection in intersections:
        # Calculate optical properties on demand when ray meets a face
        color, opacity = compute_cell_properties(intersection, grid_data, transfer_function)

        # Standard volume rendering compositing
        accumulated_color = accumulated_color + (1.0 - accumulated_opacity) * color * opacity
        accumulated_opacity = accumulated_opacity + (1.0 - accumulated_opacity) * opacity

        if accumulated_opacity >= 1.0:
            break

    return accumulated_color

def compute_cell_properties(intersection, grid_data, transfer_function):
    # Sample grid data at intersection point and apply transfer function
    # Returns (color, opacity)
    return transfer_function(grid_data.sample(intersection.point))
```

## Subtopics

### [Pixel Ray Image (PRI)](2467-pixel-ray-image-pri.md)
The Pixel Ray Image (PRI) is a semi-regular data structure for volume rendering of unstructured grids that replaces the irregular mesh with a pixelized projection plane and 1D linear splines defined on sampling rays. It solves the difficulty of mapping irregular unstructured grid data efficiently to GPU hardware by leveraging coherent sample placement and 2D texture storage, avoiding the cell-ordering bottlenecks of direct ray-casting. Unlike sibling methods like [GPU-based Ray Traversal on Tetrahedral Meshes](3165-gpu-based-ray-traversal-on-tetrahedral_meshes.md) which traverse the mesh directly, PRI pre-computes a texture-based representation. Choose it when memory bandwidth and texture sampling coherence are prioritized over explicit mesh traversal.

### [GPU-based Ray Traversal on Tetrahedral Meshes](3165-gpu-based-ray-traversal-on-tetrahedral-meshes.md)
This method implements volume rendering on tetrahedral meshes by traversing a ray through the mesh structure on the GPU, determining the exit face of the current tetrahedron and following neighbor pointers to adjacent cells. It distinguishes itself from CPU-based traversal by storing all mesh and traversal data in GPU texture memory, enabling concurrent, single-pass traversal for all pixels without CPU-GPU data transfer. Unlike sibling methods that may use hierarchical structures or voxelization, this approach relies on explicit tetrahedral adjacency and convexity assumptions. Choose it when exact geometric traversal on tetrahedral grids is required without preprocessing into texture-based projections.

## Operating characteristics
* **Time and Memory Complexity:** Not given in the sources. [G]
* **Performance:** Achieves up to 36.5 times higher performance for the ray-casting process and 18.1 to 19.7 times higher performance for the whole volume rendering process compared with the Bunyk algorithm using a CPU core [S1, unofficial][S2, unofficial].
* **Comparison to Cell Projection:** Shows up to 8.2 times higher performance than a GPU-based cell projection method (HAVS) while generating more accurate rendering results [S1, unofficial][S2, unofficial].
* **Cache and Memory Efficiency:** On-demand calculation reduces memory access by up to 1.5 times and L2 cache transactions by up to 4.5 times, while increasing L1 cache hit ratio by about 18% compared to VDFI buffers [S1, unofficial]. Ray-casting with the image-tile distribution method took 21% less L2 cache transactions than using a naive pixel-order ray distribution approach [S1, unofficial].
* **Strengths:** Guarantees accurate results for non-convex meshes; efficient memory access via flat array intersection lists; high performance on modern GPUs due to on-demand calculation and tile-based work distribution [S1, unofficial].
* **Limitations:** Allows duplicate computations due to the on-demand strategy; requires preprocessing to construct intersection lists [S1, unofficial].
* **Scaling:** Performance scales with GPU hardware capabilities; the method was tested on three unstructured grid datasets with different characteristics [S2, unofficial].

## Background
The method was introduced by Gibeom Gu and Duksu Kim at KISTI and KOREATECH in the EUROVIS 2019 poster 'Accurate and Memory-Efficient GPU Ray-Casting Algorithm for Volume Rendering Unstructured Grid Data' [S1, unofficial]. It was later published in the *ETRI Journal* in 2020 [S2, unofficial]. It extends prior image-tile-based work distribution methods to fit modern GPU architectures [S2, unofficial]. It addresses limitations in previous GPU-based approaches, such as the inefficiency of per-thread local buffers and the inaccuracy of cell projection methods for non-convex geometries [S1, unofficial]. The work is part of a broader effort to optimize volume rendering on unstructured data using GPU shaders, CUDA, and ray tracing techniques [S1, unofficial].

## Sources
[S1] Accurate and Memory-Efficient GPU Ray-Casting Algorithm for Volume Rendering Unstructured Grid Data, https://doi.org/10.2312/eurp.20191150, fetched 2026-10-26 (unofficial)
[S2] Accurate and efficient GPU ray‐casting algorithm for volume rendering of unstructured grid data, https://www.researchgate.net/publication/339158888_Accurate_and_efficient_GPU_ray_casting_algorithm_for_volume_rendering_of_unstructured_grid_data, fetched 2026-10-28 (unofficial)
