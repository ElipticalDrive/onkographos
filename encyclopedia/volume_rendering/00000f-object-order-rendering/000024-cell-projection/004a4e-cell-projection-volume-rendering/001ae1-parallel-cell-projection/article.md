# Parallel Cell Projection

Also called: Parallel Cell-Projection Implementation, Multi-threaded projection

## Overview
Parallel Cell Projection is a parallelization strategy for Cell-Projection Volume Rendering that distributes the computationally expensive task of software-based polygon generation and the subsequent image-space composition across multiple processors or threads [S1, S3, unofficial][S5]. It solves the bottleneck of rendering large unstructured grids by decomposing the workload into an object-space phase (projection/scan conversion) and an image-space phase (ray merging/composition) [S1, S4, unofficial]. It sets itself apart from synchronous alternatives by employing asynchronous communication and task management to hide data transfer overheads, and from concurrent scan-conversion/composition approaches by strictly separating these phases to reduce inter-processor communication frequency [S1, S3, S4, unofficial].

## How it works
The method operates on a distributed-memory architecture or shared-memory multiprocessors, typically involving $N$ working nodes (WNs) for computation and one control node (CN) for global management and final image output [S4, unofficial]. The algorithm proceeds in two distinct passes [S1, S4, unofficial].

### Inputs and Outputs
*   **Input:** A 3D volume dataset (structured or unstructured) and viewing parameters [G].
*   **Output:** A 2D rendered image (color and opacity) for each pixel [G].

### Data Structures
*   **Ray-Segment List:** A list of depth-sorted segments for a specific pixel or region, containing color, opacity, and depth values [S4, unofficial].
*   **Vertex Array:** Stores the projected polygonal information generated during the projection phase [S5][S6, unofficial].
*   **Partition Regions:** The image or volume is divided into regions assigned to processors via static load balancing [S3, unofficial].

### Numbered Steps

**Phase 1: Object-Space Projection and Scan Conversion** [S1, S4, unofficial]
1.  **Partitioning:** The dataset is partitioned into load-balanced chunks assigned to WNs [S2][S4, unofficial].
2.  **Per-Vertex Calculation:** Each WN performs per-vertex calculations, including point/normal transformation and lighting [S1, unofficial]. Shared vertices are processed only once by partitioning object space [S1, unofficial].
3.  **Polygon Generation:** Several threads (on shared-memory systems like SGI Onyx2) or WNs (on distributed systems) compute polygons in software for their assigned cells [S5][S6, unofficial].
4.  **Vertex Array Loading:** The resulting polygonal information is loaded into vertex arrays [S5][S6, unofficial].
5.  **Sorting:** One dedicated thread sorts the faces back-to-front [S1, unofficial][S5]. Using only face centroids as a sorting criterion may lead to incorrect sorting [S1, unofficial].
6.  **Scan Conversion:** The sorted faces are scan-converted, containing linearly interpolated vertex color, opacity, and depth value for each pixel in the projection [S1, unofficial].
7.  **Partial Composition:** Porter-Duff's over operation composites segments during scan conversion to reduce the size of ray-segment lists that need to be exchanged [S4, unofficial].
8.  **Asynchronous Communication:** Processors send their generated ray-segment lists to other nodes. A feedback mechanism is used: if sending is blocked due to full buffers, the processor switches to the ray merging task and begins receiving, regardless of the polling interval [S3, unofficial].

**Phase 2: Image-Space Parallel Composition** [S4, unofficial]
1.  **Region Assignment:** Computation for a specific region of the screen is assigned to each WN [S4, unofficial].
2.  **Gathering:** For each pixel in the assigned region, each node gathers ray-segment lists from other nodes corresponding to that pixel [S4, unofficial].
3.  **Merging:** The gathered lists are merged into a single depth-sorted ray-segment list [S4, unofficial].
4.  **Compositing:** The pixel value is calculated by compositing the depth-sorted ray-segments from front to back using Porter-Duff's over operation [S4, unofficial].
5.  **Final Output:** Each WN sends its rendered image region to the control node to output the overall volume rendered image [S4, unofficial]. Binary-Swap Image Composition (BSC) may be adopted to reduce communication overhead and load imbalance during this global composition phase [S4, unofficial].

### Formulas
*   **Compositing:** The color $C$ and opacity $\alpha$ of a pixel are updated using Porter-Duff's over operation for each segment $i$ with color $C_i$ and opacity $\alpha_i$:
    $$ \alpha_{out} = \alpha_{in} + \alpha_i (1 - \alpha_{in}) $$
    $$ C_{out} = C_{in} + C_i (1 - \alpha_{in}) $$
    [S4, unofficial]

### Parameters
*   **Buffer Depth:** The number of ray segments stored in output buffers before sending. Typical values tested are 25 and 200 segments [S3, unofficial].
*   **Polling Frequency:** The interval at which processors check for incoming messages. Typical values tested are after each cell scan conversion or after every 4 cells [S3, unofficial].
*   **Partitioner Aspect Ratio:** Configuring the partitioner to produce regions with high aspect ratios results in larger footprints in image space to reduce congestion [S3, unofficial].
*   **Minimum Region Size:** Parameters should be adjusted so regions are not allowed to fall below a certain size to prevent message traffic congestion [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_cell_projection(volume_data, view_matrix, screen_width, screen_height, num_nodes):
    # Phase 1: Object-Space Projection and Scan Conversion
    node_chunks = [volume_data[i::num_nodes] for i in range(num_nodes)]
    local_ray_segments = {}  # Maps pixel_id to list of segments

    for chunk in node_chunks:
        # 1. Per-vertex calculation
        transformed_vertices = [apply_view_transform(v, view_matrix) for v in chunk.vertices]
        # 2. Polygon generation
        polygons = [create_face(v) for v in transformed_vertices]
        # 3. Sorting (back-to-front)
        sorted_polygons = sorted(polygons, key=lambda p: sum(v.depth for v in p.vertices) / len(p.vertices), reverse=True)
        # 4. Scan conversion and partial composition
        for poly in sorted_polygons:
            segments = rasterize_face(poly, screen_width, screen_height)
            for seg in segments:
                pid = seg.pixel_id
                if pid not in local_ray_segments:
                    local_ray_segments[pid] = []
                # Partial composition to reduce data size
                local_ray_segments[pid] = merge_porter_duff(local_ray_segments[pid], [seg])

    # 5. Asynchronous communication
    all_ray_segments = communicate_and_merge(local_ray_segments, num_nodes)

    # Phase 2: Image-Space Parallel Composition
    node_regions = partition_screen(screen_width, screen_height, num_nodes)
    local_image = {}

    for pixel_id in node_regions[current_node]:
        segments = all_ray_segments.get(pixel_id, [])
        # 6. Compositing front-to-back
        color, opacity = composite_segments(segments)
        local_image[pixel_id] = (color, opacity)

    # 7. Final output
    send_image_to_control_node(local_image)
    return local_image

def apply_view_transform(vertex, view_matrix):
    return [view_matrix @ vertex[i] for i in range(3)]

def create_face(vertices):
    return {'vertices': vertices}

def rasterize_face(face, width, height):
    segments = []
    for y in range(height):
        for x in range(width):
            if point_in_face((x, y), face):
                color, opacity, depth = interpolate_face((x, y), face)
                segments.append({'pixel_id': y * width + x, 'color': color, 'opacity': opacity, 'depth': depth})
    return segments

def merge_porter_duff(existing, new):
    merged = list(existing)
    for seg in new:
        if merged:
            last = merged[-1]
            alpha_out = last['opacity'] + seg['opacity'] * (1 - last['opacity'])
            color_out = last['color'] + seg['color'] * (1 - last['opacity'])
            merged[-1] = {'pixel_id': last['pixel_id'], 'color': color_out, 'opacity': alpha_out, 'depth': seg['depth']}
        else:
            merged.append(seg)
    return merged

def communicate_and_merge(local_segments, num_nodes):
    all_segments = {}
    for pid, segs in local_segments.items():
        if pid not in all_segments:
            all_segments[pid] = []
        all_segments[pid].extend(segs)
    for pid in all_segments:
        all_segments[pid] = merge_porter_duff([], all_segments[pid])
    return all_segments

def partition_screen(width, height, num_nodes):
    h_per_node = height // num_nodes
    regions = []
    for i in range(num_nodes):
        y_start = i * h_per_node
        y_end = (i + 1) * h_per_node if i < num_nodes - 1 else height
        regions.append([y * width + x for y in range(y_start, y_end) for x in range(width)])
    return regions

def composite_segments(segments):
    color = (0.0, 0.0, 0.0)
    opacity = 0.0
    for seg in segments:
        alpha = seg['opacity']
        color = color + seg['color'] * (1 - opacity)
        opacity = opacity + alpha * (1 - opacity)
    return color, opacity

def send_image_to_control_node(image):
    pass
```

### Edge Cases and Pitfalls
*   **Sorting Errors:** Using only centroids of each face as a sorting criterion may lead to incorrect sorting [S1, unofficial].
*   **Message Congestion:** The spatial partitioner may generate regions with very small projected areas. This causes all message traffic for that region to bombard a few nodes, backing up output buffers and leaving most nodes idle [S3, unofficial].
*   **Idle Time:** Larger buffers and longer polling intervals reduce sending/receiving/polling overhead but increase idle time (`send_wait`, `recv_wait`, `sync_wait`) because idle processors wait longer for new work [S3, unofficial].
*   **Hot Spots:** High cell density regions map to few scanlines, causing load imbalance in image-space operations (ray merging) [S3, unofficial].

## Subtopics

### [Asynchronous Communication Task Management](3656-asynchronous-communication-task-management.md)
Asynchronous Communication Task Management is a strategy for managing communication costs in parallel cell-projection volume rendering by allowing processors to proceed independently, multiplexing object-space cell computations with image-space ray merging, and overlapping computation with communication using end buffering. It sets itself apart from synchronous alternatives by decoupling sending and receiving tasks, thereby hiding data transfer overheads and spreading the communication load over time.

### [Asynchronous Communication Task Management](3656-asynchronous-communication-task-management.md)
Parallel AMR Rendering with Subdivision Strategies is a framework for parallel AMR rendering that reimplements cell projection, sorts cells, and uses kD-tree subdivision in object space with various weighting strategies. It sorts cells as described by Williams, Max & Stein, subdivides in object space with a kD-tree, and subdivides the first hierarchy level using uniform blocks (equal size) or weighted blocks (similar computational effort).

## Operating characteristics
*   **Time Complexity:** The object-space computations (viewing transformations and scan conversion) are well-balanced [S3, unofficial]. The image-space operations (ray merging) show a distinct pattern with potential load imbalance due to "hot spots" [S3, unofficial].
*   **Memory Complexity:** With 64 regions, the maximum number of unmerged ray segments stored on any processor is at most 15-20% of the total number received [S3, unofficial].
*   **Measured Performance:** On a 66.7 MHz POWER2 chipset with 34 MB/s node-to-node bandwidth, using 128 processors, the system achieved a speedup of 90, a parallel efficiency of 70%, and a rendering time of 2 frames per second (excluding display) [S3, unofficial].
*   **Scaling:** On an SGI Onyx2, speed-up for Shirley-Tuchman projection leveled out after about 5 projection threads, while for general grids, it leveled out after 18 projection threads [S5][S6, unofficial]. This leveling out is attributed to communications overhead [S5][S6, unofficial].
*   **Strengths:** Scalability for large datasets (16-1000+ nodes); ability to run with or without graphics hardware; high performance on distributed-memory architectures [S2][S3, unofficial].
*   **Limitations:** Load imbalance due to hot spots; communication overhead limits speedup; static load balancing may degrade efficiency with large processor counts [S3, unofficial].
*   **When to Use:** Large unstructured grids on distributed-memory clusters where scalability is prioritized over peak per-node performance [S2][S3, unofficial].
*   **When Not to Use:** Small datasets where communication overhead dominates; systems with shared memory where simpler parallelization suffices; scenarios requiring dynamic load balancing for highly adaptive grids [S3, S4, unofficial].

## Background
Parallel Cell Projection originated from early parallelization efforts for cell-projection volume rendering [S1, unofficial]. Lucas [Lucas92] suggested a pure software system that renders volumes in two passes, handling all major types of topologies [S1, unofficial]. Wilhelms et al. [Wilhelms96] also used scan conversion for their software renderer [S1, unofficial]. The Scalable Distributed Volume Rendering (SDVR) System [15] targeted large PC clusters, aiming for scalability where rendering time remains constant as data set size increases with additional nodes [S2][S6, unofficial]. The method has been implemented on various architectures, including SGI Onyx2 and Intel Paragon [S3, unofficial][S5][S6, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Volume rendering for curvilinear and unstructured grids, https://doi.org/10.1109/cgi.2003.1214468, fetched 2026-10-24
[S3] DTIC ADA329328: A Scalable Parallel Cell-Projection Volume Rendering Algorithm for Three-Dimensional Unstructured Data, https://archive.org/details/DTIC_ADA329328, fetched 2026-10-26 (unofficial)
[S4] PDF Cell-projection Parallel Volume Rendering With Early Ray Termination, http://www.tomitaken.org/project/papers/ReVolver/2004/VIIP.pdf, fetched 2026-10-27 (unofficial)
[S5] Volume Rendering for Curvilinear and Unstructured Grids, https://www.sci.utah.edu/~csilva/papers/cgi2003.pdf, fetched 2026-10-27
[S6] Volume rendering for curvilinear and unstructured grids, https://www.researchgate.net/publication/4023713_Volume_rendering_for_curvilinear_and_unstructured_grids, fetched 2026-10-30 (unofficial)
[S7] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
