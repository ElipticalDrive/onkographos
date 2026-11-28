# World Dimension Constants and Hierarchy


## Overview
World Dimension Constants and Hierarchy defines the fixed spatial partitioning parameters for a voxel-based volume rendering system, establishing the mapping between world coordinates, storage chunks, tiles, bricks, and probe grids. It serves as the structural foundation for memory management and data access, ensuring that spatial queries (such as ray traversal or probe updates) resolve to the correct data blocks. It differs from [Multiresolution and level of detail](7-multiresolution-and-level-of-detail.md) by defining the static global coordinate system and storage layout rather than dynamic resolution selection or compression algorithms.

## How it works
The method establishes a hierarchical coordinate system using fixed integer constants. These constants define the dimensions of the world in various units (voxels, bricks, tiles, chunks) and the spacing of the GI probe grid.

**Inputs and Outputs:**
*   **Input:** None (configuration constants).
*   **Output:** A set of derived values (total counts, voxel dimensions) used by the renderer for coordinate transformations.

**Data Structures and Layout:**
The hierarchy is defined by the following constants. All dimensions are integers.

1.  **World Voxel Dimensions:**
    *   $W_{voxel\_x} = 512$
    *   $W_{voxel\_y} = 256$
    *   $W_{voxel\_z} = 512$
    *   Total Voxels: $512 \times 256 \times 512 = 67,108,864$

2.  **Brick Hierarchy:**
    *   Brick Dimension ($BRICK\_DIM$): 4 voxels per side.
    *   Bricks per Brick ($BRICK\_VOXELS$): $4^3 = 64$.
    *   World Brick Dimensions:
        *   $W_{brick\_x} = 128$
        *   $W_{brick\_y} = 64$
        *   $W_{brick\_z} = 128$
    *   Total Bricks: $128 \times 64 \times 128 = 1,048,576$.
    *   Verification: $1,048,576 \text{ bricks} \times 64 \text{ voxels/brick} = 67,108,864 \text{ voxels}$.

3.  **Tile Hierarchy:**
    *   World Tile Dimensions:
        *   $W_{tile\_x} = 32$
        *   $W_{tile\_y} = 16$
        *   $W_{tile\_z} = 32$
    *   Total Tiles: $32 \times 16 \times 32 = 16,384$.
    *   Bricks per Tile: $32 \times 16 \times 32 = 16,384$ bricks? No, $W_{brick\_x}/W_{tile\_x} = 128/32 = 4$. So $4 \times 4 \times 4 = 64$ bricks per tile.
    *   Verification: $16,384 \text{ tiles} \times 64 \text{ bricks/tile} = 1,048,576 \text{ bricks}$.

4.  **Chunk Hierarchy (Storage):**
    *   World Chunk Dimensions:
        *   $W_{chunk\_x} = 8$
        *   $W_{chunk\_y} = 4$
        *   $W_{chunk\_z} = 8$
    *   Total Chunks: $8 \times 4 \times 8 = 256$.
    *   Bricks per Chunk: $128/8 \times 64/4 \times 128/8 = 16 \times 16 \times 16 = 4,096$ bricks.
    *   Verification: $256 \text{ chunks} \times 4,096 \text{ bricks/chunk} = 1,048,576 \text{ bricks}$.

5.  **L4 Nodes:**
    *   L4 Dimensions:
        *   $W_{L4\_x} = 2$
        *   $W_{L4\_y} = 1$
        *   $W_{L4\_z} = 2$
    *   Total L4 Nodes: $2 \times 1 \times 2 = 4$.
    *   Coverage: Each L4 node covers a $256^3$ voxel region.
    *   Verification: $4 \text{ nodes} \times 256^3 \text{ voxels} = 4 \times 16,777,216 = 67,108,864 \text{ voxels}$.

6.  **Probe Grid:**
    *   Probe Spacing ($PROBE\_SPACING$): 8 voxels.
    *   Probe Grid Dimensions:
        *   $P_{dim\_x} = 64$
        *   $P_{dim\_y} = 32$
        *   $P_{dim\_z} = 64$
    *   Total Probes: $64 \times 32 \times 64 = 131,072$.
    *   World Coverage: $64 \times 8 = 512$ voxels (X), $32 \times 8 = 256$ voxels (Y), $64 \times 8 = 512$ voxels (Z). Matches world voxel dimensions.

7.  **Storage Chunks:**
    *   Storage Chunk Bricks ($STORAGE\_CHUNK\_BRICKS$): 8 bricks.
    *   Storage Chunk Voxels ($STORAGE\_CHUNK\_VOXELS$): 32 voxels.
    *   World Storage Chunk Dimensions:
        *   $W_{store\_cx} = 16$
        *   $W_{store\_cy} = 8$
        *   $W_{store\_cz} = 16$
    *   Total Storage Chunks: $16 \times 8 \times 16 = 2,048$.
    *   Verification: $2,048 \text{ chunks} \times 8 \text{ bricks/chunk} = 16,384 \text{ bricks}$. Note: This storage chunk hierarchy (16,384 bricks) matches the Tile hierarchy count, suggesting storage chunks may map 1:1 to tiles or represent a different packing unit.

**Coordinate Transformation:**
To map a world voxel coordinate $(x, y, z)$ to a brick index:
1.  $b_x = \lfloor x / BRICK\_DIM \rfloor$
2.  $b_y = \lfloor y / BRICK\_DIM \rfloor$
3.  $b_z = \lfloor z / BRICK\_DIM \rfloor$
4.  Brick Index $I = b_x + b_y \times W_{brick\_x} + b_z \times W_{brick\_x} \times W_{brick\_y}$

To map a linear probe index $idx$ to world grid coordinates $(g_x, g_y, g_z)$:
1.  $sx = idx \pmod{P_{dim\_x}}$
2.  $sy = \lfloor idx / P_{dim\_x} \rfloor \pmod{P_{dim\_y}}$
3.  $sz = \lfloor idx / (P_{dim\_x} \times P_{dim\_y}) \rfloor$
4.  $g_x = origin\_g_x + (sx - origin\_g_x) \pmod{P_{dim\_x}}$
5.  $g_z = origin\_g_z + (sz - origin\_g_z) \pmod{P_{dim\_z}}$
6.  $g_y = sy$ (Y is not streamed/wrapped)
7.  World Position $wpos = g \times PROBE\_SPACING$

**Parameters:**
*   $BRICK\_DIM = 4$ [S1, unofficial]
*   $PROBE\_SPACING = 8$ [S1, unofficial]
*   $GI\_UPDATE\_DIV = 8.0$ [S1, unofficial]

**Edge Cases:**
*   **Boundary Wrapping:** Probe coordinates in X and Z are wrapped modulo the probe grid dimensions to handle camera movement relative to the world origin. Y is not wrapped [S1, unofficial].
*   **New Probe Detection:** A probe is considered new if its weight $w < 0.5$ or its stored world position does not match the current calculated grid position [S1, unofficial].

## Subtopics

### [Supercube-based Partial DMV](18962-supercube-based-partial-dmv.md)
Supercube-based partial DMV is a multiresolution volume data representation that encodes nested tetrahedral meshes using supercubes as high-level primitives, exploiting spatial and hierarchical coherence to create sparse partial DMVs for isosurface extraction.

## Operating characteristics
*   **Memory:** The total voxel count is fixed at 67 million. The storage hierarchy uses 2,048 storage chunks.
*   **Access:** Brick indexing is $O(1)$ using integer division and multiplication. Probe coordinate mapping is $O(1)$ with modulo arithmetic.
*   **Performance:** The probe update system updates $1/8$ of the probes per frame, reducing per-frame cost by 8x [S1, unofficial].

## Background
This hierarchy is defined in the `voxelG` project by mstampfli [S1, unofficial]. It is used for managing global illumination probes and voxel data access in a real-time ray-traced renderer.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
