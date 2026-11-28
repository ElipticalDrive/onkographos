# Parallel Splatting on Cray T3D

Also called: Splatter, T3D volume rendering, VolRender parallel splatting
## Overview
Parallel Splatting on Cray T3D is a memory-optimized volume rendering method designed for the Cray T3D massively parallel supercomputer [S1, unofficial]. It projects volumetric data onto a raster using splatting, distributing both the data and the computational workload across multiple Processing Elements (PEs) to render large medical datasets, such as the Visible Human Project, at interactive rates [S1, unofficial]. It distinguishes itself from other parallel splatting approaches by its specific tiling-based compositing strategy and its optimization for the Cray T3D's distributed memory architecture, which allows shaded slices to remain in PE memory, avoiding the repeated I/O and shading overhead seen in single-node systems like the SGI Onyx [S1, unofficial].
## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume dataset consisting of raw slices (CT or RGB cryosection) [S1, unofficial].
*   **Output:** A final 2D raster image (framebuffer) representing the volume projection [S2, unofficial].

**Data Structures**
*   **Volume Data:** Stored as a sequence of 2D slices [S1, unofficial].
*   **Shaded Slice:** A 2D array of voxels, each containing 4 components (R, G, B, A) [S1, unofficial].
*   **Sample Resolution:** The number of bytes per component [S1, unofficial]. Supported values are 1, 2, 4, or 8 bytes [S1, unofficial].
*   **Voxel Size:** $V_{size} = 4 \times \text{Sample Resolution}$ bytes [S1, unofficial].
*   **Slice Size:** $S_{size} = N_x \times N_y \times V_{size}$ bytes [S1, unofficial].
*   **Processing Elements (PEs):** $p$ parallel processors [S1, unofficial].
*   **Tiles:** The output image is divided into $p$ vertical (or horizontal, depending on implementation) tiles, each of size $1/p$ of the total image area [S1, unofficial].
*   **Output Format:** The output of the VolRender module is an AVS 2D 4-vector byte regular field [S2, unofficial].

**Algorithm Steps**

1.  **Data Distribution**
    The volume is partitioned into contiguous regions of slices [S1, unofficial]. Let $n$ be the total number of slices and $p$ be the number of PEs [S1, unofficial]. The slices are distributed such that:
    *   The first $k = n - \lfloor n/p \rfloor \times p$ PEs receive $\lfloor n/p \rfloor + 1$ slices.
    *   The remaining $p - k$ PEs receive $\lfloor n/p \rfloor$ slices.
    *   Constraint: $n \ge p$. No slice is split between PEs [S1, unofficial].

2.  **Shading Slices**
    Each PE processes its assigned slices independently [S1, unofficial].
    *   **CT Slices:** For each raw slice $z$, the PE calculates the shaded value using raw slices $z-1$, $z$, and $z+1$ [S1, unofficial]. Raw slices are discarded once they are no longer needed to free memory for new shaded slices [S1, unofficial].
    *   **RGB Cryosection Slices:** Only the alpha (transparency) value is calculated for each voxel [S1, unofficial]. The alpha value is set to a small value based on the voxel's color to enhance red muscle tissue and internal organs [S1, unofficial]. RGB values are preserved [S1, unofficial].

3.  **Splatting**
    Each PE independently generates a partial image from its partition of shaded slices [S1, unofficial].
    *   The PE determines the appropriate traversal order (front-to-back) and splatting filter based on the view plane orientation [S1, unofficial].
    *   Since the view is parallel, the orientation is consistent across all PEs [S1, unofficial].
    *   No data or work dependencies exist between PEs during this phase [S1, unofficial].

4.  **Compositing (Tiling Approach)**
    The partial images from all $p$ PEs are combined to form the final image using a distributed tiling strategy to avoid data dependencies and balance load [S1, unofficial].
    *   The image is conceptually divided into $p$ tiles [S1, unofficial].
    *   At compositing stage $t$ (where $t$ ranges from $1$ to $p$):
        *   Each PE $P(i)$ (where $i$ is the PE index, $0 \le i < p$) fetches the $i$-th tile from PE $P((i+t) \mod p)$ [S1, unofficial].
        *   PE $P(i)$ composites this fetched tile with the corresponding $i$-th tile of its own local raster [S1, unofficial].
    *   This process continues for $t=1$ to $t=p$ [S1, unofficial].
    *   After $p$ stages, each PE owns a $1/p$ tile of the completed final image [S1, unofficial].

5.  **Final Transfer**
    To minimize I/O gateway contention on the Cray T3D (which has limited I/O gateways to the host Y-MP), all tiles are transferred to a single designated PE [S1, unofficial]. This PE is then responsible for transferring the complete final raster to the T3D host [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_splatting_t3d(raw_slices, num_pe, sample_resolution, view_params):
    """
    raw_slices: List of 2D arrays (z, y, x) representing raw volume data [S1, unofficial]
    num_pe: p, number of Processing Elements [S1, unofficial]
    sample_resolution: Bytes per component (1, 2, 4, or 8) [S1, unofficial]
    view_params: Contains view direction, projection matrix, and splat filter kernel [S2, unofficial]
    """
    p = num_pe
    n = len(raw_slices)
    assert n >= p, "Number of slices must be >= number of PEs [S1, unofficial]"

    # Step 1: Data Distribution [S1, unofficial]
    base_slices = n // p
    extra_slices = n % p
    pe_id = get_pe_id() # [G] standard parallel runtime call
    if pe_id < extra_slices:
        start_z = pe_id * (base_slices + 1)
        count = base_slices + 1
    else:
        start_z = extra_slices * (base_slices + 1) + (pe_id - extra_slices) * base_slices
        count = base_slices
    local_raw_indices = range(start_z, start_z + count)

    # Step 2: Shading Slices [S1, unofficial]
    local_shaded_slices = []
    is_ct = check_data_type(raw_slices[0]) # [G] standard type check
    for local_idx, global_z in enumerate(local_raw_indices):
        if is_ct:
            # CT shading requires neighbors z-1, z, z+1 [S1, unofficial]
            raw_prev = raw_slices[global_z - 1] if global_z > 0 else raw_slices[global_z]
            raw_curr = raw_slices[global_z]
            raw_next = raw_slices[global_z + 1] if global_z < n - 1 else raw_slices[global_z]
            shaded = shade_ct_slice(raw_prev, raw_curr, raw_next, sample_resolution)
        else:
            # RGB cryosection: only alpha calculated [S1, unofficial]
            shaded = shade_rgb_slice(raw_slices[global_z], sample_resolution)
        local_shaded_slices.append(shaded)
        # Discard raw slices when no longer needed to free memory [S1, unofficial]
        if global_z > 0:
            raw_slices[global_z - 1] = None

    # Step 3: Splatting [S1, unofficial]
    # Front-to-back traversal, parallel projection, constant footprint [S2, unofficial]
    local_image = splat_slices(local_shaded_slices, view_params)

    # Step 4: Compositing (Tiling Approach) [S1, unofficial]
    # Divide local_image into p tiles [S1, unofficial]
    tile_h = local_image.height // p
    my_tile_idx = pe_id
    my_tile = local_image[my_tile_idx * tile_h : (my_tile_idx + 1) * tile_h, :]

    # p stages of compositing [S1, unofficial]
    for t in range(1, p + 1):
        source_pe = (pe_id + t) % p
        # Fetch tile from source PE [G] inter-processor communication via shmem/PVM
        fetched_tile = fetch_tile_from_pe(source_pe, my_tile_idx)
        # Composite fetched tile with own tile (front-to-back blend) [S2, unofficial]
        my_tile = composite_tiles(my_tile, fetched_tile)

    final_tile = my_tile

    # Step 5: Final Transfer [S1, unofficial]
    # Transfer to designated PE to avoid I/O gateway contention [S1, unofficial]
    designated_pe = 0 # [G] arbitrary choice, typically PE 0
    if pe_id == designated_pe:
        # Gather all tiles from other PEs [G]
        all_tiles = [fetch_tile_from_pe(pid, 0) for pid in range(p)]
        # Reassemble and transfer to host [S1, unofficial]
        final_image = reassemble_tiles(all_tiles)
        transfer_to_host(final_image)
    else:
        send_tile_to_pe(designated_pe, final_tile)

def shade_ct_slice(raw_prev, raw_curr, raw_next, sample_resolution):
    """Calculates shaded CT slice using Phong model and surface normals [S2, unofficial]."""
    shaded = []
    for k in range(raw_curr.shape[0]):
        row = []
        for j in range(raw_curr.shape[1]):
            # Calculate approximate surface normal N(i,j,k) [S2, unofficial]
            nx = raw_prev[k][j] - raw_next[k][j]
            ny = raw_prev[k][j] - raw_next[k][j] # [G] placeholder for y-dimension gradient
            nz = raw_curr[k][j-1] - raw_curr[k][j+1] # [G] placeholder for z-dimension gradient
            # Apply Phong reflection model to compute color and alpha [S2, unofficial]
            color = compute_phong_color(nx, ny, nz, raw_curr[k][j])
            alpha = compute_alpha_from_normal(nx, ny, nz)
            row.append((color[0], color[1], color[2], alpha))
        shaded.append(row)
    return shaded

def shade_rgb_slice(raw_rgb, sample_resolution):
    """Calculates alpha for RGB cryosection slices [S1, unofficial]."""
    shaded = []
    for k in range(raw_rgb.shape[0]):
        row = []
        for j in range(raw_rgb.shape[1]):
            r, g, b = raw_rgb[k][j]
            # Set alpha to a small value based on voxel color to enhance red muscle tissue [S1, unofficial]
            alpha = compute_small_alpha_from_color(r, g, b)
            row.append((r, g, b, alpha))
        shaded.append(row)
    return shaded

def splat_slices(shaded_slices, view_params):
    """Projects shaded slices onto view plane using front-to-back traversal [S2, unofficial]."""
    image = create_empty_image(view_params.width, view_params.height)
    for slice in shaded_slices:
        for voxel in slice:
            # Project voxel to view plane coordinates [S2, unofficial]
            px, py = project_voxel_to_pixel(voxel, view_params)
            # Apply filter to spread color and transparency to neighboring pixels [S2, unofficial]
            apply_splat_filter(image, px, py, voxel.color, voxel.alpha, view_params.filter_kernel)
    return image

def composite_tiles(tile_a, tile_b):
    """Composites two tiles using front-to-back blending [S2, unofficial]."""
    result = []
    for y in range(tile_a.height):
        row = []
        for x in range(tile_a.width):
            ca, aa = tile_a[y][x]
            cb, ab = tile_b[y][x]
            # Front-to-back alpha compositing [G]
            ra = ca * aa + cb * ab * (1 - aa)
            result.append((ra, ab))
        result.append(row)
    return result

def fetch_tile_from_pe(source_pe, tile_idx):
    """Fetches a specific tile from another PE using shmem/PVM [G]."""
    # [G] Implementation detail: uses Cray shmem or PVM to retrieve tile_idx from source_pe
    pass

def get_pe_id():
    """Returns the unique ID of the current processing element [G]."""
    # [G] Implementation detail: standard parallel runtime call
    pass

def check_data_type(slice):
    """Determines if slice is CT or RGB cryosection [G]."""
    # [G] Implementation detail: checks metadata or header
    pass

def compute_phong_color(nx, ny, nz, density):
    """Computes color using Phong reflection model [S2, unofficial]."""
    # [G] Standard graphics operation
    pass

def compute_alpha_from_normal(nx, ny, nz):
    """Computes alpha based on normal magnitude; small difference implies homogeneous mass [S2, unofficial]."""
    # [G] Standard graphics operation
    pass

def compute_small_alpha_from_color(r, g, b):
    """Sets alpha to a small value based on color to enhance red muscle tissue [S1, unofficial]."""
    # [G] Standard graphics operation
    pass

def project_voxel_to_pixel(voxel, view_params):
    """Projects voxel coordinates to 2D image plane [S2, unofficial]."""
    # [G] Standard graphics operation
    pass

def apply_splat_filter(image, px, py, color, alpha, kernel):
    """Spreads color and transparency to neighboring pixels using a filter [S2, unofficial]."""
    # [G] Standard graphics operation
    pass

def create_empty_image(width, height):
    """Creates an empty 2D image buffer [G]."""
    # [G] Standard graphics operation
    pass

def reassemble_tiles(tiles):
    """Reassembles p tiles into a single complete image [G]."""
    # [G] Standard graphics operation
    pass

def transfer_to_host(image):
    """Transfers the final raster to the T3D host [S1, unofficial]."""
    # [G] Standard I/O operation
    pass

def send_tile_to_pe(designated_pe, tile):
    """Sends a tile to the designated PE for final assembly [G]."""
    # [G] Standard communication operation
    pass
```

**Parameters**
*   **Sample Resolution:** 1, 2, 4, or 8 bytes per component [S1, unofficial].
*   **Number of PEs ($p$):** Must be $\le$ number of slices ($n$) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Slice Count vs. PE Count:** If $n < p$, the algorithm is not supported as slices cannot be split [S1, unofficial].
*   **I/O Bottleneck:** Transferring tiles from all PEs directly to the host causes contention on the limited I/O gateways. The single-PE transfer strategy mitigates this [S1, unofficial].
*   **Memory Management:** Raw slices must be discarded promptly to make room for shaded slices, as PE memory is limited [S1, unofficial].
## Operating characteristics
*   **Time Complexity:** The shading and splatting phases are $O(N/p)$ per PE, where $N$ is the total number of voxels [S1, unofficial]. The compositing phase involves $p$ stages of communication and compositing, taking $O(W \cdot H)$ total work distributed across PEs, with communication overhead proportional to $p$ [S1, unofficial].
*   **Memory Complexity:** Each PE stores $\approx N/p$ shaded slices. For a 512x512 slice, this results in a shaded slice of 1, 2, 4 or 8 MB depending on sample resolution [S1, unofficial].
*   **Measured Performance:**
    *   512x512x1877 CT dataset (1 GB): 6.14 seconds on 128 PEs, 11.64 seconds on 64 PEs [S1, unofficial].
    *   1024x608x1878 RGBmale dataset (1.6 GB shaded slices): 11.8 seconds on 128 PEs, 23 seconds on 64 PEs [S1, unofficial].
*   **Scaling:** The method scales well with the number of PEs. Doubling the PEs roughly halves the render time [S1, unofficial].
*   **Strengths:**
    *   Significantly faster than single-node systems (e.g., SGI Onyx R4400 takes >17 minutes for the CT dataset) [S1, unofficial].
    *   Avoids re-reading and re-shading raw data for each image by keeping shaded slices in PE memory [S1, unofficial].
    *   Tiling compositing avoids data dependencies and balances workload [S1, unofficial].
*   **Limitations:**
    *   Requires $n \ge p$ [S1, unofficial].
    *   Performance is dependent on hardware I/O gateway capacity [S1, unofficial].
    *   Inter-node data transfer is a potential bottleneck, though mitigated by the tiling strategy [S1, unofficial].
## Background
Parallel Splatting on Cray T3D, also known as **Splatter**, was developed by Greg Johnson and Jon Genetti at the Arctic Region Supercomputing Center, University of Alaska, Fairbanks [S1, unofficial]. It was designed to handle the large datasets from the Visible Human Project, which were too large for single-node systems to process efficiently [S1, unofficial]. The system was integrated with the AVS visualization system for user control and Spline Animator for animation creation [S1, unofficial]. The implementation uses C, with parallelization via PVM (Parallel Virtual Machine) and shmem (Cray Shared Memory) libraries [S1, unofficial].
## Sources
[S1] Volume Rendering of Large Datasets on the Cray T3D, https://www.cs.uaf.edu/~genetti/Research/Papers/CUG96/large.html, fetched 2026-10-27 (unofficial)
[S2] Medical Diagnosis using the Cray T3D, https://www.cs.uaf.edu/~genetti/Research/Papers/CUG95/medical.html, fetched 2026-10-29 (unofficial)
