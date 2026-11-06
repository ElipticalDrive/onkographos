# Parallel Cell Projection

Also called: Parallel Cell-Projection Implementation, Multi-threaded projection

## Overview
Parallel Cell Projection is a distributed-memory and shared-memory strategy for volume rendering that decomposes the volume into geometric primitives (cells), projects them into screen space, and rasterizes them using hardware-accelerated or software-based scan-conversion [S1, unofficial][S2][S4, S6, unofficial]. It solves the performance bottleneck of sequential cell projection by parallelizing the computationally expensive polygon generation and the subsequent image-space composition [S1, unofficial][S2][S4, unofficial]. Unlike object-order ray casting, it leverages the efficiency of rasterization pipelines, while differing from concurrent scan-conversion/composition methods by strictly separating the projection/scan-conversion phase from the global composition phase to reduce asynchronous communication overhead [S4, unofficial].

## How it works
The method operates in two primary phases: an object-space phase for projection and scan-conversion, and an image-space phase for composition [S1, S4, unofficial].

### Inputs and Outputs
*   **Inputs:** Volumetric dataset (structured or unstructured), camera parameters (view and projection matrices), transfer function (mapping scalar values to color and opacity), and system topology (number of processors/nodes).
*   **Outputs:** A 2D image (framebuffer) representing the volume rendering.

### Data Structures
*   **Vertex Array:** Stores the transformed, lit, and interpolated vertex attributes (position, color, opacity, depth) for each projected face [S1, unofficial][S2][S6, unofficial].
*   **Ray-Segment List:** A list of depth-sorted segments for each pixel or region, generated during scan-conversion and used for composition [S3, S4, unofficial].
*   **Output Buffer:** Stores ray segments or partial image data for inter-processor communication [S3, S4, unofficial].

### Steps

1.  **Partitioning:** The volume data is partitioned into regions assigned to different processors or threads. Static load balancing is commonly used, dividing the image or volume into regions [S1, S3, S4, unofficial].
2.  **Object-Space Projection (Pass 1):**
    *   Each processor/thread computes per-vertex transformations (point/normal transformation, lighting) for its assigned cells [S1, unofficial][S2][S6, unofficial].
    *   Shared vertices are processed only once by partitioning object space [S1, unofficial].
    *   For each cell, the primitive is decomposed into faces [S1, S4, unofficial].
    *   Faces are transformed into screen space [S2].
    *   **Sorting:** Faces are sorted back-to-front. Note: Using only face centroids as a sorting criterion may lead to incorrect sorting [S1, S11].
    *   **Scan-Conversion:** Sorted faces are rasterized. For each pixel covered by a face, linearly interpolated vertex color, opacity, and depth values are computed [S1, unofficial].
    *   **Partial Composition:** During scan-conversion, partial composition (e.g., using Porter-Duff's over operation) is applied to reduce the size of ray-segment lists exchanged later [S4, unofficial].
    *   Results are loaded into vertex arrays or ray-segment lists [S2][S6, unofficial].
3.  **Image-Space Composition (Pass 2):**
    *   The screen is partitioned into regions assigned to processors [S3, S4, unofficial].
    *   **Gathering:** Each processor gathers ray-segment lists from other nodes corresponding to its assigned pixels [S4, unofficial].
    *   **Merging:** Ray segments are merged into a single depth-sorted list per pixel [S4, unofficial].
    *   **Compositing:** Pixel values are calculated by compositing the depth-sorted ray segments from front to back using alpha blending (Porter-Duff over operation) [S4, unofficial].
    *   **Final Output:** Processors send their rendered image regions to a control node for final display [S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_cell_projection(volume_data, camera, transfer_function, num_processors):
    # Phase 1: Object-Space Projection
    # Partition volume data among processors
    local_cells = partition_volume(volume_data, num_processors)
    
    # Shared memory structures for multi-threaded version (e.g., SGI Onyx2)
    # vertex_arrays = []
    # sorting_thread = None
    # opengl_thread = None
    
    for cell in local_cells:
        # Per-vertex calculations: transformation, lighting
        transformed_vertices = transform_vertices(cell.vertices, camera.mvp_matrix)
        lit_vertices = apply_lighting(transformed_vertices, transfer_function)
        
        # Decompose cell into faces
        faces = decompose_into_faces(cell, lit_vertices)
        
        # Sort faces back-to-front
        # Pitfall: Do not use only centroids for sorting [S1, unofficial]
        sorted_faces = sort_faces_back_to_front(faces)
        
        # Scan-convert faces
        ray_segments = []
        for face in sorted_faces:
            # Rasterize face, interpolating color, opacity, depth
            pixel_contributions = rasterize_face(face, camera.resolution)
            ray_segments.extend(pixel_contributions)
        
        # Partial composition to reduce data size for communication [S4, unofficial]
        merged_segments = partial_compose(ray_segments)
        
        # Store results for Phase 2
        store_ray_segments(merged_segments)

    # Phase 2: Image-Space Composition
    # Partition screen among processors
    local_screen_region = partition_screen(num_processors)
    
    # Gather ray segments from all processors for local screen region
    global_ray_segments = gather_ray_segments(local_screen_region)
    
    # Merge and sort segments per pixel
    sorted_pixel_segments = merge_and_sort_segments(global_ray_segments)
    
    # Composite from front to back
    final_image = []
    for pixel in local_screen_region:
        segments = sorted_pixel_segments[pixel]
        color, opacity = 0.0, 0.0
        for segment in segments:
            # Porter-Duff over operation
            color, opacity = porter_duff_over(color, opacity, segment.color, segment.opacity)
        final_image.append((color, opacity))
    
    # Send final image region to control node [S4, unofficial]
    send_to_control_node(final_image)

def porter_duff_over(c1, a1, c2, a2):
    # Standard alpha blending
    a_out = a1 + a2 * (1 - a1)
    c_out = (c1 + c2 * a2 * (1 - a1)) / a_out if a_out > 0 else c1
    return c_out, a_out
```

### Parameters
*   **Buffer Depth:** Number of ray segments stored in output buffers before sending. Typical values tested: 25 and 200 segments [S3, unofficial].
*   **Polling Frequency:** Interval at which processors check for incoming messages. Typical values: after each cell scan conversion or after every 4 cells [S3, unofficial].
*   **Partitioner Aspect Ratio:** Configuring the partitioner to produce regions with high aspect ratios can reduce message congestion [S3, unofficial].
*   **Minimum Region Size:** Parameters to prevent regions from falling below a certain size to avoid message traffic congestion [S3, unofficial].

### Edge Cases and Pitfalls
*   **Sorting Errors:** Using only centroids of each face as a sorting criterion may lead to incorrect sorting [S1, unofficial].
*   **Message Congestion:** Spatial partitioners may generate regions with very small projected areas, causing all message traffic to bombard a few nodes, backing up output buffers and leaving most nodes idle [S3, unofficial].
*   **Idle Time:** Larger buffers and longer polling intervals reduce communication overhead but increase idle time (`send_wait`, `recv_wait`, `sync_wait`) because idle processors wait longer for new work [S3, unofficial].
*   **Hot Spots:** High cell density regions may map to few scanlines, causing load imbalance in image-space operations (`t_merge`) [S3, unofficial].

## Subtopics

### [Asynchronous Communication Task Management](3656-asynchronous-communication-task-management.md)
A strategy to manage communication costs in parallel volume rendering by allowing processors to proceed independently, multiplexing object-space cell computations with image-space ray merging, and overlapping computation with communication using end buffering. It sets itself apart from synchronous cell projection by decoupling sending and receiving tasks, thereby hiding data transfer overheads and spreading the communication load over time.

### Dynamic Load Balancing
A distributed work-stealing scheme used to handle load imbalance caused by initial data distribution, view dependency of scan conversion time, and run-time dynamic behavior of Early Ray Termination (ERT) [S4, unofficial]. It differs from preprocessing-based static load balancing (which cannot handle dynamic ERT behavior) and self-scheduling based schemes (which do not work well for non-shared memory systems like PC clusters) [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The projection phase is parallelized in object space, while the composition phase is parallelized in screen space. The complexity depends on the number of cells, pixels, and the cost of sorting and merging [S1, S4, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of ray segments stored. With 64 regions, the maximum number of unmerged ray segments stored on any processor is at most 15-20% of the total number received [S3, unofficial].
*   **Measured Performance:** On a 128-processor Intel Paragon (66.7 MHz POWER2 chipset, 34 MB/s node-to-node bandwidth), a speedup of 90 was achieved with a parallel efficiency of 70%, rendering time of 2 frames per second (excluding display) [S3, unofficial].
*   **Scaling:** Speed-up levels out due to communications overhead. On an SGI Onyx2, speed-up for tetrahedral cells leveled out after about 5 projection threads, while for general grids, it leveled out after 18 projection threads [S2, S5][S6, unofficial].
*   **Strengths:** Leverages hardware-accelerated rasterization; scalable to large clusters (16-1000+ nodes) [S2][S4, unofficial]; reduces global composition cost via partial composition [S4, unofficial].
*   **Limitations:** Communication overhead can limit scalability; load imbalance due to hot spots and view dependency; static load balancing may not handle dynamic behavior [S1, S3, S4, unofficial].
*   **When to Use:** Suitable for large-scale distributed memory systems where rasterization hardware is available or software rasterization is efficient; when the volume data is complex and benefits from geometric decomposition [S2][S4, unofficial].
*   **When Not to Use:** When communication bandwidth is very low and cannot handle the ray-segment exchange; when the volume is simple and ray casting might be more efficient [S3, S4, unofficial].

## Background
Parallel Cell Projection originated as an extension of cell projection methods to parallel architectures. Early implementations by Lucas [Lucas92] described a pure software system with two passes [S1, unofficial]. Wilhelms et al. [Wilhelms96] also used scan conversion for software rendering [S1, unofficial]. The Scalable Distributed Volume Rendering (SDVR) System [15] targeted large PC clusters using OpenGL and MPI [S2][S4, unofficial]. The method has been implemented on various architectures, including the SGI Onyx2 [S2, S5][S6, unofficial] and Intel Paragon [S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S3] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S4] PDF Cell-projection Parallel Volume Rendering With Early Ray Termination, http://www.tomitaken.org/project/papers/ReVolver/2004/VIIP.pdf, fetched 2026-10-27 (unofficial)
[S5] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S6] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
