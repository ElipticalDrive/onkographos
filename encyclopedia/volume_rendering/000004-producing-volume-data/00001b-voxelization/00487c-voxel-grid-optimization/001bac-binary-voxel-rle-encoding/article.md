# Binary Voxel RLE Encoding

Also called: Run-Length Encoding Output Compression, binvox-like RLE, run-length encoded voxel writer

## Overview
Binary Voxel RLE Encoding compresses a 3D voxel grid into a binary stream using run-length encoding (RLE) to store occupancy data efficiently [S1, unofficial]. It differs from sibling methods like [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md) by storing only runs of identical values rather than explicit bits for every possible voxel, making it suitable for sparse or structured occupancy data [S2, unofficial]. Unlike [RLE4 Compression](4441-rle4-compression.md) which preserves per-voxel material and intensity information, this method stores only binary occupancy (0 or 1) [S1, unofficial].

## How it works
**Inputs:**
- A 3D voxel grid of dimensions $m\_dimX \times m\_dimY \times m\_dimZ$ [S1, unofficial].
- A voxel spacing value $m\_spacing$ [S1, unofficial].
- A function $isInside(x, y, z)$ that returns 1 if the voxel at $(x, y, z)$ is occupied (inside the mesh) and 0 otherwise [S1, unofficial].
- The lower bounds of the minimal bounding box in voxelized space: $(x_{vox\_lb}, y_{vox\_lb}, z_{vox\_lb})$ [S2, unofficial].

**Outputs:**
- A binary file containing an ASCII header followed by a compressed binary stream [S1, unofficial].

**Data Structures:**
- The output stream consists of:
    1. ASCII header: String "scale " followed by the floating-point value of $m\_spacing$, followed by the string "data" [S1, unofficial].
    2. Binary payload: A sequence of bytes representing runs. Each run is encoded as two bytes:
        - Byte 1: `value` (0 or 1) [S2, unofficial].
        - Byte 2: `count` (unsigned char, range 0–255) [S2, unofficial]. The actual run length is $count + 1$, allowing for runs of length 1 to 256 [S2, unofficial].

**Algorithm Steps:**
1. Write the ASCII header: "scale " + $m\_spacing$ + "data" [S1, unofficial].
2. Initialize the encoder state:
    - Set `currentvalue` = $isInside(0, 0, 0)$ [S1, unofficial].
    - Set `current_seen` = 1 [S1, unofficial].
    - Write `currentvalue` as a single byte to the output stream [S1, unofficial].
3. Iterate through the voxel grid in the order: $x$ from 0 to $m\_dimX-1$, $z$ from 0 to $m\_dimY-1$, $y$ from 0 to $m\_dimZ-1$ [S1, unofficial].
4. For each voxel $(x, y, z)$:
    - If $x=0, y=0, z=0$, skip to the next iteration (the first voxel was already written in step 2) [S1, unofficial].
    - Compute `nextvalue` = $isInside(x, y, z)$ [S1, unofficial].
    - If `nextvalue` $\neq$ `currentvalue` OR `current_seen` == 255:
        - Write `current_seen` as a byte to the output stream [S1, unofficial].
        - Reset `current_seen` = 1 [S1, unofficial].
        - Update `currentvalue` = `nextvalue` [S1, unofficial].
        - Write `currentvalue` as a byte to the output stream [S1, unofficial].
    - Else:
        - Increment `current_seen` by 1 [S1, unofficial].
5. After the loops complete, write the final `current_seen` value as a byte to the output stream [S1, unofficial].
6. Close the file [S1, unofficial].

**Decoding:**
- To retrieve the original voxel coordinate $(x_{orig}, y_{orig}, z_{orig})$ from the compressed output, add the lower bound of the minimal bounding box:
  $$x_{orig} = x + x_{vox\_lb}$$
  $$y_{orig} = y + y_{vox\_lb}$$
  $$z_{orig} = z + z_{vox\_lb}$$
  where $(x, y, z)$ is the coordinate derived from the run-length decoding process [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_binary_voxel_rle(grid_dims, spacing, is_inside_func, bounds):
    """
    Encodes a 3D voxel grid into Binary Voxel RLE format.
    
    Args:
        grid_dims (tuple): (m_dimX, m_dimY, m_dimZ)
        spacing (float): m_spacing
        is_inside_func (callable): Function taking (x, y, z) returning 0 or 1
        bounds (tuple): (x_vox_lb, y_vox_lb, z_vox_lb)
        
    Returns:
        bytes: The encoded binary data (header + payload)
    """
    m_dimX, m_dimY, m_dimZ = grid_dims
    x_vox_lb, y_vox_lb, z_vox_lb = bounds
    
    output = bytearray()
    
    # 1. Write ASCII header
    header = f"scale {spacing}data".encode('ascii')
    output.extend(header)
    
    # 2. Initialize state with first voxel (0,0,0)
    current_value = is_inside_func(0, 0, 0)
    current_seen = 1
    
    # Write initial value byte
    output.append(current_value)
    
    # 3. Iterate through grid
    # Order: x, then z, then y
    for x in range(m_dimX):
        for z in range(m_dimY):
            for y in range(m_dimZ):
                # 4. Skip first voxel as it was already written
                if x == 0 and y == 0 and z == 0:
                    continue
                
                next_value = is_inside_func(x, y, z)
                
                # Check for run change or max run length
                if next_value != current_value or current_seen == 255:
                    # Write the count for the previous run
                    output.append(current_seen)
                    # Start new run
                    current_seen = 1
                    current_value = next_value
                    # Write the new value
                    output.append(current_value)
                else:
                    # Continue current run
                    current_seen += 1
    
    # 5. Write final run count
    output.append(current_seen)
    
    return bytes(output)
```

**Parameters:**
- `m_spacing`: Voxel grid spacing, written as a floating-point value [S1, unofficial].
- `m_dimX`, `m_dimY`, `m_dimZ`: Grid dimensions in each axis [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Max Run Length:** The maximum run length is capped at 255 (stored as 255, representing 256 voxels). When this limit is reached, the encoder forces a new run to prevent overflow [S1, unofficial].
- **First Voxel Handling:** The first voxel $(0,0,0)$ is written before the loop to initialize the run. It must be skipped inside the loop to avoid double-writing [S1, unofficial].
- **Bounding Box:** The compression scheme assumes the output only includes voxels within the minimal bounding box of the mesh. Coordinates must be adjusted by the lower bounds $(x_{vox\_lb}, y_{vox\_lb}, z_{vox\_lb})$ when mapping back to original space [S2, unofficial].

## Subtopics

### [Run-length encoded voxel runs](1209-run-length-encoded-voxel-runs.md)
A pre-processing optimization that constructs separate encoded volumes for each of the three major viewing directions to skip transparent or opaque regions during rendering.

### [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md)
Handles file headers, coordinate transformations, and I/O protocols for standard formats like `binvox`, which uses this compact binary RLE format.

### [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md)
Maps 3D coordinates to a 1D bitstream packed into integers, suitable for dense grids requiring direct random access.

### [Dense-grid to Voxlap-slab packing](2155-dense-grid-to-voxlap-slab-packing.md)
Converts dense grids into a slab-RLE column format with explicit column offsets and color attachments.

### [RLE4 Compression](4441-rle4-compression.md)
Uses a fixed 16-bit RLE format with specific bit-widths for solid and skip runs, preserving per-voxel material and intensity data.

## Operating characteristics
- **Time Complexity:** The encoding process is $O(n)$, where $n$ is the number of voxels in the grid [S2, unofficial].
- **Memory Complexity:** The output size depends on the compressibility of the occupancy data. In the worst case (alternating 0s and 1s), the size is $2 \times n$ bytes (plus header). In the best case (all solid or all empty), the size is minimal (2 bytes per run).
- **Strengths:** Simple implementation, efficient for sparse or large uniform regions, compact storage for binary occupancy [S2, unofficial].
- **Limitations:** Does not store per-voxel attributes (color, normal) [S1, unofficial]. Run length is limited to 256 voxels per run [S2, unofficial]. Requires knowledge of the bounding box for coordinate reconstruction [S2, unofficial].
- **When to use:** When storing binary occupancy data for interchange or compression, especially when memory bandwidth is a concern and per-voxel attributes are not needed [S2, unofficial].
- **When not to use:** When per-voxel material properties are required (use [RLE4 Compression](4441-rle4-compression.md) instead) or when random access to individual voxel bits is needed without decoding runs (use [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md)) [S1, unofficial].

## Background
This method is implemented in open-source voxelization tools such as `kctess5/voxelizer` and `topskychen/voxelizer` [S1, unofficial][S2, unofficial]. It is inspired by the `binvox` file format, which uses a similar compact binary run-length encoding for occupancy data [S1, unofficial]. The technique is commonly used in computer graphics for exporting voxelized geometry data for interchange with other applications [S1, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
