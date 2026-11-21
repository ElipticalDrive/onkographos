# Parallel Splatting on Cray T3D

Also called: Splatter
## Overview
Parallel Splatting on Cray T3D is a memory-optimized volume rendering method designed for the Cray T3D massively parallel supercomputer [S1, unofficial]. It projects volumetric data onto a raster using splatting, distributing both the data and the computational workload across multiple Processing Elements (PEs) to render large medical datasets, such as the Visible Human Project, at interactive rates [S1, unofficial]. It distinguishes itself from other parallel splatting approaches by its specific tiling-based compositing strategy and its optimization for the Cray T3D's distributed memory architecture, which allows shaded slices to remain in PE memory, avoiding the repeated I/O and shading overhead seen in single-node systems like the SGI Onyx [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset consisting of raw slices (CT or RGB cryosection). The dataset is defined by dimensions $N_x \times N_y \times N_z$, where $N_z$ is the number of slices.
*   **Output:** A final 2D raster image (framebuffer) representing the volume projection.

**Data Structures**
*   **Volume Data:** Stored as a sequence of 2D slices.
*   **Shaded Slice:** A 2D array of voxels. Each voxel contains 4 components (R, G, B, A).
*   **Sample Resolution:** The number of bytes per component. Supported values are 1, 2, 4, or 8 bytes.
    *   Voxel size $V_{size} = 4 \times \text{Sample Resolution}$ bytes.
    *   Slice size $S_{size} = N_x \times N_y \times V_{size}$ bytes.
*   **Processing Elements (PEs):** $p$ parallel processors.
*   **Tiles:** The output image is divided into $p$ vertical (or horizontal, depending on implementation) tiles, each of size $1/p$ of the total image area.

**Algorithm Steps**

1.  **Data Distribution**
    The volume is partitioned into contiguous regions of slices. Let $n$ be the total number of slices and $p$ be the number of PEs. The slices are distributed such that:
    *   The first $k = n - \lfloor n/p \rfloor \times p$ PEs receive $\lfloor n/p \rfloor + 1$ slices.
    *   The remaining $p - k$ PEs receive $\lfloor n/p \rfloor$ slices.
    *   Constraint: $n \ge p$. No slice is split between PEs [S1, unofficial].

2.  **Shading Slices**
    Each PE processes its assigned slices independently.
    *   **CT Slices:** For each raw slice $z$, the PE calculates the shaded value using raw slices $z-1$, $z$, and $z+1$. Raw slices are discarded once they are no longer needed to free memory for new shaded slices [S1, unofficial].
    *   **RGB Cryosection Slices:** Only the alpha (transparency) value is calculated for each voxel. The alpha value is set to a small value based on the voxel's color to enhance red muscle tissue and internal organs. RGB values are preserved [S1, unofficial].

3.  **Splatting**
    Each PE independently generates a partial image from its partition of shaded slices.
    *   The PE determines the appropriate traversal order (front-to-back) and splatting filter based on the view plane orientation.
    *   Since the view is parallel, the orientation is consistent across all PEs.
    *   No data or work dependencies exist between PEs during this phase [S1, unofficial].

4.  **Compositing (Tiling Approach)**
    The partial images from all $p$ PEs are combined to form the final image using a distributed tiling strategy to avoid data dependencies and balance load.
    *   The image is conceptually divided into $p$ tiles.
    *   At compositing stage $t$ (where $t$ ranges from $1$ to $p$):
        *   Each PE $P(i)$ (where $i$ is the PE index, $0 \le i < p$) fetches the $i$-th tile from PE $P((i+t) \mod p)$.
        *   PE $P(i)$ composites this fetched tile with the corresponding $i$-th tile of its own local raster.
    *   This process continues for $t=1$ to $t=p$.
    *   After $p$ stages, each PE owns a $1/p$ tile of the completed final image [S1, unofficial].

5.  **Final Transfer**
    To minimize I/O gateway contention on the Cray T3D (which has limited I/O gateways to the host Y-MP), all tiles are transferred to a single designated PE. This PE is then responsible for transferring the complete final raster to the T3D host [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_splatting_t3d(volume_slices, num_pe, sample_resolution, view_params):
    """
    volume_slices: List of raw 2D slices (numpy arrays or similar)
    num_pe: Number of Processing Elements (p)
    sample_resolution: Bytes per component (1, 2, 4, or 8)
    view_params: Object containing view orientation and splatting filter
    """
    p = num_pe
    n = len(volume_slices)
    
    # Step 1: Data Distribution
    # Calculate slice counts for each PE
    base_slices = n // p
    extra_slices = n % p
    
    pe_slice_ranges = []
    current_idx = 0
    for i in range(p):
        count = base_slices + (1 if i < extra_slices else 0)
        pe_slice_ranges.append((current_idx, current_idx + count))
        current_idx += count
        
    # Each PE executes the following logic in parallel
    pe_id = get_pe_id() # 0 to p-1
    start_idx, end_idx = pe_slice_ranges[pe_id]
    local_raw_slices = volume_slices[start_idx:end_idx]
    
    # Step 2: Shading Slices
    local_shaded_slices = []
    for z in range(len(local_raw_slices)):
        raw_z = local_raw_slices[z]
        # Determine global slice index to access neighbors if needed
        # For CT, need z-1 and z+1 from the global volume
        # Since PEs hold contiguous blocks, neighbors might be in adjacent PEs' data
        # However, the source states PEs shade independently. 
        # Implementation note: Raw slices must be available. 
        # If neighbors are in other PEs, they must be communicated or the PE 
        # must have access to the global raw data buffer (or the shading 
        # logic is local to the block with boundary handling).
        # The source says "calculations... involve only... z-1, z, z+1" and 
        # "PE can shade its partition... independently". 
        # This implies the PE has access to the necessary raw data context.
        
        if is_ct_data():
            # Access global raw slices z-1, z, z+1
            # Note: z here is local index. Global index = start_idx + z
            global_z = start_idx + z
            raw_prev = get_global_slice(global_z - 1)
            raw_curr = raw_z
            raw_next = get_global_slice(global_z + 1)
            shaded_z = shade_ct_slice(raw_prev, raw_curr, raw_next, sample_resolution)
        else:
            # RGB Cryosection
            shaded_z = shade_rgb_slice(raw_z, sample_resolution)
            
        local_shaded_slices.append(shaded_z)
        
    # Step 3: Splatting
    # Each PE generates its own partial image
    local_image = splat_slices(local_shaded_slices, view_params)
    
    # Step 4: Compositing (Tiling Approach)
    # Divide local_image into p tiles
    tile_height = local_image.height // p
    tiles = [local_image[y*tile_height:(y+1)*tile_height, :] for y in range(p)]
    
    # Perform p stages of compositing
    for t in range(1, p + 1):
        # Fetch tile from PE P((i+t) mod p)
        source_pe_id = (pe_id + t) % p
        # Fetch the i-th tile (where i is pe_id) from source_pe_id
        # In a real implementation, this requires inter-PE communication (shmem/PVM)
        fetched_tile = fetch_tile(source_pe_id, pe_id)
        
        # Composite fetched tile with own tile
        tiles[pe_id] = composite(tiles[pe_id], fetched_tile)
        
    # After p stages, each PE owns one tile of the final image
    final_tile = tiles[pe_id]
    
    # Step 5: Final Transfer
    # If this is the designated PE, transfer to host
    if is_designated_pe():
        # Collect all tiles from all PEs (requires gathering)
        all_tiles = gather_all_tiles(final_tile)
        transfer_to_host(all_tiles)
    else:
        # Send tile to designated PE
        send_tile_to_designated(final_tile)
```

**Parameters**
*   **Sample Resolution:** 1, 2, 4, or 8 bytes per component [S1, unofficial].
*   **Number of PEs ($p$):** Must be $\le$ number of slices ($n$) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Slice Count vs. PE Count:** If $n < p$, the algorithm is not supported as slices cannot be split [S1, unofficial].
*   **I/O Bottleneck:** Transferring tiles from all PEs directly to the host causes contention on the limited I/O gateways. The single-PE transfer strategy mitigates this [S1, unofficial].
*   **Memory Management:** Raw slices must be discarded promptly to make room for shaded slices, as PE memory is limited [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The shading and splatting phases are $O(N/p)$ per PE, where $N$ is the total number of voxels. The compositing phase involves $p$ stages of communication and compositing, taking $O(p \cdot W \cdot H / p) = O(W \cdot H)$ total work distributed across PEs, with communication overhead proportional to $p$.
*   **Memory Complexity:** Each PE stores $\approx N/p$ shaded slices. With sample resolution $R$, memory per PE is $\approx \frac{N_x N_y N_z}{p} \times 4R$ bytes [S1, unofficial].
*   **Measured Performance:**
    *   512x512x1877 CT dataset (1 GB): 6.14 seconds on 128 PEs, 11.64 seconds on 64 PEs [S1, unofficial].
    *   1024x608x1878 RGBmale dataset (1.6 GB shaded slices): 11.8 seconds on 128 PEs, 23 seconds on 64 PEs [S1, unofficial].
*   **Scaling:** The method scales well with the number of PEs. Doubling the PEs roughly halves the render time [S1, unofficial].
*   **Strengths:**
    *   Significantly faster than single-node systems (e.g., SGI Onyx R4400 takes >17 minutes for the CT dataset) [S1, unofficial].
    *   Avoids re-reading and re-shading raw data for each image by keeping shaded slices in PE memory [S1, unofficial].
    *   Tiling compositing avoids data dependencies and balances workload [S1, unofficial].
*   **Limitations:**
    *   Requires $n \ge p$.
    *   Performance is dependent on hardware I/O gateway capacity [S1, unofficial].
    *   Inter-node data transfer is a potential bottleneck, though mitigated by the tiling strategy [S1, unofficial].

## Background
Parallel Splatting on Cray T3D, also known as **Splatter**, was developed by Greg Johnson and Jon Genetti at the Arctic Region Supercomputing Center, University of Alaska, Fairbanks [S1, unofficial]. It was designed to handle the large datasets from the Visible Human Project, which were too large for single-node systems to process efficiently. The system was integrated with the AVS visualization system for user control and Spline Animator for animation creation [S1, unofficial]. The implementation uses C, with parallelization via PVM (Parallel Virtual Machine) and shmem (Cray Shared Memory) libraries [S1, unofficial].

## Sources
[S1] Volume Rendering of Large Datasets on the Cray T3D, https://www.cs.uaf.edu/~genetti/Research/Papers/CUG96/large.html, fetched 2026-10-27 (unofficial)
