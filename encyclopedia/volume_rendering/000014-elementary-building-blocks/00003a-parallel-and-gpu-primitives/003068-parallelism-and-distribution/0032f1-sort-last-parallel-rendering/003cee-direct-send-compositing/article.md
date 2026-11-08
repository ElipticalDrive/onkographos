# Direct Send Compositing

Also called: Direct send

## Overview
Direct send compositing is a parallel image compositing method used in sort-last rendering to combine partial images rendered by multiple processors into a final composite [S1, unofficial]. It solves the problem of efficiently aggregating screen-space data by dividing the final image into $n$ tiles, where each processor is responsible for compositing one specific tile [S1, unofficial]. Unlike hierarchical methods such as Binary Swap or Tree-like algorithms that use multi-stage communication to reduce message counts, Direct Send relies on a single all-to-all communication stage, making it simpler to implement and robust even with expensive synchronization mechanisms [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $n$ partial images (tiles), one rendered by each of the $n$ processors in the cluster. Each partial image contains the ray-cast results for a specific screen-space tile.
*   **Output:** One final composite image, assembled by a master process.

**Data Structures:**
*   **Tile:** A rectangular sub-region of the final image. The final image is divided into $n$ equal-sized tiles, one per processor.
*   **Message:** Contains the pixel data for the tile rendered by a specific processor, intended for a specific destination processor.

**Algorithm Steps:**
1.  **Tile Assignment:** The final image is divided into $n$ screen-space tiles. Processor $i$ is assigned the responsibility of compositing the $i$-th tile.
2.  **Local Rendering:** Each processor $i$ renders its portion of the volume data, producing a partial image. This partial image is split into $n$ tiles. Processor $i$ keeps its own tile (Tile $i$) and prepares the other $n-1$ tiles (Tiles $j$ where $j \neq i$) for transmission.
3.  **All-to-All Exchange:** All processors simultaneously send their $n-1$ non-local tiles to the respective processors responsible for compositing them. Processor $i$ receives tiles $j$ from all other processors $j$.
    *   *Optimization:* The algorithm can be tuned to send only pixels that actually need compositing (e.g., transparent pixels may be skipped if the background is known, or only opaque fragments are sent), reducing the average number of exchanging nodes to $n$ [S1, unofficial].
4.  **Independent Compositing:** Once all tiles for a specific destination have arrived at processor $k$, processor $k$ independently composites its assigned tile using front-to-back or back-to-front alpha blending.
5.  **Final Gathering:** Each processor sends its fully composited tile to a designated master process.
6.  **Assembly:** The master process assembles the $n$ composited tiles into the final image.

**Parameters:**
*   **$n$:** Number of processors/nodes.
*   **Communication Steps:** 1 (the all-to-all exchange).
*   **Synchronization Points:** 2 (one after the exchange, one after the gathering) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_send_compositing(my_id, n, my_partial_image_tiles):
    """
    my_id: Integer ID of this processor (0 to n-1).
    n: Total number of processors.
    my_partial_image_tiles: List of n tiles. 
                            my_partial_image_tiles[i] is the tile for screen position i,
                            rendered by this processor.
    """
    
    # Step 1: Prepare messages
    # Each processor sends tile j to processor j, for all j != my_id
    messages_to_send = []
    for dest_id in range(n):
        if dest_id != my_id:
            # Send the tile that belongs to dest_id's compositing responsibility
            tile_data = my_partial_image_tiles[dest_id]
            messages_to_send.append((dest_id, tile_data))
            
    # Step 2: All-to-All Communication
    # Send all prepared messages and receive messages from all other processors
    received_tiles = send_and_receive_all(messages_to_send, n)
    # received_tiles[j] contains the tile data sent by processor j to my_id
    
    # Step 3: Composite the assigned tile
    # Processor my_id is responsible for compositing tile my_id
    # It needs to composite all received tiles for position my_id
    # Note: received_tiles[my_id] is the tile rendered by myself, 
    # received_tiles[k] is the tile rendered by processor k for position my_id
    
    composited_tile = empty_tile()
    for source_id in range(n):
        tile_data = received_tiles[source_id]
        # Perform alpha blending of tile_data into composited_tile
        # Standard front-to-back compositing:
        composited_tile = alpha_blend(composited_tile, tile_data)
        
    # Step 4: Send composited tile to master
    # Assuming processor 0 is the master, or a specific master_id is known
    master_id = 0 
    if my_id != master_id:
        send_to_master(master_id, composited_tile)
    else:
        # Master receives tiles from all other processors
        # and assembles the final image
        final_image = assemble_final_image(n, my_composited_tile, receive_from_all())
        
    return final_image if my_id == master_id else None
```

**Edge Cases and Pitfalls:**
*   **Network Congestion:** In the worst-case scenario, Direct Send generates up to $n(n-1)$ messages [S1, unofficial]. On low-bandwidth networks, this many-to-many communication pattern can cause significant bottlenecks [S2].
*   **Synchronization Cost:** While the algorithm has only one communication step, the synchronization required for the all-to-all exchange can be expensive if the underlying MPI or network library is not optimized [S1, unofficial].
*   **Tile Size:** As $n$ increases, tile sizes decrease. While this reduces the volume of data per message, it increases the overhead of message management and synchronization [S1, unofficial].

## Subtopics

### SLIC Parallel Image Compositing
SLIC (Scalable Low-latency Image Compositing) is an optimized variant of Direct Send that minimizes the number of messages exchanged among processors [S2]. It uses a view-dependent precomputed compositing schedule to determine exactly which pixels need to be sent, avoiding unnecessary data transfer [S2]. The preprocessing step to compute this schedule introduces very low overhead (generally under 10 milliseconds) [S2]. SLIC is particularly beneficial for high-resolution images (1024x1024 pixels or larger) and low-bandwidth networks, where reducing the message count is critical to performance [S2]. It outperforms standard Direct Send because communication is generally more expensive than computation [S2].

## Operating characteristics
*   **Time Complexity:** The communication complexity is $O(n^2)$ in the worst case due to the all-to-all exchange of $n(n-1)$ messages [S1, unofficial]. However, with optimization (sending only necessary pixels), the average number of exchanging nodes can be reduced to $n$, improving practical performance [S1, unofficial].
*   **Memory Complexity:** Each processor must store $n$ tiles (one for each screen position) during the exchange phase, even though it only composites one. This requires $O(n \cdot T)$ memory, where $T$ is the size of a tile.
*   **Measured Performance:** SLIC, an optimized Direct Send variant, outperforms previous algorithms, especially for high-resolution images (1024x1024 pixels or larger) [S2].
*   **Strengths:**
    *   Simple implementation; does not require special network topology [S2].
    *   Only one communication step with two synchronization points, making it robust even with expensive synchronization mechanisms [S1, unofficial].
    *   Tile sizes decrease as processor count increases, potentially reducing communication volume per message [S1, unofficial].
*   **Limitations:**
    *   High message count ($n(n-1)$) can saturate low-bandwidth networks [S2].
    *   Many-to-many communication patterns can lead to bottlenecks [S2].
    *   Less efficient than hierarchical methods (like Binary Swap) in terms of total data volume for large $n$ [S1, unofficial].
*   **When to use:** Suitable for high-performance clusters with high-bandwidth networks where simplicity and low synchronization overhead are prioritized. Also suitable when the number of processors is small to moderate.
*   **When not to use:** Unsuitable for low-bandwidth networks or very large $n$ where the $O(n^2)$ message count becomes prohibitive.

## Background
Direct Send compositing was introduced as a parallel compositing method for volume rendering on large-scale displays [S1, unofficial]. It is based on earlier work in parallel image compositing, such as [Hsu93, Neu94, MC97], which utilized the approach because it is easy to implement and does not require special network topology [S2]. The method is often contrasted with Binary Swap and Tree-like algorithms, which use hierarchical communication to reduce message counts [S1, unofficial]. SLIC, an optimized version of Direct Send, was developed to address the communication bottlenecks of the original method [S2].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S2] I/O Strategies for Parallel Rendering of Large Time-Varying Volume Data, https://doi.org/10.2312/egpgv/egpgv04/031-040, fetched 2026-10-26
