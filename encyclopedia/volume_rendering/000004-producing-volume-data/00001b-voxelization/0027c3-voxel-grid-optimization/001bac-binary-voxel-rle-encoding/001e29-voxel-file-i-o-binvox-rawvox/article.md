# Voxel File I/O (binvox/rawvox)
Also called: Binvox File Writing, Sparse Voxel Grid Export, Text-based Voxel Output, WriteBinvox

## Overview
Exports voxelized geometry data to standard file formats for interchange with other graphics applications. It implements two primary output modes: the `binvox` format, which uses a compact binary run-length encoding for occupancy data, and the `rawvox` format, which provides a simple binary dump of grid dimensions and metadata. Unlike its sibling [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md) which focuses on the internal compression algorithm, this method handles the specific file headers, coordinate transformations, and I/O protocols required by external tools.

## How it works
The method supports two distinct output formats: `binvox` (binary, sparse) and `rawvox` (binary, dense metadata). Both require a source voxel grid with defined dimensions, bounding box, and unit size.

### Inputs and Outputs
*   **Inputs**:
    *   `size_x_`, `size_y_`, `size_z_`: Integer grid dimensions.
    *   `lb_`: Lower bound coordinates of the bounding box (Vec3f).
    *   `unit_`: Voxel unit sizes (Vec3f).
    *   `bound_`: Bounding box object providing `norm()` method for normalization.
    *   `Filled(x, y, z)`: Boolean function returning true if the voxel at integer coordinates $(x, y, z)$ is occupied.
    *   `OutFilePath()`: Path string for the output file.
*   **Outputs**:
    *   A binary file containing either `binvox` or `rawvox` data.

### Data Structures
*   **Binvox Header**: Text-based ASCII lines defining format version, dimensions, translation, and scale.
*   **Binvox Data**: Binary stream of pairs `(value, count)`.
    *   `value`: 1 byte (unsigned char), 1 if occupied, 0 if empty.
    *   `count`: 1 byte (unsigned char), number of consecutive voxels with the same `value`.
*   **Rawvox Header**: Binary stream of floats.
    *   First 3 floats: `size_x_`, `size_y_`, `size_z_` (interpreted as floats or integers depending on implementation, but typically stored as floats in the raw header sequence described).
    *   Next 3 floats: `lb_[0]`, `lb_[1]`, `lb_[2]`.
    *   Next 3 floats: `unit_[0]`, `unit_[1]`, `unit_[2]`.

### Steps: Binvox Format
The `binvox` format stores data in row-major order (X, then Y, then Z) with normalization.

1.  **Calculate Normalization Parameters**:
    *   Let $B = \text{bound\_}.\text{norm()}$.
    *   Calculate `norm_translate` from `lb_`. The source implies a specific transformation, typically mapping the bounding box to a unit cube or similar canonical space. Based on the header line `translate -norm_translate[0] -norm_translate[2] -norm_translate[1]`, the coordinate system appears to swap Y and Z axes during translation.
    *   Calculate `norm_scale` from $B$.
2.  **Write Header**:
    *   Open file in binary mode.
    *   Write line 1: `#binvox 1\n`
    *   Write line 2: `dim {bx} {by} {bz}\n` where $bx=size\_x_, by=size\_y_, bz=size\_z_$.
    *   Write line 3: `translate {-norm\_translate[0]} {-norm\_translate[2]} {-norm\_translate[1]}\n`
    *   Write line 4: `scale {norm\_scale}\n`
    *   Write line 5: `data\n`
3.  **Encode Data**:
    *   Initialize $x = lx, y = ly, z = lz$ (grid bounds).
    *   Initialize `total_ones = 0`, `bytes_written = 0`.
    *   Loop while $x \le ux$:
        *   Get `value = Filled(x, y, z)`.
        *   Initialize `count = 0`.
        *   Inner Loop: While $x \le ux$ AND `count < 255` AND `value == Filled(x, y, z)`:
            *   Increment $z$.
            *   If $z > uz$:
                *   Reset $z = lz$.
                *   Increment $y$.
                *   If $y > uy$:
                    *   Reset $y = ly$.
                    *   Increment $x$.
            *   Increment `count`.
        *   If `value` is true, `total_ones += count`.
        *   Write `value` (1 byte) and `count` (1 byte) to file.
        *   `bytes_written += 2`.
4.  **Close File**.

### Steps: Rawvox Format
The `rawvox` format is a simpler binary dump of metadata.

1.  **Open File**: Open output file in binary mode.
2.  **Write Header**:
    *   Write `size_x_` (as float or int, source implies sequence of components).
    *   Write `size_y_`.
    *   Write `size_z_`.
    *   Write `lb_[0]`, `lb_[1]`, `lb_[2]` (cast to double/float).
    *   Write `unit_[0]`, `unit_[1]`, `unit_[2]` (cast to double/float).
3.  **Close File**.

### Steps: Voxel Metadata File Output (Variant)
A companion text-based format often used with rawvox.

1.  **Derive Path**: Strip `.rawvox` or `.binvox` extension from base path and append `.meta`.
2.  **Open File**: Open output file in text mode.
3.  **Write Header**:
    *   Write `#voxmeta 1\n`.
    *   Write `size_x_ size_y_ size_z_\n`.
4.  **Write Voxel Data**:
    *   Iterate `voxel_index` from $0$ to $\text{TotalVoxelSize()} - 1$.
    *   Write `voxel_index` and `flags` separated by space, followed by newline.
5.  **Close File**.

### Pseudocode: Binvox Writer

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_binvox(out_path, size_x, size_y, size_z, lb, bound, filled_func):
    """
    Writes a voxel grid to the binvox format.
    lb: tuple (lb_x, lb_y, lb_z)
    bound: object with norm() method returning a scalar or vector for normalization
    filled_func: callable(x, y, z) -> bool
    """
    # 1. Calculate normalization parameters
    # Note: Exact formula for norm_translate and norm_scale depends on bound_.norm() implementation.
    # Assuming bound_.norm() returns a bounding box norm or similar scale factor.
    # The source uses: translate -norm_translate[0] -norm_translate[2] -norm_translate[1]
    # This implies a Y/Z swap in the translation vector components.
    
    norm_translate = lb  # Placeholder: derive from lb_ as per source logic
    norm_scale = bound.norm()
    
    # 2. Write Header
    with open(out_path, 'wb') as f:
        f.write(b'#binvox 1\n')
        f.write(f'dim {size_x} {size_y} {size_z}\n'.encode('ascii'))
        # Y and Z components of translation are swapped in the header
        f.write(f'translate {-norm_translate[0]} {-norm_translate[2]} {-norm_translate[1]}\n'.encode('ascii'))
        f.write(f'scale {norm_scale}\n'.encode('ascii'))
        f.write(b'data\n')
        
        # 3. Encode Data
        lx, ly, lz = 0, 0, 0
        ux, uy, uz = size_x - 1, size_y - 1, size_z - 1
        
        x, y, z = lx, ly, lz
        while x <= ux:
            value = 1 if filled_func(x, y, z) else 0
            count = 0
            
            # Inner loop: run-length encode consecutive voxels with same value
            while x <= ux and count < 255 and (1 if filled_func(x, y, z) else 0) == value:
                z += 1
                if z > uz:
                    z = lz
                    y += 1
                    if y > uy:
                        y = ly
                        x += 1
                count += 1
            
            # Write value and count
            f.write(bytes([value, count]))
            
    return

def write_rawvox(out_path, size_x, size_y, size_z, lb, unit):
    """
    Writes voxel grid metadata to rawvox format.
    """
    with open(out_path, 'wb') as f:
        # Write dimensions
        f.write(struct.pack('fff', float(size_x), float(size_y), float(size_z)))
        # Write lower bound
        f.write(struct.pack('fff', lb[0], lb[1], lb[2]))
        # Write unit size
        f.write(struct.pack('fff', unit[0], unit[1], unit[2]))
```

## Subtopics

### [MagicaVoxel File Output](1752-magicavoxel-file-output.md)
Exports voxel particles to the MagicaVoxel `.vox` binary format. It handles grid sizing and the chunk-based structure required by the MagicaVoxel engine. Unlike the sparse binvox format, this often deals with colored voxel particles. [MagicaVoxel File Output](1752-magicavoxel-file-output.md)

### [MagicaVoxel File Parsing](2405-magicavoxel-file-parsing.md)
A parser for the `.vox` file format developed for Magica Voxel. It structures data into chunks containing tags, sizes, and child chunks. This is the inverse operation of the output format. [MagicaVoxel File Parsing](2405-magicavoxel-file-parsing.md)

### [PLY File Parsing](2781-ply-file-parsing.md)
Parses the ASCII header of a PLY file to determine vertex count, property types, and the byte offset of the binary data section. It handles the header by reading chunks until `end_header` is found. This is relevant for importing mesh data to be voxelized. [PLY File Parsing](2781-ply-file-parsing.md)

### [Voxel File I/O](2786-voxel-file-i-o.md)
General I/O utilities for voxel data, potentially including both reading and writing for various formats. [Voxel File I/O](2786-voxel-file-i-o.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the bounding box, as every voxel must be checked for occupancy. The binvox encoding step is also linear in the number of runs.
*   **Memory Complexity**: $O(1)$ auxiliary memory for the writer, as it streams data. The input grid must be stored in memory.
*   **Performance**: Binvox is significantly smaller in file size for sparse grids due to RLE. Rawvox is fast to write but produces larger files if metadata is dense.
*   **Strengths**: Standardized formats for interchange. Binvox is widely supported by 3D modeling tools.
*   **Limitations**: Binvox is binary and opaque without a parser. Rawvox lacks color/texture data in the basic header. Coordinate normalization in binvox can introduce precision issues if not handled carefully.

## Background
The `binvox` format was developed as a compact binary format for storing 3D voxel data, particularly for use in computer graphics and 3D modeling. It is widely used in datasets like ShapeNet and by tools like MagicaVoxel. The `rawvox` format is a simpler, less standardized binary format often used in research voxelizers for quick debugging or intermediate storage. The implementation described here originates from the `topskychen/voxelizer` project, which provides a C++ voxelization pipeline. [S1, unofficial]

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
