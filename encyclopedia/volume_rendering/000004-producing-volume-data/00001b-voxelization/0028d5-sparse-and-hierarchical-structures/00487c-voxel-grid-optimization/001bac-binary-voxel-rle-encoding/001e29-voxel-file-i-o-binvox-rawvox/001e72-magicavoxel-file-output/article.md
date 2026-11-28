# MagicaVoxel File Output
Also called: MagicaVoxel Export, VOX File Export

## Overview
Exports voxel particle data to the MagicaVoxel `.vox` binary format, handling grid sizing, chunk structure, and coordinate transformation. It differs from [binvox](1698-voxel-file-i-o-binvox-rawvox.md) by using a sparse XYZI representation rather than a dense octree, and from [rawvox](1698-voxel-file-i-o-binvox-rawvox.md) by including a structured chunk hierarchy and palette support.

## How it works
**Inputs:** A list of voxel particles (position $p=(x,y,z)$ and color index $c$), resolution $res$, and optional flags for squared padding.
**Outputs:** A binary `.vox` file.

**Data Structures:**
The file consists of a header followed by chunks. Each chunk has an ID (4 bytes), a size (4 bytes), and content.
- **Header:** `'VOX '` (4 bytes), version `150` (4 bytes, little-endian u32).
- **MAIN Chunk:** ID `'MAIN'`, size = sum of child chunk sizes + padding.
- **SIZE Chunk:** ID `'SIZE'`, size `12`, padding `0`, dimensions $size_x, size_y, size_z$ (4 bytes each, little-endian u32).
- **XYZI Chunk:** ID `'XYZI'`, content size $4 + N \times 4$ (where $N$ is voxel count), padding `0`, count $N$ (4 bytes, u32), followed by $N$ tuples of $(vx, vy, vz, index)$ (4 bytes each: i32, i32, i32, u8).

**Steps:**
1. **Calculate Bounding Box:** Initialize $min\_x, min\_y, min\_z$ to $+\infty$ and $max\_x, max\_y, max\_z$ to $-\infty$. Iterate over all particles $p_i$ to update bounds.
2. **Compute Grid Dimensions:**
   $$size\_x = \left\lfloor \frac{max\_x - min\_x}{res} \right\rfloor + 1$$
   $$size\_y = \left\lfloor \frac{max\_y - min\_y}{res} \right\rfloor + 1$$
   $$size\_z = \left\lfloor \frac{max\_z - min\_z}{res} \right\rfloor + 1$$
   Clamp each dimension to a maximum of 256 to respect MagicaVoxel's per-model limit [S1, unofficial].
3. **Write Header:** Write `'VOX '` and version `150` as little-endian u32.
4. **Write MAIN Chunk:**
   - Calculate total size of children: $SIZE$ chunk (24 bytes) + $XYZI$ chunk (16 bytes + $4 \times N$ bytes).
   - Write ID `'MAIN'` and calculated size.
5. **Write SIZE Chunk:**
   - Write ID `'SIZE'`, size `12`, padding `0`.
   - Write $size_x, size_y, size_z$ as little-endian u32.
6. **Write XYZI Chunk:**
   - Calculate voxel coordinates for each particle:
     $$vx = \left\lfloor \frac{p.x - min\_x}{res} + 0.5 \right\rfloor$$
     $$vy = \left\lfloor \frac{p.y - min\_y}{res} + 0.5 \right\rfloor$$
     $$vz = \left\lfloor \frac{p.z - min\_z}{res} + 0.5 \right\rfloor$$
     (Note: Source [S2, unofficial] uses coordinate transformation $(x, z, y)$ for export, implying a swap of Y and Z axes relative to standard Cartesian or source grid [S2, unofficial].)
   - Write ID `'XYZI'`, content size $4 + N \times 4$, padding `0`.
   - Write count $N$.
   - Write each particle's $(vx, vy, vz, index)$ as little-endian integers/byte.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def export_vox(particles, resolution, squared=False):
    if not particles:
        raise ValueError("No particles to export.")

    # 1. Calculate Bounding Box
    min_x = min_y = min_z = float('inf')
    max_x = max_y = max_z = float('-inf')
    for p in particles:
        min_x = min(min_x, p.x)
        max_x = max(max_x, p.x)
        min_y = min(min_y, p.y)
        max_y = max(max_y, p.y)
        min_z = min(min_z, p.z)
        max_z = max(max_z, p.z)

    # 2. Compute Grid Dimensions
    size_x = int((max_x - min_x) / resolution) + 1
    size_y = int((max_y - min_y) / resolution) + 1
    size_z = int((max_z - min_z) / resolution) + 1

    # Clamp to 256
    size_x = min(size_x, 256)
    size_y = min(size_y, 256)
    size_z = min(size_z, 256)

    # 3. Write Header
    write_bytes(b'VOX ')
    write_u32_le(150)

    # 4. Prepare XYZI Data
    xyzis = []
    for p in particles:
        # Coordinate transformation: x, z, y [S2, unofficial]
        vx = int((p.x - min_x) / resolution + 0.5)
        vz = int((p.z - min_z) / resolution + 0.5)
        vy = int((p.y - min_y) / resolution + 0.5)
        xyzis.append((vx, vy, vz, p.index))

    n_voxels = len(xyzis)

    # 5. Write MAIN Chunk
    # SIZE chunk size: 4 (ID) + 4 (size) + 4 (padding) + 12 (dims) = 24
    # XYZI chunk size: 4 (ID) + 4 (content_size) + 4 (padding) + 4 (count) + 4*n_voxels
    xyzi_content_size = 4 + n_voxels * 4
    xyzi_chunk_size = 16 + n_voxels * 4
    main_children_size = 24 + xyzi_chunk_size
    
    write_bytes(b'MAIN')
    write_u32_le(main_children_size)

    # 6. Write SIZE Chunk
    write_bytes(b'SIZE')
    write_u32_le(12)
    write_u32_le(0) # padding
    write_u32_le(size_x)
    write_u32_le(size_y)
    write_u32_le(size_z)

    # 7. Write XYZI Chunk
    write_bytes(b'XYZI')
    write_u32_le(xyzi_content_size)
    write_u32_le(0) # padding
    write_u32_le(n_voxels)
    for vx, vy, vz, idx in xyzis:
        write_i32_le(vx)
        write_i32_le(vy)
        write_i32_le(vz)
        write_u8(idx)
```

**Edge Cases:**
- **Empty Input:** Bail out if no particles exist [S1, unofficial].
- **Grid Size Limit:** If calculated dimensions exceed 256, clamp to 256 [S1, unofficial].
- **Filename:** If no extension is provided, append a random integer and `.vox` [S2, unofficial].

## Subtopics

### VOX Export
Exports the voxel grid to the `.vox` format with specific handling for squared padding and coordinate transformation $(x, z, y)$. If squared, it loops over end dimensions, adding voxels within original bounds or `VOXEL_EMPTY` otherwise. If not squared, it loops over `_numDivs` and adds voxels if `_grid[positionIndex]._value > VOXEL_FREE`, storing `_value - VOXEL_FREE`. It handles filename generation by appending a random integer if no `.vox` extension exists [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of particles, for bounding box calculation and chunk writing.
- **Memory Complexity:** $O(N)$ to store particle data and the XYZI list.
- **Strengths:** Sparse representation is efficient for low-density scenes. Simple chunk structure.
- **Limitations:** Limited to 256x256x256 per model. No hierarchical compression (unlike binvox).
- **When to use:** When exporting to MagicaVoxel for editing or rendering.
- **When not to use:** For very large, dense volumes where hierarchical storage is needed.

## Background
Originated from the MagicaVoxel software by Eric Haines. The `.vox` format is a binary container used by the application. The method described here implements the exporter for this format, often found in voxelization tools like `voxelizer-rs` and `VoxelFragmentML` [S1, S2, unofficial].

## Sources
[S1] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
