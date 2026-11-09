# MagicaVoxel File Output
Also called: MagicaVoxel Export, VOX File Export

## Overview
Exports voxel particles to the MagicaVoxel `.vox` binary format, handling grid sizing and chunk structure [S1, unofficial]. It differs from generic voxel I/O by enforcing the specific chunk-based binary layout and resolution limits required by the MagicaVoxel application [S1, unofficial].

## How it works
**Inputs:** A list of voxel particles, where each particle has floating-point coordinates $(x, y, z)$ and an integer color index (phase) [S1, unofficial]. A resolution parameter $res$ (voxel size in world units) [S1, unofficial].

**Outputs:** A binary file in `.vox` format [S1, unofficial].

**Data Structures:**
The file consists of a header followed by a `MAIN` chunk [S1, unofficial]. The `MAIN` chunk contains child chunks: `SIZE` and `XYZI` [S1, unofficial].
- **Header:** 4 bytes ID (`'VOX '`), 4 bytes version (`150`) [S1, unofficial].
- **Chunk Header:** 4 bytes ID, 4 bytes size (size of children/payload) [S1, unofficial].
- **SIZE Chunk Payload:** 4 bytes padding, 4 bytes $size_x$, 4 bytes $size_y$, 4 bytes $size_z$ (all little-endian u32) [S1, unofficial].
- **XYZI Chunk Payload:** 4 bytes padding, 4 bytes count (u32), then `count` records of (i32 $x$, i32 $y$, i32 $z$, u8 $index$) [S1, unofficial].

**Steps:**
1. **Calculate Bounding Box:** Initialize $min_x, min_y, min_z$ to $+\infty$ and $max_x, max_y, max_z$ to $-\infty$ [S1, unofficial]. Iterate through all particles to update these bounds [S1, unofficial].
2. **Compute Grid Dimensions:** Calculate the dimensions in voxels:
   $$ size_x = \left\lfloor \frac{max_x - min_x}{res} + 0.5 \right\rfloor + 1 $$
   $$ size_y = \left\lfloor \frac{max_y - min_y}{res} + 0.5 \right\rfloor + 1 $$
   $$ size_z = \left\lfloor \frac{max_z - min_z}{res} + 0.5 \right\rfloor + 1 $$
   Clamp each dimension to a maximum of 256 to respect MagicaVoxel's per-model limit [S1, unofficial].
3. **Write Header:** Write the ASCII string `'VOX '` followed by the version number `150` as a little-endian u32 [S1, unofficial].
4. **Write SIZE Chunk:**
   - Write ID `'SIZE'` (4 bytes).
   - Calculate payload size: 12 bytes (padding + 3 u32s) [S1, unofficial].
   - Write size of children (12 bytes) as little-endian u32 [S1, unofficial].
   - Write padding (4 bytes, value 0) [S1, unofficial].
   - Write $size_x, size_y, size_z$ as little-endian u32 [S1, unofficial].
5. **Write XYZI Chunk:**
   - Calculate voxel coordinates for each particle $p$:
     $$ vx = \left\lfloor \frac{p.x - min_x}{res} + 0.5 \right\rfloor $$
     $$ vy = \left\lfloor \frac{p.y - min_y}{res} + 0.5 \right\rfloor $$
     $$ vz = \left\lfloor \frac{p.z - min_z}{res} + 0.5 \right\rfloor $$
   - Calculate payload size: $4 + 4 \times N$ bytes, where $N$ is the number of particles [S1, unofficial].
   - Write ID `'XYZI'` (4 bytes).
   - Write content size (payload size) as little-endian u32 [S1, unofficial].
   - Write padding (4 bytes, value 0) [S1, unofficial].
   - Write count $N$ as little-endian u32 [S1, unofficial].
   - For each particle, write $vx, vy, vz$ as little-endian i32, and $index$ as u8 [S1, unofficial].
6. **Write MAIN Chunk:**
   - Calculate total size of children: Size of SIZE chunk header+payload (24 bytes) + Size of XYZI chunk header+payload ($12 + 4 + 4N$ bytes) [S1, unofficial].
   - Write ID `'MAIN'` (4 bytes).
   - Write total children size as little-endian u32 [S1, unofficial].
   - Write the previously prepared SIZE and XYZI chunks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def write_vox(particles: list, resolution: float, output_file):
    if not particles:
        raise ValueError("Cannot generate empty VOX file.")

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
    # Using round() equivalent: floor(val + 0.5)
    size_x = int((max_x - min_x) / resolution + 0.5) + 1
    size_y = int((max_y - min_y) / resolution + 0.5) + 1
    size_z = int((max_z - min_z) / resolution + 0.5) + 1

    # Clamp to 256
    size_x = min(size_x, 256)
    size_y = min(size_y, 256)
    size_z = min(size_z, 256)

    # 3. Write Header
    output_file.write(b'VOX ')
    output_file.write(struct.pack('<I', 150))

    # 4. Prepare SIZE Chunk
    size_payload = struct.pack('<I', 0) + struct.pack('<III', size_x, size_y, size_z)
    size_chunk_data = b'SIZE' + struct.pack('<I', 12) + size_payload

    # 5. Prepare XYZI Chunk
    xyzi_particles = []
    for p in particles:
        vx = int((p.x - min_x) / resolution + 0.5)
        vy = int((p.y - min_y) / resolution + 0.5)
        vz = int((p.z - min_z) / resolution + 0.5)
        xyzi_particles.append((vx, vy, vz, p.index))
    
    count = len(xyzi_particles)
    
    # Payload: 4 bytes padding + 4 bytes count + count * 13 bytes (3 i32 + 1 u8)
    # Note: Source notes state content size as 4 + N*4, but the layout specifies 3 i32 + 1 u8 per particle (13 bytes).
    # We follow the layout for data packing to ensure correct binary output.
    xyzi_payload = struct.pack('<I', 0) + struct.pack('<I', count)
    for vx, vy, vz, idx in xyzi_particles:
        xyzi_payload += struct.pack('<iiiB', vx, vy, vz, idx)
        
    # Use the layout-derived size for correctness
    xyzi_chunk_data = b'XYZI' + struct.pack('<I', 4 + 4 + 13 * count) + xyzi_payload

    # 6. Write MAIN Chunk
    # Children: SIZE chunk (24 bytes) + XYZI chunk (16 + 13*N bytes)
    main_children_size = 24 + (16 + 13 * count)
    main_chunk_data = b'MAIN' + struct.pack('<I', main_children_size) + size_chunk_data + xyzi_chunk_data

    output_file.write(main_chunk_data)
```

**Parameters:**
- `resolution`: Voxel size in world units. Typical values depend on scene scale [S1, unofficial].

**Edge Cases:**
- Empty particle list: Bail out with error [S1, unofficial].
- Grid size > 256: Clamp dimensions to 256 [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for bounding box calculation and coordinate conversion, where $N$ is the number of particles [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store particle data and chunk payloads [S1, unofficial].
- **Strengths:** Standard format for MagicaVoxel; efficient binary storage [S1, unofficial].
- **Limitations:** Limited to 256x256x256 per model; no metadata or animation support in basic XYZI export [S1, unofficial].

## Background
Originated from the MagicaVoxel application by Eric Wette [G]. The `.vox` format is a binary container used by the software for storing voxel models [S1, unofficial]. This method implements the exporter side of that format [S1, unofficial].

## Sources
[S1] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
