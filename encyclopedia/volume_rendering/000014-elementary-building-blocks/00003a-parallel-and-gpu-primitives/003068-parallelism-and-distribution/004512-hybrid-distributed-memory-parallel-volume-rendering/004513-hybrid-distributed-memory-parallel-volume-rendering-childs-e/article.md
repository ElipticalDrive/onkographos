# Hybrid Distributed-Memory Parallel Volume Rendering (Childs et al.)

Also called: Childs et al. Hybrid Approach, Hybrid Volume Rendering

## Overview
The Childs et al. hybrid approach is a distributed-memory parallel volume rendering algorithm that balances load by classifying cells as "small" or "large" based on their estimated sample count [S1]. It combines object-order sampling for small cells with image-order sampling for large cells, ensuring that neither the number of cells nor the number of samples per processor becomes a bottleneck for load imbalance [S1]. Unlike pure object-order methods which suffer when camera zoom or mesh heterogeneity causes uneven sample distribution, and pure image-order methods which suffer when tile boundaries cut through dense cell regions, this method bounds both metrics to maintain scalability on unstructured grids [S1].

## How it works
The algorithm operates in three distinct phases: Object Order Sampling (Small Cells), Exchange, and Image Order Sampling (Large Cells) [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Unstructured volume dataset (cells, scalar values, geometry).
    *   Camera parameters (position, orientation, field of view).
    *   Image dimensions: Width $W$, Height $H$.
    *   System parameters: Total processors $N$, samples per ray $SPR$.
    *   Threshold: A predefined value to classify cells as small or large (Not given in the sources).
*   **Outputs:**
    *   Final rendered image (RGB pixels).

### Data Structures
*   **Cell:** Contains geometry (bounding box) and scalar data.
*   **Sample Buffer:** Stores sampled values for small cells.
*   **Partial Composite:** A compact representation of a segment of a ray. It consists of:
    *   $C$: RGB color value (4 floating-point values).
    *   $\alpha$: Accumulated opacity (1 floating-point value, implied by the compositing equation).
    *   $Z_s$: Integer storing the depth of the first value in the partial composite.
    *   $Z_e$: Integer storing the depth of the final sample in the partial composite.
    *   Total size: 4 floats + 2 integers (24 bytes on standard 32-bit/64-bit mixed architectures, significantly smaller than storing all intermediate samples) [S1].
*   **Large Cell Buffer:** Stores the full cell data deferred from the first phase.

### Numbered Steps

1.  **Distribute Data:**
    *   If data is not decomposed, each processor $i$ reads $C/P$ cells, where $C$ is total cells and $P$ is total processors [S1].
    *   If data is decomposed into blocks, each processor reads its assigned block [S1].

2.  **Cell Assessment (Object Order Sampling Phase):**
    *   For each cell owned by processor $i$, estimate the number of samples $SPC_{ij}$ for cell $j$.
    *   Calculation: Multiply the number of pixels covered by the cell's bounding box in image space by the number of samples in depth overlapping the bounding box [S1].
    *   Classify cell $j$ as **Small** if $SPC_{ij} \leq \text{Threshold}$, otherwise **Large** [S1].

3.  **Small Cell Processing:**
    *   Sample all **Small** cells using a multi-threaded implementation (adapted from Larsen et al.) [S1].
    *   Store sampled values in a sample buffer [S1].
    *   Generate **Partial Composites** for each pixel. Parallelize by working on different pixels simultaneously [S1].
    *   Composite continuous samples along the depth-axis into partial composites. Each partial composite is defined by $(C, \alpha, Z_s, Z_e)$ [S1].

4.  **Exchange Phase:**
    *   **Distribute Pixels:** Divide the image into $\sqrt{N}$ tiles. Assign each processor a tile of size $(W/\sqrt{N}) \times (H/\sqrt{N})$ [S1].
    *   **Determine Ownership:** Identify which processors own the **Large** cells that contribute to each processor's tile, and which processors hold the **Partial Composites** for pixels in each tile [S1].
    *   **Communication:**
        *   Pack **Large Cells** and **Partial Composites** into buffers [S1].
        *   Perform exchange using a direct send approach with MPI [S1].
        *   Unpack messages into defined structures (partial composite or large cell) [S1].

5.  **Large Cell Processing (Image Order Sampling Phase):**
    *   Sample the **Large** cells received during the exchange phase using multi-threading [S1].
    *   Restrict sampling to the boundaries of the processor's assigned tile to avoid redundant sampling across processors [S1].
    *   Generate **Partial Composites** for these large cells within the tile [S1].

6.  **Compositing:**
    *   Order all partial composites (from small cells and large cells) for each pixel using depth information $Z_s$ and $Z_e$ [S1].
    *   Combine partial composites in front-to-back order using the equation:
        $$C_{out} = C_{in} + (1 - \alpha) \ast C$$
        Where $C_{out}$ is the final RGB value, $C_{in}$ is the current accumulated color, $\alpha$ is the accumulated opacity, and $C$ is the color of the incoming partial composite [S1].

7.  **Collecting Final Image:**
    *   One processor acts as the host [S1].
    *   All other processors send their final image tiles to the host [S1].
    *   The host receives and arranges the tiles according to the assignment to produce the final image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_volume_rendering(dataset, camera, W, H, N, threshold):
    """
    dataset: List of cells with geometry and scalar data
    camera: Camera parameters
    W, H: Image dimensions
    N: Number of processors
    threshold: Sample count threshold for small/large classification
    """
    # Phase 1: Object Order Sampling (Small Cells)
    small_cells = []
    large_cells = []
    
    # Distribute data (simplified for single processor logic, 
    # in reality distributed across ranks)
    local_cells = distribute_data(dataset)
    
    for cell in local_cells:
        # Estimate samples
        pixels_covered = count_pixels_in_bbox(cell.bbox, camera, W, H)
        depth_samples = count_depth_samples(cell.bbox, camera, SPR)
        estimated_samples = pixels_covered * depth_samples
        
        if estimated_samples <= threshold:
            small_cells.append(cell)
        else:
            large_cells.append(cell)
            
    # Sample small cells and generate partial composites
    partial_composites_small = []
    for cell in small_cells:
        samples = sample_cell(cell, camera)
        # Composite samples into partial composites
        pc = composite_samples_to_partial(samples)
        partial_composites_small.append(pc)
        
    # Phase 2: Exchange
    # Divide image into sqrt(N) tiles
    tile_size_w = W / sqrt(N)
    tile_size_h = H / sqrt(N)
    
    # Determine which large cells and partial composites belong to which tiles
    # This requires communication to gather ownership info
    # For pseudocode, assume we have a mapping: tile_id -> (large_cells, partial_composites)
    
    # Exchange large cells and partial composites
    # In MPI: send large_cells and partial_composites_small to appropriate ranks
    # receive large_cells_from_others and partial_composites_from_others
    
    # Phase 3: Image Order Sampling (Large Cells)
    partial_composites_large = []
    for cell in large_cells:
        # Restrict sampling to tile boundaries
        samples = sample_cell_tile_restricted(cell, camera, tile_id)
        pc = composite_samples_to_partial(samples)
        partial_composites_large.append(pc)
        
    # Combine all partial composites
    all_partial_composites = partial_composites_small + partial_composites_large + partial_composites_large_from_others
    
    # Sort by depth Zs
    all_partial_composites.sort(key=lambda pc: pc.Zs)
    
    final_image = []
    for pixel in range(W * H):
        # Get partial composites for this pixel
        pcs_for_pixel = [pc for pc in all_partial_composites if pc.pixel_id == pixel]
        
        # Front-to-back compositing
        C_in = (0, 0, 0)
        alpha = 0
        for pc in pcs_for_pixel:
            # Cout = Cin + (1 - alpha) * C
            C_out = C_in + (1 - alpha) * pc.C
            alpha = alpha + (1 - alpha) * pc.alpha
            C_in = C_out
            
        final_image.append(C_out)
        
    # Collect Final Image
    if is_host():
        receive_tiles_from_others()
        arrange_tiles(final_image)
        return final_image
    else:
        send_tile_to_host(final_image)
```

### Parameters
*   $N$: Total number of processors [S1].
*   $W, H$: Image width and height [S1].
*   $SPR$: Number of samples per ray [S1].
*   $C_i$: Number of cells owned by processor $i$ [S1].
*   $SC_i$: Number of small cells owned by processor $i$ [S1].
*   $LC_i$: Number of large cells owned by processor $i$ [S1].
*   $PC_i$: Number of partial composites owned by processor $i$ [S1].
*   $SPC_{ij}$: Estimated number of samples for cell $j$ owned by processor $i$ [S1].
*   $Z_s, Z_e$: Depth integers for partial composites [S1].

### Edge Cases and Pitfalls
*   **Load Imbalance in Object Order:** If the camera zooms into one region or the mesh has unequal cell sizes, object order suffers from unequal distribution of samples [S1]. The hybrid approach avoids this by deferring large cell sampling [S1].
*   **Load Imbalance in Image Order:** If the camera zooms out, some processors may have more cells contributing to their tile, causing imbalance [S1]. The hybrid approach addresses this by ensuring equal distribution via the tile assignment and deferred sampling [S1].
*   **Precision:** The use of integers for $Z_s$ and $Z_e$ assumes discrete depth steps. If depth resolution is very high, precision loss may occur [G].
*   **Boundary Conditions:** Sampling must be strictly restricted to tile boundaries in the large cell phase to avoid redundant work and ensure correct compositing order [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Object Order Sampling: $O(SC_i)$, proportional to the number of small cells owned by processor $i$ [S1].
    *   Exchange: $O(\sum_{i=0}^{n} PC_i) + O(\sum_{i=0}^{n} LC_i)$, cost of exchanging partial composites and large cells [S1].
    *   Image Order Sampling: $O(\sum_{i=0}^{n} \frac{P \times SPR}{N})$, bounded by the potential samples in the tile [S1].
*   **Memory Complexity:**
    *   Partial composites reduce storage from storing all intermediate samples (e.g., 400 bytes for 100 samples) to 24 bytes (4 floats + 2 integers) [S1].
*   **Measured Performance:**
    *   Tested on 256 nodes with 1 MPI task per node and 12 threads of on-node parallelism [S1].
    *   Outperforms pure object-order and image-order approaches in scenarios with varying camera positions (zoom in/out) and unstructured meshes with varying cell sizes [S1].
*   **Strengths:**
    *   Bounds both the number of cells and samples per processor, preventing load imbalance [S1].
    *   Reduces memory and communication costs by using partial composites [S1].
*   **Limitations:**
    *   Requires two communication phases (exchange of large cells and partial composites) [S1].
    *   Complexity of implementation due to hybrid logic [S1].

## Background
The hybrid approach was developed by Childs et al. [S1]. It was designed to address the difficulties of volume rendering on unstructured meshes, where cell sizes vary dramatically, leading to load imbalance, and where interleaving conditions affect compositing ordering [S1]. The algorithm was tested on distributed memory systems and extended to take advantage of many-core architectures [S1]. It serves as the basis for subsequent work that further reduces memory costs and optimizes for modern hardware [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
