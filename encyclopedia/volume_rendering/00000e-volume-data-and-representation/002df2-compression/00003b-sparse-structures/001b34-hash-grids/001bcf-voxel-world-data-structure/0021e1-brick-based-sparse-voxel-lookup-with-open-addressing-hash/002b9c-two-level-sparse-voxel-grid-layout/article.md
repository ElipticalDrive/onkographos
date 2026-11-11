# Two-Level Sparse Voxel Grid Layout

Also called: BrickMap, BrickPool, Brickmaps / two-level grids, brickgrid, two-level grids

## Overview
Two-Level Sparse Voxel Grid Layout stores a sparse voxel world using an outer grid of cells pointing to fixed-size inner bricks of packed voxels [S1, unofficial]. It differs from its parent [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md) by using a uniform grid for the top level rather than a hash table, trading the hash collision resolution overhead for simpler, direct indexing at the cost of potential empty top-level cells [S2].

## How it works
**Inputs and Outputs**
- Input: World dimensions in voxels (`WORLDSIZE`), brick dimensions (`BRICKSIZE`), and voxel data (occupancy, material, color).
- Output: A memory layout consisting of a top-level grid of pointers and a pool of brick structures.

**Data Structures**
1.  **Top-Level Grid**: A 3D array of size $GRIDSIZE^3$. Each cell stores a pointer (or index) to a brick, or `null` if the region is empty [S1, unofficial].
    -   $GRIDSIZE = \lceil WORLDSIZE / BRICKSIZE \rceil$ [S1, unofficial].
    -   Default `WORLDSIZE` is 512, default `GRIDSIZE` is 64 [S1, unofficial].
2.  **Brick (BrickMap)**: A fixed-size structure representing an $8 \times 8 \times 8$ block of voxels [S2].
    -   **Occupancy Mask**: A 512-bit bitset indicating which of the 512 voxels are active [S2].
    -   **Voxel Data**: Packed `uint32_t` values for active voxels. The exact layout of these packed values is not specified in the sources, but they are stored contiguously following the mask [S1, unofficial].
    -   **Size**: The occupancy mask alone is "exactly 71 bytes, not counting shading attributes" [S2].
    -   **Color**: Stored separately, DXT1-compressed to ~4 bits per voxel. Each brick also stores an LOD color [S2].

**Algorithm Steps**
1.  **Initialization**:
    -   Calculate `GRIDSIZE` from `WORLDSIZE` and `BRICKSIZE` [S1, unofficial].
    -   Allocate the top-level grid of size $GRIDSIZE^3$ initialized to `null` [S1, unofficial].
    -   Initialize an empty dynamic brick pool [S1, unofficial].
2.  **Insertion/Update**:
    -   Given a voxel coordinate $(x, y, z)$, calculate the brick coordinate:
        $$ b_x = \lfloor x / BRICKSIZE \rfloor, \quad b_y = \lfloor y / BRICKSIZE \rfloor, \quad b_z = \lfloor z / BRICKSIZE \rfloor $$
    -   Access the top-level grid cell at $(b_x, b_y, b_z)$ [S1, unofficial].
    -   If the cell is `null`:
        -   Allocate a new brick from the dynamic pool [S1, unofficial].
        -   Set the cell pointer to the new brick.
        -   Safety clamp the pool growth to $GRIDSIZE^3$ to prevent exceeding the world bounds [S1, unofficial].
    -   If the cell points to an existing brick:
        -   Retrieve the brick.
    -   Set the corresponding bit in the brick's 512-bit occupancy mask for the local voxel index:
        $$ local\_index = (x \pmod{BRICKSIZE}) + BRICKSIZE \cdot ((y \pmod{BRICKSIZE}) + BRICKSIZE \cdot (z \pmod{BRICKSIZE})) $$
        $$ brick.occupancy\_mask[local\_index] = 1 $$
    -   Store the voxel data (material, color) in the brick's packed voxel array at the position corresponding to `local_index` among active voxels [S1, unofficial].
3.  **Access**:
    -   To read voxel $(x, y, z)$, compute brick coordinates $(b_x, b_y, b_z)$ [S1, unofficial].
    -   Check if `top_level_grid[b_x][b_y][b_z]` is `null`. If so, the voxel is empty [S1, unofficial].
    -   Otherwise, retrieve the brick and check `brick.occupancy_mask[local_index]` [S1, unofficial].
    -   If set, retrieve the voxel data from the brick [S1, unofficial].

**Parameters**
-   `WORLDSIZE`: World dimension in voxels. Default: 512 [S1, unofficial].
-   `BRICKSIZE`: Brick dimension. Default: 8 (implied by $8 \times 8 \times 8$ bricks) [S2].
-   `GRIDSIZE`: Outer grid dimension. Calculated as $WORLDSIZE / BRICKSIZE$. Default: 64 [S1, unofficial].

**Edge Cases and Pitfalls**
-   **Top-Level Memory Cost**: On planet-scale worlds, the dense top-level grid becomes the largest memory cost (e.g., 64 MB static grid vs ~50 MB bricks/color) [S2]. The top level must be sparse for large worlds [S2].
-   **Precision**: The sources do not specify floating-point precision requirements.
-   **Boundaries**: The dynamic brick pool is safety-clamped to $GRIDSIZE^3$ [S1, unofficial].

## Subtopics

### [Brick with Sparse Top-Level](6925-brick-with-sparse-top-level.md)
This method stores planet-scale voxel data using sparse $8^3$ (512 voxel) bricks organized under a sparse top-level index, combining an occupancy mask with palette-compressed material IDs to minimize memory usage. It differs from dense grid approaches by trading trivial random access for excellent streaming properties and significantly lower memory footprint, while avoiding the split/relink complexity of full sparse trees.

### Debris and Fluid Grid Handling
Debris and Fluids maintain their own small grids and convert to and from the main brick format [S3, unofficial]. There is no heightfield or mesh fallback [S3, unofficial].

## Operating characteristics
-   **Time Complexity**: Lookup is $O(1)$ for the top-level grid access, assuming constant-time pointer dereference. Brick access is $O(1)$ given the fixed size.
-   **Memory Complexity**: $O(GRIDSIZE^3)$ for the top-level grid plus $O(N)$ for active bricks, where $N$ is the number of active bricks. The top-level grid is dense, leading to high memory usage for large worlds if not sparse [S2].
-   **Measured Performance**: 40–164 MRays/s on a Radeon HD 7970 [S2]. This beat the ESVO comparison (43.8 and 22.5 MRays/s) on the same scenes [S2].
-   **Strengths**: Data stays in raw form, making it directly editable [S2]. Supports streaming from disk and basic LOD [S2]. Updates are fast [S2].
-   **Limitations**: The dense top-level grid does not scale to planet-scale worlds without becoming the dominant memory cost [S2].
-   **When to Use**: Suitable for worlds where the top-level sparsity is low or the world size is small enough that the top-level grid fits in memory [S2].
-   **When Not to Use**: Planet-scale worlds where the top-level grid would consume excessive memory (e.g., >64 MB) [S2].

## Background
The method is described in van Wingerden's thesis (2015) [S2]. It uses a uniform "brickgrid" of pointers to $8 \times 8 \times 8$ "brickmaps" [S2]. The thesis supports rendering "potentially infinite landscapes while using a finite amount of memory" [S2]. The dense top-level grid was identified as a bottleneck, with 64 MB of static grid against at most ~50 MB of bricks and color [S2]. The "Hybrid Voxel Formats" study (Arbore et al. 2024) notes that raw grids that are too fine can be detrimental on sparse models [S2].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
[S3] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
