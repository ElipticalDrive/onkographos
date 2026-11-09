# Brick with Sparse Top-Level

Also called: 64-tree top level, 8³ Brick with Sparse Top-Level, Brick-based Sparse Voxel Lookup with Open-Addressing Hash > Two-Level Sparse Voxel Grid Layout > Brick with Sparse Top-Level, Brick-based voxel storage, Brickmap, Sparse 8³ bricks, Sparse Voxel Brick Storage with Palette Compression

## Overview
This method stores planet-scale voxel data using sparse $8^3$ (512 voxel) bricks organized under a sparse top-level index, combining an occupancy mask with palette-compressed material IDs to minimize memory usage [S1, unofficial][S2]. It differs from dense grid approaches by trading trivial random access for excellent streaming properties and significantly lower memory footprint, while avoiding the split/relink complexity of full sparse trees [S2].

## How it works
### Inputs and Outputs
- **Input**: Planet-frame integer coordinates $(X, Y, Z)$; Material definitions; Procedural generation seed.
- **Output**: A brick structure containing material data for a $8 \times 8 \times 8$ region, or a flag indicating uniformity.

### Data Structures

#### Brick Layout
Each brick represents a $8 \times 8 \times 8$ volume. The physical size is 0.8 meters, with a voxel resolution of 0.1 meters [S1, unofficial].

1.  **Uniform Flag**: A boolean in the parent node indicating if the brick is all-air or all-one-material. If true, the brick data is 0 bytes [S1, unofficial][S2].
2.  **Occupancy Mask**: 512 bits (64 bytes). A bit is set if the corresponding voxel is solid [S1, unofficial][S2].
3.  **Material Palette**: A list of unique material indices present in the brick.
4.  **Compressed Material IDs**: Variable bit-depth indices (1, 2, 4, or 8 bits per voxel) referencing the palette [S1, unofficial][S2].

#### Top-Level Index
A sparse structure mapping brick coordinates to brick data. Options include:
-   A sparse hash map keyed by brick coordinate (CPU-side) [S1, unofficial].
-   A shallow 64-tree (GPU-side for ray traversal) [S1, unofficial][S2].

### Numbered Steps

1.  **Coordinate to Brick Mapping**:
    Given a voxel coordinate $V$, compute the brick coordinate $B = \lfloor V / 8 \rfloor$ [S2].

2.  **Brick Lookup**:
    Query the top-level index for $B$.
    -   If $B$ is not present, the brick is empty (air) [S2].
    -   If $B$ is present, retrieve the brick data [S2].

3.  **Uniform Check**:
    If the brick is marked as uniform:
    -   If "all-air", return air.
    -   If "all-material", return the single material ID [S1, unofficial][S2].

4.  **Voxel Extraction**:
    If the brick is not uniform:
    a.  Check the occupancy mask bit at local index $L = V \pmod 8$. If 0, return air [S1, unofficial][S2].
    b.  If 1, extract the material ID from the compressed material array using the local index $L$ and the brick's bit-depth [S1, unofficial][S2].

5.  **Procedural Appearance**:
    Compute color and roughness procedurally from the voxel's position, as per-voxel color is not stored [S1, unofficial].

### Formulas

**Per-Brick Raw Cost (Teardown-style, 1 byte/voxel):**
$$ C_{raw} = 512 \text{ B} + 8 \text{ B (header)} \approx 520 \text{ B} $$
[S2]

**Per-Brick Compact Cost (Palette-compressed):**
$$ C_{compact} = 64 \text{ B (mask)} + 128 \text{ B (2-bit avg indices)} + 18 \text{ B (palette/header)} \approx 210 \text{ B} $$
[S2]

**Cost per Surface Voxel:**
Assuming the brick count crossing a surface is $\frac{\text{Surface Voxels}}{64}$:
$$ \text{Cost}_{surface, raw} \approx \frac{520}{64} \approx 8 \text{ B} $$
$$ \text{Cost}_{surface, compact} \approx \frac{210}{64} \approx 3.3 \text{ B} $$
[S2]

### Parameters
-   **Brick Size**: $8^3$ voxels [S1, unofficial][S2].
-   **Physical Size**: 0.8 meters [S1, unofficial].
-   **Voxel Resolution**: 0.1 meters (10 cm) [S1, unofficial].
-   **Material Index Bits**: 1, 2, 4, or 8 bits per voxel [S1, unofficial][S2].
-   **GPU Budget**: ~2 GB of voxel data on RTX 4090 [S1, unofficial].

### Edge Cases and Pitfalls
-   **Uniform Bricks**: All-air or all-one-material bricks cost 0 bytes for data, marked by a parent flag [S1, unofficial][S2].
-   **Color**: Per-voxel color is not stored; variation is procedural [S1, unofficial].
-   **Downsampling**: A coarse voxel is solid if at least 4 of its 8 children are solid, and takes the most common solid child material [S1, unofficial].
-   **Edits**: Edited bricks are persisted as full-resolution snapshots; unedited bricks are regenerable cache [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lookup_voxel(brick_map, voxel_coord):
    """
    Lookup material for a voxel in a planet-scale world.
    brick_map: dict mapping (bx, by, bz) -> BrickData
    voxel_coord: tuple (x, y, z)
    """
    # 1. Compute brick coordinate
    bx = voxel_coord[0] // 8
    by = voxel_coord[1] // 8
    bz = voxel_coord[2] // 8
    brick_key = (bx, by, bz)

    # 2. Lookup brick
    if brick_key not in brick_map:
        return AIR_MATERIAL

    brick = brick_map[brick_key]

    # 3. Check uniform flag
    if brick.is_uniform:
        if brick.uniform_type == "EMPTY":
            return AIR_MATERIAL
        elif brick.uniform_type == "SOLID":
            return brick.uniform_material

    # 4. Check occupancy mask
    # Local index within the 8x8x8 brick
    lx = voxel_coord[0] % 8
    ly = voxel_coord[1] % 8
    lz = voxel_coord[2] % 8
    local_index = lx + 8 * ly + 64 * lz

    if not (brick.occupancy_mask >> local_index) & 1:
        return AIR_MATERIAL

    # 5. Extract material ID
    # Assuming bit_depth is known for the brick
    bit_depth = brick.bit_depth
    # Extract 'bit_depth' bits starting at local_index
    # Note: Implementation details for bit-packing extraction depend on storage format
    # This is a placeholder for the actual bit-extraction logic
    material_id = extract_bits(brick.material_data, local_index, bit_depth)
    
    return brick.palette[material_id]

def extract_bits(data, start_index, bit_depth):
    """
    Extract bit_depth bits from data starting at start_index.
    data: bytes or bit array
    start_index: bit index
    bit_depth: number of bits to extract
    """
    # [G] Placeholder for actual bit-packing extraction logic
    # In a real implementation, this would handle byte alignment and masking
    pass
```

## Subtopics

### [Two-Level Sparse Voxel Grid Layout](5065-two-level-sparse-voxel-grid-layout.md)
The parent structure that organizes bricks. Use this when you need a hierarchical organization of sparse voxel data.

### [Sparse 64-tree](5066-sparse-64-tree.md)
An alternative top-level structure. Use this when you need very low memory overhead for geometry but can tolerate more complex edit operations.

## Operating characteristics
-   **Memory**: Compact cost is ~0.66 GB per km² at 10 cm resolution for 2x surface voxels, compared to 1.6 GB raw and 10 GB for dense grids [S2].
-   **Edit Cost**: Trivial; rewrite affected bricks and mark dirty [S2].
-   **Ray Traversal**: Good; two-level DDA plus bitmask. Performance ~40–164 MRays/s on a 2012 GPU [S2].
-   **LOD**: Easy; mip bricks to 2x voxel size, LOD color per brick [S2].
-   **Streaming**: Excellent; a brick is a natural network and disk unit [S2].
-   **Scaling**: Memory scales linearly with the number of surface voxels [S2].

## Background
This structure is recommended for planet-scale voxel worlds, matching Gustafsson's next engine (2024), van Wingerden's brickmaps, Arbore et al.'s "R(N³) G(M)" finding, and Aokana's many-shallow-structures result [S2]. It was proposed as a solution to the high memory costs of dense grids and the complexity of full sparse trees [S2].

## Sources
[S1] Choose voxel data structure & LOD scheme #9, https://github.com/theHimanshuShekhar/bhayanakvoxel/issues/9, fetched 2026-10-28 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
