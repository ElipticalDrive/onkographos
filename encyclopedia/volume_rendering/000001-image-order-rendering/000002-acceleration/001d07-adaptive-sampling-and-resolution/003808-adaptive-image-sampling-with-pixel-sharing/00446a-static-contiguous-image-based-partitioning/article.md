# Static Contiguous Image-Based Partitioning

Also called: Image Order Volume Rendering, Image-order distribution, Sort-first

## Overview
Static Contiguous Image-Based Partitioning is a parallel volume rendering strategy that distributes the final image plane into disjoint, contiguous blocks (tiles) among processors, where each processor independently renders its assigned section [S1][S4, unofficial]. It solves the problem of load imbalance and communication overhead inherent in adaptive image sampling methods by avoiding the replicated ray casting costs associated with shared pixels at partition boundaries [S3, unofficial]. Unlike its parent method, Adaptive Image Sampling with Pixel Sharing, which requires complex synchronization to handle shared pixels between adaptive regions, this method uses static contiguous partitioning to ensure that shared pixels are computed only once by neighboring processors in a specific order, eliminating synchronization overhead [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$, a camera configuration (view matrix, projection matrix), and a target image resolution $W \times H$.
*   **Output:** A final composite image $I$ of resolution $W \times H$.

**Data Structures:**
*   **Volume Data:** Distributed across $P$ processors. Each processor $p$ holds a subset of the volume data $V_p$ responsible for rendering its tile.
*   **Image Partition:** The image is divided into $P$ contiguous blocks. If adaptive sampling is used, the image is conceptually divided into square sample regions of size $I_{max} \times I_{max}$ pixels [S3, unofficial]. These regions are grouped into $P$ contiguous large blocks with roughly equal amounts of work [S3, unofficial].
*   **Shared Memory/Communication:** In shared-memory architectures, processors communicate via shared memory. In distributed-memory systems, processors communicate via message passing [S4, unofficial].

**Steps:**
1.  **Data Distribution:**
    *   Volume data is distributed arbitrarily across processors initially [S4, unofficial].
    *   Each processor determines the ownership of the subvolume required for its assigned tile [S4, unofficial].
    *   Processors keep the data they are responsible for and send all other data to the corresponding processor in charge [S4, unofficial]. This redistribution may happen at the beginning of every frame computation [S4, unofficial].
    *   *Note:* In shared-memory systems, this step may be simplified if data is already accessible, but logical ownership must be established [S2, unofficial].

2.  **Tile Assignment:**
    *   The image is partitioned into $P$ contiguous blocks [S3, unofficial].
    *   Each processor $p$ is assigned one block $B_p$ [S3, unofficial].

3.  **Rendering (Scanline Order):**
    *   Each processor renders its assigned block $B_p$ in scanline order from top to bottom [S3, unofficial].
    *   For each pixel (or sample region) in the block, the processor casts rays into the volume data it owns ($V_p$) and composites the result [S1].
    *   If adaptive sampling is employed, the processor calculates sample regions within its block [S3, unofficial].

4.  **Handling Shared Pixels (Boundary Condition):**
    *   Pixels shared by image sections processed by different processors are located only in the square sample regions at the top and bottom of each image section [S3, unofficial].
    *   **Top Regions:** Each processor calculates the shared pixels in the top sample regions of its section [S2, unofficial].
    *   **Bottom Regions:** When a processor reaches the bottom sample regions, it does *not* cast rays for shared pixels. Instead, it uses the shared pixel values already available in shared memory (or received from lower neighbors), which were calculated by its lower neighboring processors [S2, unofficial].
    *   This mechanism avoids replicated ray casting for shared pixels [S3, unofficial].

5.  **Result Collection:**
    *   The sub-images from all processors are collected onto one processor (or composited in a shared frame buffer) to produce the final image [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def static_contiguous_partitioning_rendering(volume_data, camera, image_width, image_height, num_processors, processor_id):
    """
    Renders a portion of the image using static contiguous partitioning.
    
    Args:
        volume_data: The full 3D volume dataset (accessible locally or via communication).
        camera: Camera parameters (view and projection matrices).
        image_width: Width of the final image in pixels.
        image_height: Height of the final image in pixels.
        num_processors: Total number of processors P.
        processor_id: ID of the current processor (0 to P-1).
        
    Returns:
        sub_image: The rendered portion of the image for this processor's tile.
    """
    
    # Step 1: Data Distribution (Simplified for single-process simulation)
    # In a real distributed system, volume_data would be filtered to only include
    # cells contributing to this processor's tile.
    local_volume = redistribute_volume(volume_data, processor_id, num_processors)
    
    # Step 2: Tile Assignment
    # Divide image into P contiguous blocks.
    # Assuming equal height distribution for simplicity.
    tile_height = image_height // num_processors
    tile_start_y = processor_id * tile_height
    tile_end_y = (processor_id + 1) * tile_height if processor_id < num_processors - 1 else image_height
    
    sub_image = empty_image(image_width, tile_end_y - tile_start_y)
    
    # Step 3 & 4: Rendering with Shared Pixel Handling
    # Iterate through the tile in scanline order (top to bottom)
    for y in range(tile_start_y, tile_end_y):
        for x in range(0, image_width):
            
            # Determine if this pixel is a "shared" pixel at the boundary
            is_top_boundary = (y == tile_start_y) and (processor_id > 0)
            is_bottom_boundary = (y == tile_end_y - 1) and (processor_id < num_processors - 1)
            
            if is_top_boundary:
                # Calculate shared pixels in the top sample regions
                # Cast rays and composite
                pixel_value = cast_ray_and_composite(local_volume, camera, x, y)
                sub_image[y - tile_start_y][x] = pixel_value
                # Store in shared memory for lower neighbors
                store_in_shared_memory(x, y, pixel_value)
                
            elif is_bottom_boundary:
                # Use shared pixel values from lower neighbors
                # These were calculated by the processor with ID (processor_id + 1)
                pixel_value = retrieve_from_shared_memory(x, y)
                sub_image[y - tile_start_y][x] = pixel_value
                
            else:
                # Standard pixel rendering
                pixel_value = cast_ray_and_composite(local_volume, camera, x, y)
                sub_image[y - tile_start_y][x] = pixel_value
                
    return sub_image

def redistribute_volume(full_volume, processor_id, num_processors):
    """
    Distributes volume data based on tile ownership.
    In a real system, this involves MPI_Send/MPI_Recv or shared memory allocation.
    """
    # Determine which cells contribute to the tile assigned to processor_id
    # This is a placeholder for the actual geometric culling and redistribution logic.
    # [G] In practice, this requires projecting the volume cells onto the image plane
    # to determine which cells fall within the tile's screen-space bounds.
    local_cells = []
    for cell in full_volume.cells:
        if cell_contributes_to_tile(cell, processor_id, num_processors):
            local_cells.append(cell)
    return VolumeData(local_cells)

def cast_ray_and_composite(volume, camera, x, y):
    """
    Casts a ray from the camera through pixel (x, y) into the volume and composites.
    """
    # [G] Standard ray casting loop
    ray_origin = camera.unproject(x, y)
    ray_direction = camera.get_direction(x, y)
    color = [0.0, 0.0, 0.0, 0.0]
    t = 0.0
    while t < ray_length:
        sample = volume.sample(ray_origin + t * ray_direction)
        color = composite_step(color, sample)
        t += step_size
    return color
```

**Parameters:**
*   $I_{max}$: The side length of square sample regions in pixels, used when adaptive sampling is active. Not given in the sources.
*   $P$: Number of processors.

**Edge Cases and Pitfalls:**
*   **Load Imbalance:** Image order does not guarantee equal cell distribution. If one tile contains more cells (e.g., due to uneven cell sizes in unstructured data or camera focus on a specific region), its processor performs more work, causing load imbalance [S1].
*   **Replicated Loading:** Cells covering more than one tile result in replicated loading of those cells across multiple processors, increasing memory usage and communication costs [S1].
*   **Scalability Limits:** In distributed memory systems, as $P$ increases, the image space grows into too many pieces, causing data redistribution and boundary data redundancy to become serious scalability limitations [S4, unofficial].
*   **Per-Frame Coherence:** In real-time volume rendering, user interaction can cause sudden scene changes, breaking per-frame coherence and preventing the use of retained-mode optimizations that reduce transferred data [S4, unofficial].

## Subtopics

### [Hybrid Partitioning Parallel Ray Casting](11261-hybrid-partitioning-parallel-ray-casting.md)
Combines object-based partitioning for boundary cell projection with image-based partitioning for ray traversal to achieve load balancing. It sets itself apart by decoupling the partitioning strategy to avoid replicated ray casting costs at processor boundaries, unlike the purely image-based approach of this method.

## Operating characteristics
*   **Time Complexity:** Depends on the number of pixels in the tile and the number of cells contributing to those pixels. Load imbalance can cause significant variance in execution time across processors [S1].
*   **Memory Complexity:** Each processor must store the volume data contributing to its tile. Replicated loading of cells covering multiple tiles increases memory usage [S1].
*   **Measured Performance:** Scalability depends directly on the size of the final image and indirectly on the size of the underlying dataset [S4, unofficial]. It scales well with increasing image resolutions because only complete pixels are transmitted [S4, unofficial].
*   **Strengths:**
    *   Avoids replicated ray casting costs for shared pixels without extra synchronization overhead [S3, unofficial].
    *   Instantly compatible with all serial rendering algorithms because it executes the full graphics pipeline in each processor [S4, unofficial].
    *   Simpler implementation compared to dynamic partitioning methods [S3, unofficial].
*   **Limitations:**
    *   Does not guarantee load balance [S1].
    *   Replicated cell loading increases memory and communication costs [S1].
    *   Scalability is limited in distributed memory systems due to data redistribution overhead [S4, unofficial].
*   **When to Use:** Suitable for shared-memory architectures where data redistribution is cheap, or when compatibility with existing serial rendering algorithms is a priority [S4, unofficial]. Not ideal for distributed systems with high communication latency or highly uneven data distributions [S1][S4, unofficial].

## Background
Static Contiguous Image-Based Partitioning, also known as Sort-first rendering, is a parallelization strategy for volume rendering that divides the image into contiguous blocks assigned to processors [S3, S4, unofficial]. It was proposed as a simpler and more effective solution than dynamic image-based partitioning (e.g., Nieh and Levoy [2]) to avoid the cost of replicated ray casting associated with pixel sharing in adaptive image sampling [S3, unofficial]. The method is compatible with standard APIs like OpenGL, which lack high-performance access to intermediate rendering results, making sort-middle impractical [S4, unofficial]. It has been used in various parallel volume rendering schemes, including those for unstructured volume data [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S2] Time-Critical Volume Rendering, https://ntrs.nasa.gov/api/citations/19990040407/downloads/19990040407.pdf, fetched 2026-10-26 (unofficial)
[S3] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
[S4] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
