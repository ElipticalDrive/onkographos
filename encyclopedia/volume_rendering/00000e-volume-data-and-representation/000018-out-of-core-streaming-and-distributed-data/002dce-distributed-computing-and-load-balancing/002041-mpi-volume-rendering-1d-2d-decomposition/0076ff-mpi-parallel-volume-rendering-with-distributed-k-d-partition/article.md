# MPI Parallel Volume Rendering with Distributed k-d Partitioning

Also called: HIAC Software Rendering Parallel Algorithm (SP2), MPI software rendering, distributed k-d partitioning

## Overview
This method is a parallel software rendering algorithm for high-accuracy volume rendering that performs exact analytic or numerical integration for each primary color per pixel, distributed across nodes via MPI [S1]. It distinguishes itself from sibling methods by distributing the entire dataset to all nodes and using a distributed k-d partitioning algorithm to create load-balanced partitions based on bounding box statistics, avoiding data movement during the balancing phase [S1]. Unlike hardware-accelerated approaches, this method is strictly software-based to handle arbitrary polyhedra with high accuracy [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar dataset (e.g., tetrahedral or hexahedral mesh) and a viewpoint.
*   **Output:** A final RGB image written to disk.

**Data Structures:**
*   **Dataset:** Distributed in its entirety to all MPI nodes.
*   **Frame Buffers:** Each node maintains three separate frame buffers (Red, Green, Blue) for its assigned image tiles.
*   **Bounding Boxes:** Used for load balancing statistics exchange.

**Numbered Steps:**
1.  **Distribution:** The entire dataset is distributed to all nodes of the MPI cluster [S1].
2.  **Load Balancing:** For each viewpoint, nodes run a distributed k-d partitioning algorithm. Only bounding box statistics are exchanged between nodes until a satisfactory balance is achieved. No actual volume data moves during this phase [S1].
3.  **Visibility Ordering:** Each node calculates a visibility ordering for its local partition of the data [S1].
4.  **Integration:** Each node performs volume rendering integrations on its partition. For $n$ cells, if cell $i$ projects onto $p_i$ pixels, the total number of integrations required is:
    $$ \sum_{i=1}^n (3p_i) $$
    where the factor of 3 accounts for the three primary colors (R, G, B) [S1].
5.  **Tile Generation:** The integration results are stored in three separate frame buffers (R, G, B) on each node, resulting in a tile of the final image per node [S1].
6.  **Color Master Assembly:** Green frame buffer tiles are sent to a "green master" node, red tiles to a "red master" node, and blue tiles to a "blue master" node via the interconnect switch [S1]. Each master node assembles its final monochromatic image [S1].
7.  **Final Composition:** The green, red, and blue master nodes send their assembled images to an overall master node. The overall master accumulates the three frame buffers into the final RGB image and writes it to disk [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assumptions: MPI environment, rank 0 is overall master, 
# ranks are assigned roles (green_master, red_master, blue_master) 
# based on a pre-determined mapping or round-robin.

def main():
    # Step 1: Distribution
    # The entire dataset is distributed to all nodes.
    # Implementation detail: MPI_Bcast or MPI_Scatter of the full dataset.
    dataset = distribute_dataset()

    # Step 2: Load Balancing
    # Nodes exchange bounding box statistics to achieve balance.
    # No data moves.
    partitions = distributed_kd_partitioning(dataset)

    # Step 3 & 4: Visibility Ordering and Integration
    # Each node processes its partition.
    # Threading is not used [S1].
    local_r_tile = empty_frame_buffer()
    local_g_tile = empty_frame_buffer()
    local_b_tile = empty_frame_buffer()

    for cell in partitions[my_rank]:
        # Calculate visibility ordering for cell
        visible_pixels = get_projected_pixels(cell)
        p_i = len(visible_pixels)
        
        # Perform exact analytic or numerical integration for each pixel
        # Total integrations for this cell = 3 * p_i
        for pixel in visible_pixels:
            r_val = integrate_color(cell, pixel, channel='R')
            g_val = integrate_color(cell, pixel, channel='G')
            b_val = integrate_color(cell, pixel, channel='B')
            
            local_r_tile[pixel] = r_val
            local_g_tile[pixel] = g_val
            local_b_tile[pixel] = b_val

    # Step 6: Color Master Assembly
    # Determine role based on rank or configuration
    if my_rank == green_master_rank:
        # Receive green tiles from all other nodes
        green_image = assemble_image(local_g_tile)
        for other_rank in all_ranks:
            if other_rank != green_master_rank:
                tile = receive_tile(other_rank, channel='G')
                green_image.merge(tile)
        send_image(green_image, overall_master_rank)
        
    elif my_rank == red_master_rank:
        red_image = assemble_image(local_r_tile)
        for other_rank in all_ranks:
            if other_rank != red_master_rank:
                tile = receive_tile(other_rank, channel='R')
                red_image.merge(tile)
        send_image(red_image, overall_master_rank)
        
    elif my_rank == blue_master_rank:
        blue_image = assemble_image(local_b_tile)
        for other_rank in all_ranks:
            if other_rank != blue_master_rank:
                tile = receive_tile(other_rank, channel='B')
                blue_image.merge(tile)
        send_image(blue_image, overall_master_rank)

    # Step 7: Final Composition
    if my_rank == overall_master_rank:
        green_image = receive_image(green_master_rank)
        red_image = receive_image(red_master_rank)
        blue_image = receive_image(blue_master_rank)
        
        final_rgb = combine_channels(red_image, green_image, blue_image)
        write_to_disk(final_rgb)
```

**Parameters:**
*   **Number of Nodes:** Tested on 4, 8, 16, 32, 64, and 128 nodes [S1].
*   **Dataset Size:** 600,000 tetrahedra (for timing benchmarks) [S1].

**Edge Cases and Pitfalls:**
*   **Gathering Bottleneck:** The gathering operation (Step 6/7) becomes a bottleneck at around 32 processors, causing the speed-up curve to flatten [S1].
*   **No Threading:** The algorithm does not use threading for rendering [S1].

## Operating characteristics
*   **Time Complexity:** The total number of integrations is $\sum_{i=1}^n (3p_i)$ [S1].
*   **Measured Performance (SP2, 600,000 tetrahedra):**
    *   4 nodes: 39.69s (Load balance: 0.09s, Sort/Render: 39.17s, Gather: 0.43s)
    *   8 nodes: 15.79s (Load balance: 0.09s, Sort/Render: 13.30s, Gather: 2.40s)
    *   16 nodes: 9.64s (Load balance: 0.08s, Sort/Render: 6.09s, Gather: 2.75s)
    *   32 nodes: 4.24s (Load balance: 0.08s, Sort/Render: 2.64s, Gather: 1.52s)
    *   64 nodes: 6.47s (Load balance: 0.08s, Sort/Render: 3.30s, Gather: 3.09s)
    *   128 nodes: 2.15s (Load balance: 0.08s, Sort/Render: 1.02s, Gather: 1.05s) [S1]
*   **Scaling:** Speed-up flattens around 32 processors due to the gathering bottleneck [S1].
*   **Strengths:** High accuracy via exact analytic/numerical integration; load balancing without data movement during partitioning.
*   **Limitations:** Gathering operation bottleneck; no threading; software-only rendering.

## Background
This method was developed for the HIAC (High Accuracy Hardware-Assisted) renderer, specifically adapted for the IBM SP2 supercomputer using MPI [S1]. It was designed to handle meshes with arbitrary polyhedra, requiring software-based rendering for high accuracy [S1].

## Sources
[S1] Parallelizing a High Accuracy Hardware-Assisted Volume Renderer for Meshes with Arbitrary Polyhedra, https://digital.library.unt.edu/ark:/67531/metadc1406332/?q=texture slicing volume rendering, fetched 2026-10-28
