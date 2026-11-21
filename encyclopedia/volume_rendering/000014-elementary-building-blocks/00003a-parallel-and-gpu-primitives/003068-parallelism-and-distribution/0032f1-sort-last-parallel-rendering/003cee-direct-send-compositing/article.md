# Direct Send Compositing

Also called: Direct Send

## Overview
Direct Send Compositing is the simplest parallel image compositing technique for sort-last volume rendering, where each processor sends its partial image pixel data directly to the specific processor responsible for compositing that image partition [S3]. It solves the problem of global image assembly in a distributed memory system by avoiding the exchange of full-size images, instead exchanging only the specific tiles assigned to each processor [S2, unofficial]. Unlike [Binary Swap](6889-sort-last-parallel-rendering.md), which uses a hierarchical tree structure to reduce network congestion, Direct Send performs a single stage of all-to-all communication, making it easier to implement but potentially prone to link contention [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Partial image tiles rendered by $N$ processors. Each processor $p$ holds a full-size image of resolution $S \times S$, but is responsible for compositing only a specific tile (partition) of the final image.
*   **Output:** The final composited image, assembled from the tiles processed by each processor.

**Data Structures:**
*   **Image Tile:** A 2D array of pixel values (e.g., RGBA). The size of the tile depends on the number of processors $N$ and the total image resolution. If the image is divided into $N$ equal tiles, the tile dimensions are $S \times (S/N)$ (assuming horizontal striping) or similar partitioning.
*   **Message Buffer:** A contiguous block of memory containing the pixel data for the tile that processor $p$ needs to send to processor $q$.

**Steps:**
1.  **Tile Assignment:** The final image is divided into $N$ screen-space tiles. Each processor $p$ is assigned one unique tile $T_p$ to composite.
2.  **Data Exchange (All-to-All):**
    *   Each processor $p$ sends its rendered partial image data for every tile $T_q$ (where $q \neq p$) to the processor $q$ responsible for compositing $T_q$.
    *   Simultaneously, each processor $p$ receives the partial image data for its assigned tile $T_p$ from all other $N-1$ processors.
    *   This constitutes a single stage of all-to-all communication.
3.  **Local Compositing:**
    *   Once all required tiles have arrived, each processor $p$ composites its assigned tile $T_p$ independently.
    *   Compositing involves sorting the volume samples (ray-casting fragments) by depth and blending them according to the volume rendering integral equation (e.g., standard front-to-back alpha blending).
4.  **Final Gathering:**
    *   Each processor sends its fully composited tile $T_p$ to a designated master process (or all processors, depending on the implementation) for final assembly into the complete image [S2, unofficial].

**Formulas:**
Let $N$ be the number of processors.
Let $P$ be the total number of pixels in the final image.
Let $P_{sendrecv}$ be the number of pixels a processor sends and receives.
Let $N_{sendrecv}$ be the number of messages a processor sends and receives.

*   Number of messages per processor:
    $$N_{sendrecv} = N - 1$$
    [S3]

*   Number of pixels sent/received per processor:
    $$P_{sendrecv} = P \times \left(1 - \frac{1}{N}\right)$$
    [S3]

*   Total compositing time for a processor ($T_{comp}$):
    $$T_{comp} = T_{exchange} + T_{blend}$$
    [S3]

    Where $T_{exchange}$ is the time for sending and receiving, and $T_{blend}$ is the time for blending. Assuming full-duplex communication and equal latency/bandwidth for send/receive:
    $$T_{exchange} = \max(L_{send} + T_{send}, L_{recv} + T_{recv})$$
    [S3]

    With:
    $$L_{send} = N_{sendrecv} \times l_{sendrecv}$$
    $$T_{send} = \frac{P_{sendrecv}}{B_{sendrecv}}$$
    [S3]

    And blending time:
    $$T_{blend} = \frac{P_{blend}}{B_{blend}}$$
    [S3]

    Where:
    *   $l_{sendrecv}$: Latency per message.
    *   $B_{sendrecv}$: Network bandwidth for sending/receiving.
    *   $B_{blend}$: Blending speed (pixels per second).
    *   $P_{blend}$: Number of pixels to blend (typically $P/N$).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_send_compositing(rank, num_procs, image_resolution, render_function, composite_function):
    """
    rank: Current processor ID (0 to num_procs - 1)
    num_procs: Total number of processors N
    image_resolution: Tuple (width, height) of the final image
    render_function: Function that renders the volume and returns partial image tiles
    composite_function: Function that composites a tile from received partial images
    """
    # 1. Render the volume
    # Each processor renders the entire volume but only stores/keeps the fragments
    # relevant to its assigned tile. However, for Direct Send, it must send
    # all its rendered fragments to all other processors.
    # In practice, this means rendering the full image but partitioning the output.
    my_partial_image = render_function(rank, num_procs)
    
    # Determine tile assignment
    # Assuming horizontal striping for simplicity
    tile_height = image_resolution[1] // num_procs
    my_tile_id = rank
    my_tile_y_start = my_tile_id * tile_height
    my_tile_y_end = my_tile_y_start + tile_height
    
    # 2. All-to-All Communication
    # Each processor sends its partial image to all other processors
    # and receives partial images from all other processors.
    # In MPI, this is MPI_Alltoall.
    
    # Prepare send buffer: list of partial images to send to each rank
    # Each processor sends its entire rendered partial image to everyone.
    send_buffers = [my_partial_image] * num_procs
    
    # Prepare receive buffer: list to store received partial images
    recv_buffers = [None] * num_procs
    
    # Perform all-to-all exchange
    # Note: In a real implementation, this would be an MPI call.
    # Here we simulate the data flow.
    for target_rank in range(num_procs):
        if target_rank != rank:
            # Send my_partial_image to target_rank
            # Receive partial image from target_rank into recv_buffers[target_rank]
            pass # MPI_Alltoall(send_buffers, recv_buffers)
            
    # 3. Local Compositing
    # Each processor composites its assigned tile using the received partial images
    # from all other processors.
    
    # Collect all partial images relevant to my_tile_id
    # Since every processor sent its full partial image, we need to extract
    # the fragments that fall within my_tile_y_start to my_tile_y_end.
    # However, the standard Direct Send model assumes each processor sends
    # the specific tile data it rendered. If the render function returns
    # the full image, we must slice it.
    
    received_fragments = []
    for source_rank in range(num_procs):
        # Extract the portion of the received image that corresponds to my tile
        # This assumes the received image is the full resolution or properly partitioned
        fragment = extract_tile(recv_buffers[source_rank], my_tile_y_start, my_tile_y_end)
        received_fragments.append(fragment)
        
    # Composite the tile
    composited_tile = composite_function(received_fragments, my_tile_y_start, my_tile_y_end)
    
    # 4. Final Gathering
    # Send the composited tile to the master process
    if rank == 0:
        master_tile = composited_tile
        # Receive tiles from all other ranks
        for r in range(1, num_procs):
            other_tile = receive_tile_from(r)
            # Assemble the final image
            pass
    else:
        send_tile_to(0, composited_tile)
        
    return composited_tile if rank == 0 else None
```

**Edge Cases and Pitfalls:**
*   **Link Contention:** Because all processors send to all other processors simultaneously, link contention is likely to occur, especially on networks with limited bandwidth or non-full-duplex capabilities [S3].
*   **Message Size Limits:** On some systems (e.g., Cray XT4 with MPICH), large data sizes and all-to-all communication can exceed MPI capabilities, causing failures beyond a certain number of processors (e.g., 1021 processors for $4096^2$ images) [S3].
*   **Protocol Switching:** Timing drops can occur when the message size exceeds the `MPICH_MAX_SHORT_MSG_SIZE` (default 128000 bytes), switching from short (eager) to long message protocols, which increases latency [S3].
*   **Power of Two:** Unlike Binary Swap, Direct Send does not require the number of processors to be a power of two [S1].

## Subtopics

### [Tiling Compositing](12057-tiling-compositing.md)
Tiling Compositing is a parallel compositing strategy for distributed volume rendering where processing elements (PEs) hold tiles of the final image and pass partial results to adjacent PEs in a staggered, linear fashion to merge them into the final image. It solves the problem of global image assembly in a distributed memory system by avoiding the exchange of full-size images, instead exchanging only the specific tile assigned to each PE. Unlike the inverted binary tree method, which leaves a majority of PEs idle during latter cycles, tiling keeps all PEs working on the problem until the final image is completed.

### [Parallel Compositing](16580-parallel-compositing.md)
Parallel Compositing is a distributed rendering technique that partitions the final image into spatial tiles, assigning each tile to a specific compositing task. It solves the bottleneck of global image assembly by exchanging only the ray-casting fragments relevant to each tile, performing local compositing, and gathering the result. It differs from Direct Send Compositing by using a collective `MPI_Alltoallv` communication pattern rather than explicit point-to-point sends, and from Tiling Compositing by its specific reliance on the Alltoallv primitive for fragment exchange.

### SLIC Parallel Image Compositing
SLIC Parallel Image Compositing is an optimized parallel image compositing algorithm for sort-last rendering that minimizes the number of messages exchanged among processors using a view-dependent precomputed compositing schedule [S4]. It solves the bottleneck of Direct Send by reducing the total amount of data sent over the network. The algorithm involves a preprocessing step (overhead < 10ms) to compute a compositing schedule for each new view, which is then used to minimize data transmission [S4]. SLIC outperforms previous algorithms, especially for high-resolution images (1024x1024 pixels or larger) [S4]. On low-bandwidth networks, care must be taken to avoid many-to-one or many-to-many communication patterns inherent in direct send [S4].

## Operating characteristics
**Time and Memory Complexity:**
*   **Communication Steps:** Direct Send requires only one stage of all-to-all communication [S3].
*   **Messages:** In the worst case, $N \times (N - 1)$ messages are exchanged among $N$ compositing processors [S3].
*   **Data Volume:** Each processor sends and receives $P \times (1 - 1/N)$ pixels, where $P$ is the total number of pixels in the image [S3].
*   **Memory:** Each processor must store the full partial image it rendered and the partial images received from all other processors before compositing.

**Measured Performance:**
*   On the Cray XT4 machine, Direct Send failed beyond 1021 processors for $4096^2$ images due to MPI limitations with large data sizes [S3].
*   Performance drops significantly when message sizes exceed the short message protocol threshold (e.g., 128000 bytes in MPICH) [S3].
*   Direct Send scales poorly compared to Binary Swap or 2-3 Swap as the number of processors increases, due to the $O(N^2)$ message complexity and network congestion [S3].

**Strengths and Limitations:**
*   **Strengths:** Simplest implementation; requires only one communication stage; does not require the number of processors to be a power of two; works well for small numbers of processors or high-bandwidth networks [S1, S3].
*   **Limitations:** Prone to link contention; poor scalability for large $N$; high communication overhead; sensitive to network topology and MPI implementation limits [S3].

**When to Use:**
*   Use Direct Send when the number of processors is small, the network bandwidth is high, and implementation simplicity is prioritized over scalability [S1].
*   Avoid Direct Send when the number of processors is large, the network is congested, or the image resolution is very high, leading to large message sizes [S3].

## Background
Direct Send Compositing was introduced as the simplest image compositing technique for sort-last parallel rendering [S3]. It has been used often in prior work [S1]. The method is based on the idea that each processor sends its pixel data directly to the processor responsible for the corresponding image partition in a single stage of communication [S3]. It is often compared with Binary Swap, which was proposed to reduce network congestion by using a hierarchical communication scheme [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S3] Massively Parallel Volume Rendering Using 2-3 Swap ..., http://vis.cs.ucdavis.edu/papers/SC08_compositing.pdf, fetched 2026-10-29
[S4] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
