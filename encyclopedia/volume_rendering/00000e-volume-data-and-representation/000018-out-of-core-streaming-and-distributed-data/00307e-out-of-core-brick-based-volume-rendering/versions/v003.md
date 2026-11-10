# Out-of-core brick-based volume rendering

Also called: Brick pyramid rendering, Volume virtualization

## Overview
Out-of-core brick-based volume rendering is a technique that streams multi-GB volumetric datasets over HTTP and renders them interactively with a fixed VRAM budget by decoupling logical 3D grid coordinates from physical GPU memory using a brick pyramid and indirection texture [S1, unofficial]. It solves the problem of rendering datasets too large to fit in memory or download monolithically by fetching only the visible bricks via HTTP range reads [S1, unofficial]. It sets itself apart from [Sectioning](1636-sectioning.md) by using fixed-size spatial chunks (bricks) rather than view-dependent slabs, and from [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) by relying on a software-managed page cache and indirection texture rather than hardware texture streaming features [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source volumetric dataset (e.g., 16-bit raw, OME-Zarr) and a camera view (position, orientation, field of view).
*   **Output:** A rendered 2D image (framebuffer) and a fixed-size GPU page cache containing active bricks.

**Data Structures:**
1.  **Brick Pyramid:** A multi-resolution hierarchy. The finest level contains the full-resolution data. Coarser levels are downsampled versions of the previous level. All levels are split into fixed-size chunks called bricks [S1, unofficial].
    *   **Logical Brick Size:** $64^3$ voxels.
    *   **Physical Brick Size:** $66^3$ voxels (to facilitate interpolation during sampling) [S1, unofficial].
    *   **Storage:** Compressed, stored server-side in structured binary blobs [S1, unofficial].
2.  **Indirection Texture:** A compact 2D or 3D texture storing cache slot indices. Each texel corresponds to a logical brick coordinate in the pyramid and points to its current physical location (slot index) in the GPU page cache [S1, unofficial].
3.  **GPU Page Cache (Atlas):** A fixed-size array of slots holding decompressed bricks.
    *   **Default Size:** 512 slots.
    *   **Scalable Size:** Up to $768^3$ or $1024^3$ slots for larger volumes or high-DPI screens [S1, unofficial].
    *   **VRAM Footprint:** Approximately 300 MB at 512 slots for 16-bit data, remaining fixed regardless of total dataset size [S1, unofficial].

**Algorithm Steps:**

1.  **Pre-processing:**
    *   Split the source dataset into fixed-size bricks ($64^3$ logical, $66^3$ physical).
    *   Generate downsampled LOD levels to create the brick pyramid.
    *   Compress the bricks.
    *   Store the pyramid server-side [S1, unofficial].

2.  **Runtime Initialization:**
    *   Initialize the GPU page cache with empty slots.
    *   Initialize the indirection texture (mapping logical coordinates to cache slots, initially invalid/null) [S1, unofficial].

3.  **View Update and Fetching:**
    *   Identify bricks relevant to the current camera view.
    *   For each relevant brick not currently in the cache:
        *   Fetch the compressed brick data from the server using HTTP range reads [S1, unofficial].
        *   Decompress the brick asynchronously in dedicated worker threads [S1, unofficial].
        *   Upload the decompressed brick to the GPU page cache.
        *   Update the indirection texture: map the logical brick coordinate to the new cache slot index [S1, unofficial].
    *   **Cache Eviction:** If the cache is full, evict the Least Recently Used (LRU) brick. Update the indirection texture to mark the old logical coordinate as unmapped or pointing to a new slot if re-fetched [S1, unofficial].

4.  **Rendering (Compute Shader Ray Marching):**
    *   For each pixel in the output image:
        *   Cast a ray through the volume.
        *   For each sample point along the ray:
            *   Calculate the logical brick coordinate $(B_x, B_y, B_z)$ containing the sample point.
            *   Sample the **Indirection Texture** at $(B_x, B_y, B_z)$ to retrieve the cache slot index $S$ [S1, unofficial].
            *   If $S$ is valid:
                *   Calculate the local coordinates $(L_x, L_y, L_z)$ within the brick.
                *   Perform density lookup and compositing using the data in cache slot $S$ at local coordinates $(L_x, L_y, L_z)$ [S1, unofficial].
            *   If $S$ is invalid (brick not in cache):
                *   Trigger a fetch for the brick (if not already pending) and skip compositing for this sample or use a placeholder/coarse LOD value [G]. *Note: The source implies the system handles availability, but does not specify the exact fallback for a cache miss during the critical path other than the indirection lookup. A robust implementation would stall or use a coarse proxy.*
        *   Composite the accumulated color and opacity for the pixel [S1, unofficial].

**Parameters:**
*   **Brick Logical Size:** $64^3$ voxels [S1, unofficial].
*   **Brick Physical Size:** $66^3$ voxels [S1, unofficial].
*   **Cache Slots:** 512 (default), scalable to $768^3$ or $1024^3$ [S1, unofficial].
*   **Render Scale:** May need reduction (e.g., 0.25) on older/mobile GPUs to maintain 60 FPS [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bandwidth Limitation:** Volume ray marching is texture bandwidth intensive. Older or mobile GPUs may drop below 60 FPS at full resolution [S1, unofficial].
*   **Pre-processing Requirement:** Raw volume files cannot be rendered directly; they must be pre-processed and compressed, adding an offline ingestion step and increasing total storage requirements by ~15–20% [S1, unofficial].
*   **Network Dependency:** User experience is tied to internet connection quality. Lack of smart prefetching or offline capabilities means performance degrades with poor connectivity [S1, unofficial].
*   **Scalability Limit:** Tested up to ~$2048^3$ voxels. Larger datasets require manual adjustment of indirection table dimensions and page cache size [S1, unofficial].

## Subtopics

### [I/O-Conscious Volume Rendering](6929-i-o-conscious-volume-rendering.md)
I/O-Conscious Volume Rendering is an out-of-core rendering technique that minimizes end-to-end rendering time by overlapping disk I/O with rendering computation using a data-driven execution model where data blocks are accessed as indivisible units. It sets itself apart from Out-of-core brick-based volume rendering by tightly integrating algorithm-specific prefetching with the ray casting traversal to ensure data is available exactly when needed, rather than relying on general-purpose disk prefetching or fixed memory budgets. Unlike program-driven approaches where the renderer controls the I/O, this method uses a dedicated prefetch thread that mirrors the ray caster's traversal logic to hide latency.

### Out-of-core bricking
Out-of-core bricking is a technique to create bricks of volumetric data along with a directed acyclic graph (DAG) of the relative distance of each brick along a single view direction. Bricks are aligned with cropped subvolumes [S2, unofficial]. A heuristic is used to evaluate the coherency of a series of data slices, which are parsed into a set of bricks sized for rendering hardware [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Performance depends on network latency, decompression speed, and GPU texture bandwidth. First render times for 2-3 GB datasets range from 544 ms to 2162 ms depending on hardware and dataset density [S1, unofficial].
*   **Memory Complexity:** Fixed $O(1)$ VRAM usage determined by cache slot count, independent of the total volumetric dataset size [S1, unofficial].
*   **Measured Performance:**
    *   **Chameleon Dataset (1024×1024×1080, 2160 MB):** First render on M2: 544 ms, Zenfone: 655 ms. Data streamed: M2 0.73 MB, Zenfone 0.77 MB. Atlas occupancy: 2/512 (0.4%) [S1, unofficial].
    *   **Beechnut Dataset (1024×1024×1546, 3092 MB):** First render on M2: 1079 ms, Zenfone: 2162 ms. Data streamed: M2 1.10 MB, Zenfone 2.19 MB. Atlas occupancy: 18/512 (4%) [S1, unofficial].
*   **Strengths:** Fixed VRAM footprint; interactive frame rates for multi-GB datasets; leverages standard HTTP range reads; decouples logical and physical memory.
*   **Limitations:** Requires pre-processing; storage overhead ~15-20%; dependent on network quality; no smart prefetching; limited tested scale (~2048³); texture bandwidth bound on GPUs.
*   **When to use:** Interactive web-based visualization of large bioimaging or scientific datasets where VRAM is constrained and network bandwidth is sufficient.
*   **When not to use:** Offline scenarios; datasets requiring immediate full-volume access without streaming latency; environments with very low-end GPUs lacking WebGPU support.

## Background
The method is implemented in the **Kiln** project, a WebGPU-native out-of-core rendering system for large virtualized volumetric data [S1, unofficial]. Kiln was developed by Martin Panknin and the Paleoanthropology group and Visualization and MultiMedia Lab, University of Zurich [S1, unofficial]. It was accepted into the OME-NGFF ecosystem (v0.2.1+) as a recognized viewer for OME-Zarr datasets [S1, unofficial]. The technique adapts out-of-core principles from geospatial applications and game engines (virtual textures) to volumetric data in browser environments [S1, unofficial].

## Sources
[S1] Kiln: WebGPU-Native Out-of-Core Volume Rendering for Multi-GB Datasets, https://dev.to/mpanknin/kiln-webgpu-native-out-of-core-volume-rendering-for-multi-gb-datasets-2alb, fetched 2026-10-26 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
