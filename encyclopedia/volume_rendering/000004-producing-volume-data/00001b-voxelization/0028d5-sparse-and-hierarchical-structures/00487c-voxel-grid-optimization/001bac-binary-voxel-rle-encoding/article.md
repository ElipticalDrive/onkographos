# Binary Voxel RLE Encoding

Also called: Run-Length Encoding Output Compression, binvox-like RLE, run-length encoded voxel writer

## Overview
Binary Voxel RLE Encoding compresses a 3D voxel grid into a binary stream using run-length encoding (RLE) to store occupancy data efficiently [S1, unofficial][S2, unofficial]. It differs from sibling methods like [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md) by storing explicit occupancy bits for every possible voxel, whereas this method stores only runs of identical values, making it suitable for sparse data or when minimal bounding box optimization is applied [S2, unofficial][S1, unofficial]. It differs from [RLE4 Compression](4441-rle4-compression.md) by storing only binary occupancy (0 or 1) rather than per-voxel material or intensity information [S1, unofficial].

## How it works
**Inputs:**
- A 3D voxel grid defined by dimensions `m_dimX`, `m_dimY`, `m_dimZ` [S1, unofficial].
- Voxel grid spacing `m_spacing` [S1, unofficial].
- A function `isInside(x, y, z)` that returns the occupancy status (0 for empty, 1 for filled) of a voxel at coordinates `(x, y, z)` [S1, unofficial].

**Outputs:**
- A binary file containing an ASCII header followed by RLE-encoded data [S1, unofficial].

**Data Structures:**
- The output stream consists of:
    1. ASCII header strings: `"scale "` followed by the floating-point `m_spacing`, then the string `"data"` [S1, unofficial].
    2. A sequence of byte pairs representing runs [S2, unofficial].
    3. Each pair consists of:
        - `value`: 1 byte, representing the occupancy (0 or 1) [S2, unofficial].
        - `count`: 2 bytes (little-endian or big-endian, not specified in sources, but typically little-endian in such formats), representing the run length [S2, unofficial]. The stored value is `count - 1`, allowing a range of [0, 255] to represent run lengths of [1, 256] [S2, unofficial].

**Algorithm Steps:**
1. Write the ASCII header: `"scale "` followed by `m_spacing` as a float, then `"data"` [S1, unofficial].
2. Initialize `currentvalue = char(isInside(0, 0, 0))` and `current_seen = 1` [S1, unofficial].
3. Write `currentvalue` to the output stream [S1, unofficial].
4. Loop `x` from 0 to `m_dimX - 1`, `z` from 0 to `m_dimY - 1`, and `y` from 0 to `m_dimZ - 1` [S1, unofficial]. Note: The source specifies the loop order as x, z, y [S1, unofficial].
5. If `x == 0 && y == 0 && z == 0`, skip the iteration to avoid double-writing the first voxel [S1, unofficial].
6. Compute `nextvalue = char(isInside(x, y, z))` [S1, unofficial].
7. If `nextvalue != currentvalue` or `current_seen == 255`:
    a. Write `current_seen` to the output stream [S1, unofficial].
    b. Reset `current_seen = 1` [S1, unofficial].
    c. Update `currentvalue = nextvalue` [S1, unofficial].
    d. Write `currentvalue` to the output stream [S1, unofficial].
8. Else:
    a. Increment `current_seen` [S1, unofficial].
9. After the loops, write the final `current_seen` count to the output stream [S1, unofficial].
10. Close the file [S1, unofficial].

**Decoding:**
- To retrieve the original voxel coordinate `(x, y, z)` from the compressed output, add the lower bound of the minimal bounding box in voxelized space: `(x + x_vox_lb, y + y_vox_lb, z + z_vox_lb)` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_voxel_rle_encode(is_inside_func, dim_x, dim_y, dim_z, spacing, output_file):
    """
    Encodes a 3D voxel grid into a binary RLE format.
    
    Args:
        is_inside_func: A callable that takes (x, y, z) and returns 0 or 1.
        dim_x, dim_y, dim_z: Dimensions of the voxel grid.
        spacing: Voxel grid spacing.
        output_file: A file-like object opened for writing in binary mode.
    """
    # Write ASCII header
    output_file.write(b"scale ")
    output_file.write(struct.pack('f', spacing))
    output_file.write(b"data")
    
    # Initialize first voxel
    current_value = 1 if is_inside_func(0, 0, 0) else 0
    current_seen = 1
    
    # Write first value
    output_file.write(bytes([current_value]))
    
    # Loop through voxels
    for x in range(dim_x):
        for z in range(dim_y):
            for y in range(dim_z):
                # Skip the first voxel as it was already written
                if x == 0 and y == 0 and z == 0:
                    continue
                
                next_value = 1 if is_inside_func(x, y, z) else 0
                
                if next_value != current_value or current_seen == 255:
                    # Write count (run length - 1)
                    output_file.write(bytes([current_seen - 1]))
                    # Write new value
                    output_file.write(bytes([next_value]))
                    current_value = next_value
                    current_seen = 1
                else:
                    current_seen += 1
    
    # Write final count
    output_file.write(bytes([current_seen - 1]))
```

**Parameters:**
- `m_spacing`: Voxel grid spacing, written as a floating-point value immediately after the "scale " header string [S1, unofficial].
- `m_dimX`, `m_dimY`, `m_dimZ`: Grid dimensions in each axis, used as loop bounds [S1, unofficial].

**Edge Cases and Pitfalls:**
- Maximum run length is capped at 255 (char type); when reached, the encoder forces a new run to prevent overflow [S1, unofficial].
- The first voxel (0,0,0) is written before the loop to initialize the run, then skipped inside the loop to avoid double-writing [S1, unofficial].
- The count is stored as `count - 1`, so a stored value of 0 represents a run length of 1 [S2, unofficial].

## Subtopics

### [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md)
A pre-processing optimization for volume rendering that compresses voxel runs via run-length encoding based on pre-classified opacities to skip transparent or opaque regions during rendering. It differs from single-axis methods by constructing a separate encoded volume for each of the three major viewing directions, allowing the renderer to bypass empty space efficiently along the viewing axis. [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md)

### [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md)
Exports voxelized geometry data to standard file formats for interchange with other graphics applications. It implements two primary output modes: the `binvox` format, which uses a compact binary run-length encoding for occupancy data, and the `rawvox` format, which provides a simple binary dump of grid dimensions and metadata. Unlike its sibling [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which focuses on the internal compression algorithm, this method handles the specific file headers, coordinate transformations, and I/O protocols required by external tools. [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md)

### [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md)
Voxel Indexing and Bit Compression maps 3D voxel coordinates to a 1D binary bitstream and packs these bits into fixed-width integers (typically 64-bit) to minimize memory footprint and enable efficient parallel processing. It differs from [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) by storing explicit occupancy bits for every possible voxel in the bounding box rather than run-length encoding sparse data, making it suitable for dense grids or scenarios requiring direct random access to occupancy status via atomic operations. [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md)

### [Dense-grid to Voxlap-slab packing](2155-dense-grid-to-voxlap-slab-packing.md)
Converts a dense voxel mask and color grid of size VSID × VSID × MAXZDIM into the Voxlap slab-RLE column format. It seeds an all-solid VXL and carves air voxels using a column function to preserve buried-voxel-aware encoding. [Dense-grid to Voxlap-slab packing](2155-dense-grid-to-voxlap-slab-packing.md)

### [RLE4 Compression](4441-rle4-compression.md)
RLE4 Compression converts a dense 3D voxel grid into a sparse, column-oriented structure called **Map4** to solve memory bloat in large volumes with significant empty space. It processes the volume slice-by-slice (XZ planes), encoding runs of empty space (skip) and occupied space (solid) along the Y-axis into 16-bit words, while storing per-voxel color and intensity data in a separate texture buffer. Unlike [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which stores binary occupancy, RLE4 preserves per-voxel material and intensity information for splatting. It differs from [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md) by using a fixed 16-bit RLE format with specific bit-widths for solid (6 bits) and skip (10 bits) runs, and by including a perturbation step for surface normals. [RLE4 Compression](4441-rle4-compression.md)

### RLE Export
Compresses the voxel grid using Run-Length Encoding and saves it to a binary file [S3, unofficial].
**Steps:**
1. Initialize `value` to `std::numeric_limits<uint16_t>::max()`, `repetitions` to 0, and `idx` to 0 [S3, unofficial].
2. While `idx < size`:
    a. While `idx < size` and `_grid[idx]._value == value`, increment `repetitions` and `idx` [S3, unofficial].
    b. If `repetitions > 0`, push `{value, repetitions}` to `rleData` [S3, unofficial].
    c. Set `value = _grid[idx]._value` and `repetitions = 0` [S3, unofficial].
3. Write `_numDivs` (`glm::uvec3`) to file [S3, unofficial].
4. Iterate `rleData`, writing `value` (`uint16_t`) and `repetitions` (`uint32_t`) for each entry [S3, unofficial].
**Layout:**
- `RLEData` struct: `uint16_t value`, `uint32_t repetitions` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** O(n), where n is the number of voxels in the grid [S2, unofficial].
- **Memory Complexity:** Proportional to the number of runs, which is typically much less than n for sparse or smooth volumes.
- **Performance:** Runs much faster than basic flood fill search [S2, unofficial].
- **Scaling:** The compression ratio depends on the sparsity and continuity of the voxel data. Dense, noisy data will have poor compression ratios.
- **Strengths:** Simple to implement, efficient for sparse data, reduces file size significantly for typical voxelized geometry.
- **Limitations:** Limited to binary occupancy (unless extended), run length capped at 256, requires sequential access for decoding (unless index is maintained).
- **When to use:** When storing or transmitting voxelized geometry data, especially when the data is sparse or has large uniform regions.
- **When not to use:** When random access to individual voxels is required without decoding the entire stream, or when per-voxel attributes (color, normal) need to be stored in the same RLE structure (use [RLE4 Compression](4441-rle4-compression.md) instead).

## Background
Binary Voxel RLE Encoding is a compression scheme for the output voxel data, storing only voxels within the minimal bounding box of the mesh using run-length encoding [S2, unofficial]. It is commonly used in voxelization tools to save results to disk [S1, unofficial]. The format is similar to the `binvox` format, which is a standard for storing voxelized 3D models [S2, unofficial][S1, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
