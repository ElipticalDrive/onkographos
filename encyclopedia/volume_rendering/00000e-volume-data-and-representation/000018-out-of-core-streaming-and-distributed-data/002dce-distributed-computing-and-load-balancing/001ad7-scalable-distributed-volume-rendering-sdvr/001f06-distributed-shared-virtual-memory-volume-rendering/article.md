# Distributed Shared Virtual Memory Volume Rendering

Also called: DSVM Volume Rendering

## Overview
Distributed Shared Virtual Memory (DSVM) Volume Rendering is a method for rendering tera-scale volumes on GPU clusters by mapping the entire volume into a single shared 3D virtual memory space accessible by all nodes [S2]. It solves the problem of managing massive datasets that exceed the physical memory of individual GPUs by using on-demand paging of small voxel blocks, allowing multiple nodes to render disjoint parts of the volume simultaneously without explicit data partitioning or load balancing [S1, unofficial][S2]. It differs from sibling methods like Distributed Volume Ray Tracing with Global Kd-Tree by relying on a flat virtual memory address space with hardware-assisted page fault handling rather than a distributed spatial index, and by targeting small GPU clusters where implicit load balancing via interleaved distribution units suffices [S2].

## How it works
**Inputs and Outputs:**
- Input: A large 3D volume dataset (e.g., 955 GB EM data) stored on disk or a file server; current camera view parameters (position, orientation, zoom); cluster configuration (number of GPU nodes).
- Output: A composited 2D image rendered on the client.

**Data Structures:**
1.  **Virtual Octree:** A multi-resolution hierarchy representing the volume. Each node in the octree corresponds to a block of $32^3$ voxels [S2]. The octree is not stored as a traditional tree in memory but is mapped to a large shared virtual memory address space [S2].
2.  **Virtual Distribution Unit (VDU):** A large cubical region of voxels (e.g., $1024^3$) assigned to a specific GPU node [S2]. VDUs are arranged in an interleaved, repeating spatial layout across the global volume [S2].
3.  **Page Tables:** A two-level hierarchy managing the mapping between virtual memory addresses (VDU blocks) and physical memory addresses (GPU block cache) [S2].
4.  **GPU Block Cache:** A cache on each GPU holding resident $32^3$ voxel blocks [S2].
5.  **CPU Block Cache:** A larger cache on the CPU side, acting as an intermediate layer between disk and GPU [S2].

**Algorithm Steps:**

1.  **Initialization and Assignment:**
    - Assign responsibility for different cubical regions (VDUs) in the global virtual memory space to each GPU node [S2].
    - Arrange VDUs in an interleaved, repeating spatial layout [S2].
    - Initialize the two-level page table hierarchy for each node [S2].

2.  **Frame Preparation (Per Frame):**
    - **VDU Culling:** Each node culls the list of VDUs assigned to it based on their geometric position relative to the current view frustum [S2].
    - **Sorting:** Sort the active VDUs into a front-to-back order [S2].
    - **Synchronization (Zoom Change):** If the view zoom factor changes, switch the size of VDUs (e.g., from $1024^3$ to $512^3$) [S2].
        - Prefetch volume data corresponding to the new VDU size [S2].
        - Use a synchronization flag to signal when prefetching is complete, then all nodes switch in sync [S2].

3.  **Ray-Casting and Memory Management (Per Node, Per VDU):**
    - For each active VDU in sorted order:
        - **Ray-Casting:** Traverse the volume in virtual memory [S2].
        - **LOD Calculation:** For each sample along the ray, compute a level-of-detail (LOD) value individually, determining the octree level and thus the resolution to be sampled [S2].
        - **Address Translation:** Perform on-the-fly address translation from virtual to physical memory addresses for each sample [S2].
        - **Cache Check:** Detect cache misses for virtual octree nodes (blocks of $32^3$ voxels) [S2].
            - If the block is not in the GPU block cache:
                - Check if it is in the CPU block cache.
                - If not in CPU cache, read the physical data of matching resolution from disk in the background [S2].
                - Rendering proceeds at interactive rates in the foreground while data is fetched [S2].
        - **Rendering:** Render the VDU using the resident data.

4.  **Compositing and Communication:**
    - **Occlusion Culling:** Use the alpha image of the rendered VDU to employ occlusion culling for remaining VDUs [S2].
        - Send a low-resolution occlusion map of the rendered VDU to all other nodes [S2].
        - Other nodes blend this map with their local occlusion information [S2].
        - Skip rendering VDUs that are occluded [S2].
    - **Image Transmission:** Read back the rendered image for the VDU and directly send it out for parallel compositing [S2].
    - **Asynchronous Communication:** Use asynchronous MPI messages for communication between render nodes [S2]. Decouple sending image tiles/occlusion maps from receiving images/occlusion maps to avoid stalling [S2].
    - **Final Compositing:** Composite the images from all nodes.
    - **Client Transmission:** Transmit the final composited image to the client using TCP sockets, supporting different image compression schemes [S2].

**Parameters:**
- **Block Size:** $32^3$ voxels. Used for efficient culling, streaming, and memory management [S2].
- **VDU Size:** Typically $1024^3$ voxels. Chosen to balance data load fitting into GPU memory and avoiding excessive compositing overhead [S2].
- **Hardware:** NVIDIA Quadro 5000 (2.5 GB memory) [S2].

**Edge Cases and Pitfalls:**
- **Zoom Level:** Large VDUs lead to low GPU utilization if only a single VDU is in view; small VDUs increase compositing overhead. Dynamic switching is required [S2].
- **Precision:** On-the-fly address translation must handle virtual-to-physical mapping accurately for each sample [S2].
- **Synchronization:** VDU size switching must be synchronized across all nodes to prevent inconsistent views [S2].

## Operating characteristics
- **Time Complexity:** Rendering time remains relatively constant as data size increases, provided the working set fits in the combined physical memory of the cluster [S2]. Performance scales with the number of GPUs, though communication overhead increases [S2].
- **Memory Complexity:** Physical memory usage is limited to the current working set of active VDU blocks, not the entire volume [S2].
- **Measured Performance:** On a 955 GB dataset (21,494 x 25,790 x 1,850 voxels) with 2 GPUs and 512x512 screen resolution (1024^3 VDU size): Ray-cast 58.8 ms, Total frame 208.3 ms, max 4 distribution units per node [S2]. With 4 GPUs: Ray-cast 60.9 ms, Total frame 188.6 ms, max 2 distribution units per node [S2].
- **Strengths:** Simplifies distribution of rendering and data; no explicit load balancing scheme required; leverages large total physical cache memory; interactive rates for tera-scale data [S2].
- **Limitations:** Targets small GPU clusters; requires efficient I/O for background data fetching; communication overhead for occlusion maps and compositing [S2].
- **When to Use:** Large tera-scale volumes on GPU clusters where implicit load balancing is sufficient and explicit partitioning is undesirable [S2].
- **When Not to Use:** Very large clusters where explicit load balancing is needed; scenes with extremely high depth complexity per VDU [S2].

## Background
- **Origin:** Proposed by Beyer et al. in 2011 [S1, unofficial][S2].
- **History:** Developed to address the challenges of rendering tera-scale volumes on distributed systems, building on the concept of distributed shared virtual memory [S2]. It was designed to simplify the distribution of rendering and volume data by leveraging a shared virtual address space [S2].
- **Usage:** Used in scientific visualization, particularly for large biomedical datasets like electron microscopy (EM) data [S1, unofficial][S2].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Distributed Terascale Volume Visualization Using Distributed Shared Virtual Memory, https://vcg.seas.harvard.edu/publications/20111023-distributed-terascale-volume-visualization-using-distributed-shared-virtual-memory/paper, fetched 2026-10-27
