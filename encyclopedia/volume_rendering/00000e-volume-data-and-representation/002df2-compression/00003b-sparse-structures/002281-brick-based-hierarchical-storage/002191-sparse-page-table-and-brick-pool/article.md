# Sparse Page Table and Brick Pool

Also called: Heterogeneous Volumes, MRMS sparse volume, Sparse Volume Textures (SVT), Sparse Volume Textures (SVT) / Heterogeneous Volumes, VDB-style sparse volume for web

## Overview
Sparse Page Table and Brick Pool is a sparse volume representation that avoids materialating a dense 3D grid by using a two-level structure: a page table mapping logical $8^3$-texel regions to resident bricks, and a pool of apron-padded bricks [S3, unofficial]. This allows full source resolution (up to 2048 cells per axis) while bounding memory usage and enabling empty-space skipping [S3, unofficial]. It represents a trade-off between rendering speed and data compression by storing only non-empty 3D tiles [S1, unofficial]. In Unreal Engine, this method is exposed as the "Heterogeneous Volumes" feature [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel dataset (e.g., OpenVDB) with logical resolution up to $2048^3$ [S3, unofficial].
*   **Output:** Two GPU textures: a "Physical Tile Data Texture" (Brick Pool) and a "Page Table Volume Texture" [S1, S2, unofficial].

**Data Structures:**
1.  **Page Table:** A 3D volume texture where each entry corresponds to a logical $8^3$-texel page.
    *   **Format:** RG8 (8-bit Red, 8-bit Green).
    *   **Layout:** One entry per logical page.
    *   **Encoding:**
        *   Value `0`: The page is empty (no echo/data).
        *   Value `>0`: The page contains data. The value encodes the pool slot index + 1 [S3, unofficial].
2.  **Brick Pool:** A collection of resident bricks stored densely in a volume texture.
    *   **Brick Layout:** Each brick consists of an $8^3$ texel core wrapped in a one-texel apron, resulting in a total size of $10^3$ texels [S3, unofficial].
    *   **Apron:** The one-texel apron is filled by copying neighboring logical texels, clamped at the grid edge, to ensure trilinear filtering stays within the brick [S3, unofficial].
    *   **Capacity:** Capped at `MAX_VOLUME_BRICKS` (default 8192) [S3, unofficial]. This caps the pool size at approximately 16.4 MB (assuming 2 KB per brick) [S3, unofficial].

**Parameters:**
*   `MAX_VOLUME_BRICKS`: 8192 [S3, unofficial].
*   `MAX_RAY_STEPS`: 384 [S3, unofficial].
*   `Extinction Power`: 2.75 (uses `pow(t, 2.75)`) to correct for light echo inflation compared to dense max-pooled grids [S3, unofficial].
*   Logical Page Size: $8^3$ texels [S3, unofficial].

**Build Phase (CPU/Host):**
The `build_volume_texture` function emits the page table and brick pool [S3, unofficial].
1.  Iterate through the logical grid.
2.  For each logical $8^3$ page:
    *   If the page is empty, set the page table entry to 0 [S3, unofficial].
    *   If the page contains data:
        *   Check if the brick pool has space (current count < `MAX_VOLUME_BRICKS`) [S3, unofficial].
        *   If space is available:
            *   Allocate a slot in the pool.
            *   Copy the $8^3$ core data into the slot.
            *   Fill the one-texel apron from neighboring logical texels (clamped at edges) [S3, unofficial].
            *   Set the page table entry to `slot_index + 1` [S3, unofficial].
        *   If the pool is full:
            *   Perform a count-only pass to determine coarsening needs.
            *   Coarsen the grid by the next whole footprint multiple [S3, unofficial].
            *   Re-evaluate pages in the coarsened grid.

**Raymarching Step (GPU Shader):**
1.  Calculate the logical page index for the current ray position [S3, unofficial].
2.  Fetch the corresponding RG8 entry from the Page Table [S3, unofficial].
3.  **If entry == 0 (Empty Page):**
    *   Jump the ray to the exit face of that $8^3$ page in one loop iteration [S3, unofficial].
    *   Adjust the ray position to land back on the jittered sample lattice to maintain consistent sampling cadence [S3, unofficial].
    *   **Terrain Occlusion Check:** Before skipping, check if the page lies wholly above the terrain. `buildGroundPageMax` computes the highest ground height under each page column (including a one-column halo for linear filtering). If the page is not wholly above the terrain, do not skip; instead, step through the page with per-sample ground tests but without volume fetches [S3, unofficial].
4.  **If entry != 0 (Resident Page):**
    *   Lookup the brick in the pool at index `(entry - 1)` [S3, unofficial].
    *   Sample the volume trilinearly using the brick data [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_VOLUME_BRICKS = 8192
PAGE_SIZE = 8
APRON_SIZE = 1  # One-texel apron
EXTINCTION_POWER = 2.75

# Data Structures
# PageTable: 3D texture, RG8 format. Entry 0 = empty, Entry > 0 = pool_index + 1
# BrickPool: 3D texture, dense storage of 10^3 texel bricks.
# GroundPageMax: 2D texture, stores max ground height per page column.

def build_volume_texture(logical_voxels, max_ground_heights):
    """
    Builds the Page Table and Brick Pool from logical voxel data.
    logical_voxels: Sparse map of (x,y,z) -> value.
    max_ground_heights: 2D array of max ground height per (x,y) column.
    """
    page_table = {}
    brick_pool = []
    
    # Iterate over logical grid pages
    for page_x, page_y, page_z in logical_voxels.pages():
        # Check if page has any non-empty voxels
        if not logical_voxels.page_has_data(page_x, page_y, page_z):
            page_table[(page_x, page_y, page_z)] = 0
            continue
        
        # Check pool capacity
        if len(brick_pool) >= MAX_VOLUME_BRICKS:
            # Pool full: trigger coarsening logic (simplified)
            # In practice, this involves a count-only pass and re-gridding.
            # For this implementation, we assume coarsening handles the overflow
            # by reducing resolution or skipping lower-priority bricks.
            # Here we simply skip adding to pool if full, effectively treating as empty
            # or handled by the coarsening pass mentioned in sources.
            # [G]: In a full implementation, this block would trigger grid coarsening.
            page_table[(page_x, page_y, page_z)] = 0
            continue
            
        # Allocate brick
        slot_index = len(brick_pool)
        brick = create_brick(logical_voxels, page_x, page_y, page_z)
        brick_pool.append(brick)
        
        # Update page table
        page_table[(page_x, page_y, page_z)] = slot_index + 1

    return page_table, brick_pool

def create_brick(logical_voxels, page_x, page_y, page_z):
    """
    Creates a 10^3 brick with 8^3 core and 1-texel apron.
    """
    brick = empty_3d_array(10, 10, 10)
    
    # Copy core (8^3)
    for dx in range(8):
        for dy in range(8):
            for dz in range(8):
                lx = page_x * 8 + dx
                ly = page_y * 8 + dy
                lz = page_z * 8 + dz
                brick[dx+1, dy+1, dz+1] = logical_voxels.get(lx, ly, lz)
                
    # Fill apron (1-texel border)
    # Core is at indices 1..8. Apron is at 0 and 9.
    for x in range(10):
        for y in range(10):
            for z in range(10):
                if x == 0 or x == 9 or y == 0 or y == 9 or z == 0 or z == 9:
                    # Clamp to nearest core voxel
                    cx = clamp(x - 1, 1, 8)
                    cy = clamp(y - 1, 1, 8)
                    cz = clamp(z - 1, 1, 8)
                    brick[x, y, z] = brick[cx, cy, cz]
                    
    return brick

def raymarch_ray(ray_origin, ray_direction, page_table, brick_pool, ground_page_max):
    """
    Raymarches through the sparse volume.
    """
    t = 0.0
    max_steps = 384
    step_size = 1.0 / 2048.0 # Assuming normalized coordinates
    
    for _ in range(max_steps):
        if t > 1.0:
            break
            
        # Calculate logical page index
        pos = ray_origin + ray_direction * t
        page_x = floor(pos.x / 8)
        page_y = floor(pos.y / 8)
        page_z = floor(pos.z / 8)
        
        # Fetch page table entry
        entry = page_table.get((page_x, page_y, page_z), 0)
        
        if entry == 0:
            # Empty page
            # Check terrain occlusion
            # [G]: ground_page_max is 2D, so we use page_x, page_y
            max_ground = ground_page_max.get((page_x, page_y), 0)
            # If the entire page is above the max ground height, skip it
            # Page Z range is [page_z * 8, (page_z + 1) * 8)
            if page_z * 8 >= max_ground:
                # Jump to exit of page
                t += (1.0 - pos.z) / ray_direction.z if ray_direction.z != 0 else float('inf')
                # Adjust to lattice
                t = adjust_to_lattice(t)
                continue
            else:
                # Step through without volume fetch
                t += step_size
                continue
        else:
            # Resident page
            slot_index = entry - 1
            brick = brick_pool[slot_index]
            
            # Sample trilinearly
            # Map ray position to local brick coordinates (0..9)
            local_pos = (pos - (page_x * 8, page_y * 8, page_z * 8)) * 1.0 # Scale to brick size
            # [G]: Trilinear sampling logic depends on specific texture coordinate mapping.
            # Assuming brick is stored as 10^3 texture.
            value = trilinear_sample(brick, local_pos)
            
            # Apply extinction
            alpha = 1.0 - pow(1.0 - value, EXTINCTION_POWER)
            
            # Composite
            # [G]: Standard front-to-back compositing
            t += step_size
            
    return accumulated_color

def adjust_to_lattice(t):
    """
    Adjusts ray position to land on jittered sample lattice.
    """
    # [G]: Specific lattice adjustment algorithm not detailed in sources.
    # Placeholder for maintaining sampling cadence.
    return t
```

## Subtopics

### [Memory Pool Sizing](2898-memory-pool-sizing.md)
Memory pool sizing defines the capacity and allocation limits for the node pool and brick pool in sparse volume representations, bounding memory usage while enabling the referencing of large logical volumes. It sets apart from dense grid approaches by decoupling logical resolution from resident memory, allowing full source resolution (up to $2048^3$ cells per axis) while skipping empty space.

### [Sparse Brickmap Volume Rendering](2978-sparse-brickmap-volume-rendering.md)
Sparse Brickmap Volume Rendering is a voxel rendering technique that uses a two-level data structure (coarse brick grid and fine voxel bricks) to render sparse volumes efficiently, integrating hardware depth for correct compositing with rasterized geometry. It builds on [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md) by replacing the generic page-table indirection with a fixed $8^3$ brick hierarchy optimized for two-level Digital Differential Analyzer (DDA) traversal. Unlike dense voxel grids, it significantly reduces memory footprint by collapsing fully-buried terrain into uniform entries and storing only mixed (surface) bricks in a compact pool.

### [Multi-Resolution Multi-Channel Page Table (MRMCPT)](3189-multi-resolution-multi-channel-page-table-mrmcpt.md)
Multi-Resolution Multi-Channel Page Table (MRMCPT) is a baseline out-of-core volume rendering method that manages a multi-resolution hierarchy using page tables for spatial subdivision and cache residency. It relies on explicit page table lookups to translate virtual addresses to physical brick locations, distinguishing it from tree traversal or hash-based spatial indexing. Its primary trade-off is limited empty-space skipping capabilities compared to octree-based methods, which results in higher cache usage and lower performance on sparse or multi-channel datasets.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The memory footprint is bounded by the number of resident bricks (`MAX_VOLUME_BRICKS`) and the page table size. The page table size is proportional to the logical volume resolution ($N^3 / 8^3$ entries). The brick pool is capped at 8192 bricks, resulting in ~16.4 MB [S3, unofficial]. The effective displayed volume size is reduced by a factor of $4^3 = 64$ compared to the original dataset due to the page table lookup and sparse storage [S1, unofficial].
*   **Time:** Rendering time is reduced by empty-space skipping. The maximum ray steps (`MAX_RAY_STEPS`) remain at 384, but the number of actual volume fetches is reduced [S3, unofficial].

**Measured Performance:**
*   **Kolumbo Dataset:** Rendering the Kolumbo dataset (2.6 billion non-empty voxels) achieved about 20 fps after fixes [S2, unofficial].
*   **Smoke Test:** On a live KATL weather payload (456 × 385 × 96 logical texels at 1× coarsening), the system used 2,158 bricks and a 5.0 MB pool [S3, unofficial].
*   **Visual Accuracy:** Comparing old dense grid vs. new sparse source-resolution renders on live KATL (wide) data: Old 6.11M, Cubic at source res 5.08M (-17%), 2.75 power 6.12M (0%). On KATL close: Old 4.57M, Cubic 3.65M (-20%), 2.75 power 4.75M (+4%). On KSEA stratiform: Old 270k, Cubic 193k (-28%), 2.75 power 259k (-4%) [S3, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Supports full source resolution (up to $2048^3$) [S3, unofficial]. Bounded memory usage [S3, unofficial]. Efficient empty-space skipping [S3, unofficial].
*   **Limitations:** Trilinear filtering requires an apron, increasing brick size from $8^3$ to $10^3$ [S3, unofficial]. Terrain occlusion requires additional ground height data to prevent rays from passing through terrain when skipping empty pages [S3, unofficial]. Importing large VDB files (>2GB) can cause crashes due to integer overflows if not handled with `uint32` [S2, unofficial]. Chunking large datasets can cause lighting artifacts at chunk borders [S2, unofficial].

## Background
Sparse Page Table and Brick Pool is an experimental feature in Unreal Engine, known as "Heterogeneous Volumes" or "Sparse Volume Textures" (SVT) [S1, S2, unofficial]. It was introduced to handle large general-purpose volumetric data, such as seismic or weather datasets, which are too large for dense voxel grids [S2, unofficial]. The method was notably tested on the Kolumbo dataset, which contains about 2.6 billion non-empty voxels [S2, unofficial]. It is considered the most appropriate way to render large general-purpose data in Unreal Engine, compared to Volumetric Fog (blurry) or Volumetric Cloud (cloud-specific) [S2, unofficial]. The implementation also appears in web-based raymarchers, such as the MRMS volume renderer in the `approach-viz` project, where it enables full source resolution and bounded uploads [S3, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S2] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
[S3] Render the MRMS volume through a sparse page table and brick pool, https://github.com/andyfangdz/approach-viz/pull/74, fetched 2026-10-28 (unofficial)
