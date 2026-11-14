# Sparse Page Table and Brick Pool

Also called: Heterogeneous Volumes, MRMS sparse volume, Sparse Volume Textures (SVT), VDB-style sparse volume for web

## Overview
Sparse Page Table and Brick Pool is a sparse volume representation that avoids materialating a dense 3D grid by using a two-level structure: a page table mapping logical $8^3$-texel regions to resident bricks, and a pool of apron-padded bricks [S3, unofficial]. This allows full source resolution (up to 2048 cells per axis) while bounding memory usage and enabling empty-space skipping [S3, unofficial]. It builds on [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md) by replacing generic hierarchical traversal with a fixed-size page table lookup, trading rendering speed for data compression [S1, S2, unofficial]. Unlike [ExaBricks Hierarchy](2182-exabricks-hierarchy.md) which uses a deep octree, this method uses a flat page table for spatial subdivision [S3, unofficial].

## How it works
**Inputs:** A sparse voxel dataset (e.g., OpenVDB) with logical dimensions up to $2048^3$.
**Outputs:** A page table texture and a brick pool texture.

**Data Structures:**
1.  **Page Table:** A 3D volume texture where each entry corresponds to a logical $8^3$-texel page. The entry format is RG8 (16 bits):
    *   If the page is empty (no echo/data): value is `0`.
    *   If the page contains data: value is `pool_slot_index + 1` (1-based index into the brick pool) [S3, unofficial].
2.  **Brick Pool:** A collection of resident bricks. Each brick consists of:
    *   **Core:** $8^3$ texels of the original data.
    *   **Apron:** A one-texel border surrounding the core, making the total brick size $10^3$ texels. The apron is filled by copying neighboring logical texels, clamped at the grid edge, to ensure trilinear filtering stays within the brick [S3, unofficial].

**Parameters:**
*   `MAX_VOLUME_BRICKS`: Maximum number of resident bricks allowed in the pool. Default is 8192 [S3, unofficial]. This caps the pool size at approximately 16.4 MB (assuming 2 KB per brick) [S3, unofficial].
*   `Extinction Power`: The extinction ramp uses a power function `pow(t, 2.75)` to correct for light echo inflation inherent in source-resolution rendering [S3, unofficial].
*   `MAX_RAY_STEPS`: Maximum ray steps per pixel. Remains at 384 [S3, unofficial].

**Build Phase (CPU/Compute):**
1.  Iterate through the logical grid in $8^3$ pages.
2.  For each page:
    *   If the page is empty, set the page table entry to `0`.
    *   If the page contains data:
        *   If the current number of resident bricks is less than `MAX_VOLUME_BRICKS`:
            *   Allocate a new brick in the pool.
            *   Copy the $8^3$ core data into the brick.
            *   Fill the one-texel apron from neighboring logical texels (clamped at edges).
            *   Set the page table entry to `allocated_slot_index + 1`.
        *   If the pool is full:
            *   Perform a count-only pass to determine how many pages would fit if the grid were coarsened.
            *   Coarsen the grid by the next whole footprint multiple [S3, unofficial].
3.  **Terrain Occlusion Handling:** `buildGroundPageMax` computes the highest ground height under each page column (including a one-column halo for linear filtering). An empty page is only skipped if the entire page lies wholly above this maximum ground height [S3, unofficial].

**Raymarching Step (GPU Shader):**
1.  For each ray step, calculate the logical page index for the current ray position.
2.  Fetch the corresponding RG8 entry from the page table.
3.  **If entry == 0 (Empty Page):**
    *   Jump the ray to the exit face of that $8^3$ page in one loop iteration.
    *   Adjust the ray position to land back on the jittered sample lattice to maintain consistent sampling cadence [S3, unofficial].
    *   **Terrain Check:** If the page is not wholly above the `buildGroundPageMax` height, step through the page with per-sample ground tests but without volume fetches [S3, unofficial].
4.  **If entry != 0 (Resident Page):**
    *   Lookup the brick in the pool at index `(entry - 1)`.
    *   Sample the volume trilinearly using the brick data [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Build Phase
def build_volume_texture(logical_grid, max_bricks=8192):
    page_table = {} # Maps (x, y, z) page index to brick index
    brick_pool = [] # List of bricks, each brick is 10^3 texels
    
    # Iterate through logical grid in 8x8x8 pages
    for page_coords in logical_grid.get_page_iterator():
        if not logical_grid.is_page_empty(page_coords):
            if len(brick_pool) < max_bricks:
                brick_index = len(brick_pool)
                # Allocate brick
                core = logical_grid.get_page_data(page_coords) # 8^3 texels
                apron = fill_apron(core, logical_grid, page_coords) # 10^3 texels total
                brick_pool.append(apron)
                page_table[page_coords] = brick_index + 1 # 1-based index
            else:
                # Pool full, coarsen grid
                coarsening_factor = determine_coarsening(logical_grid, max_bricks)
                return build_volume_texture(logical_grid.coarsen(coarsening_factor), max_bricks)
        else:
            page_table[page_coords] = 0 # Empty page
            
    return page_table, brick_pool

def fill_apron(core, logical_grid, page_coords):
    # Create 10x10x10 buffer
    brick = zeros(10, 10, 10)
    # Copy core to center (indices 1..8)
    brick[1:9, 1:9, 1:9] = core
    # Fill apron with clamped neighbors
    for axis in [0, 1, 2]:
        for direction in [-1, 1]:
            # Logic to copy neighbor texels, clamped at grid edge
            # [G] Specific implementation details for apron filling are omitted as they are standard texture border handling
            pass
    return brick

# Raymarching Step
def raymarch_step(ray_position, page_table, brick_pool):
    page_idx = get_page_index(ray_position)
    entry = page_table.get(page_idx, 0)
    
    if entry == 0:
        # Empty page
        page_exit = get_page_exit_face(ray_position, page_idx)
        # Jump to exit and re-jitter to lattice
        ray_position = jump_to_lattice(page_exit)
        return ray_position, 0 # No sample
        
    # Resident page
    brick_index = entry - 1
    brick = brick_pool[brick_index]
    local_pos = get_local_position_in_brick(ray_position, page_idx)
    density = trilinear_sample(brick, local_pos)
    return ray_position, density
```

## Subtopics

### [Sparse Brickmap Volume Rendering](2978-sparse-brickmap-volume-rendering.md)
Sparse Brickmap Volume Rendering is a voxel rendering technique that uses a two-level data structure (coarse brick grid and fine voxel bricks) to render sparse volumes efficiently, integrating hardware depth for correct compositing with rasterized geometry. It builds on [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md) by replacing the generic page-table indirection with a fixed $8^3$ brick hierarchy optimized for two-level Digital Differential Analyzer (DDA) traversal. Unlike dense voxel grids, it significantly reduces memory footprint by collapsing fully-buried terrain into uniform entries and storing only mixed (surface) bricks in a compact pool.

### [Multi-Resolution Multi-Channel Page Table (MRMCPT)](3189-multi-resolution-multi-channel-page-table-mrmcpt.md)
Multi-Resolution Multi-Channel Page Table (MRMCPT) is an out-of-core volume rendering method that manages a multi-resolution hierarchy using page tables for spatial subdivision and cache residency. It builds on [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md) by organizing the mapping of logical voxel regions to resident bricks into a hierarchical structure that supports multiple resolution levels. Unlike [Sparse Brickmap Volume Rendering](2978-sparse-brickmap-volume-rendering.md) which may use different spatial indexing, MRMCPT relies on explicit page table lookups to translate virtual addresses to physical brick locations. Its primary distinction is the use of a page-table-based residency management system rather than a tree traversal or hash-based approach, which results in limited empty-space skipping capabilities compared to octree-based methods.

### Memory Pool Sizing
In related implementations, the node pool is typically small (4 MB), corresponding to $64^3$ entries. Using $16^3$ bricks, $1024^3$ indices can be referenced. The brick pool uses 430 MB, giving room for $42^3$ bricks [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The effective displayed volume size is reduced by a factor of $4^3 = 64$ compared to the original dataset due to the page table lookup and sparse storage [S1, unofficial]. The memory footprint is quadrupled compared to a scalar density value [S1, unofficial].
*   **Time:** The sparse representation reduces per-pixel work by skipping empty pages, but the maximum step count (`MAX_RAY_STEPS`) remains at 384 [S3, unofficial].

**Measured Performance:**
*   On a live KATL weather payload ($456 \times 385 \times 96$ logical texels at 1× coarsening), the system used 2,158 bricks and a 5.0 MB pool [S3, unofficial].
*   Rendering the Kolumbo dataset (2.6 billion non-empty voxels) without artifacts at about 20 fps [S2, unofficial].
*   Comparing old dense grid vs. new sparse source-resolution renders on live KATL (wide) data: Old 6.11M, Cubic at source res 5.08M (-17%), 2.75 power 6.12M (0%) [S3, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Supports full source resolution (up to 2048 cells per axis) [S3, unofficial]. Bounded memory usage via `MAX_VOLUME_BRICKS` [S3, unofficial]. Efficient empty-space skipping [S3, unofficial].
*   **Limitations:** Trilinear filtering requires an apron, increasing brick size from $8^3$ to $10^3$ [S3, unofficial]. Terrain occlusion requires additional computation (`buildGroundPageMax`) to prevent rays from passing through terrain when skipping empty pages [S3, unofficial]. Importing large VDB files (>2GB) can cause crashes due to integer overflows if not handled with `uint32` [S2, unofficial].

## Background
Sparse Page Table and Brick Pool is an experimental feature in Unreal Engine known as Heterogeneous Volumes or Sparse Volume Textures (SVT) [S1, S2, unofficial]. It was introduced to handle large general-purpose volume data more efficiently than previous methods like Volumetric Fog or Volumetric Cloud [S2, unofficial]. The method trades rendering speed for data compression by storing only non-empty 3D tiles [S1, unofficial]. It is particularly useful for rendering large datasets like the Kolumbo dataset (2.6 billion non-empty voxels) [S2, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S2] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S3] Render the MRMS volume through a sparse page table and brick pool, https://github.com/andyfangdz/approach-viz/pull/74, fetched 2026-10-28 (unofficial)
[S4] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
