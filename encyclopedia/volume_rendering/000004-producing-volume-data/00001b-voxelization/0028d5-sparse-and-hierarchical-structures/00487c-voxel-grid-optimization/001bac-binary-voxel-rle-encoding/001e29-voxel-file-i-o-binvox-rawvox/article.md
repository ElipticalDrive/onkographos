# Voxel File I/O (binvox/rawvox)
Also called: Binvox File Writing, Sparse Voxel Grid Export, Text-based Voxel Output, WriteBinvox

## Overview
This method serializes a 3D voxel grid into specific legacy file formats, primarily the `binvox` binary format and the `rawvox` text/binary hybrid format, to enable interoperability with external visualization tools and pipelines [S1, S3, unofficial]. It differs from generic voxel I/O by implementing the strict header structures and data ordering (row-major vs column-major) required by these specific standards, rather than providing a flexible, format-agnostic serialization interface [S1, S4, unofficial].

## How it works
The method supports multiple output variants depending on the target format and compression requirements. The core logic involves writing a header describing grid dimensions, spatial bounds, and voxel size, followed by the voxel data itself.

### Inputs and Outputs
*   **Inputs**:
    *   `size_x_, size_y_, size_z_`: Integer dimensions of the voxel grid [S1, S3, unofficial].
    *   `lb_`: Lower bound coordinates of the bounding box (Vec3f or similar) [S1, unofficial].
    *   `unit_`: Voxel unit sizes (Vec3f or similar) [S1, unofficial].
    *   `bound_`: Bounding box object containing normalization parameters (`norm()` method) [S1, unofficial].
    *   `Filled(x, y, z)`: A predicate returning `true` if the voxel at integer coordinates $(x, y, z)$ is occupied [S1, unofficial].
    *   `_grid`: A 1D or 3D array of voxel data (e.g., `uint16_t` or `uint8_t`) [S2, S3, unofficial].
    *   `_numDivs`: Grid dimensions as `glm::uvec3` [S2, unofficial].
*   **Outputs**:
    *   Binary or text files written to a specified path [S1, S3, unofficial].

### Data Structures and Layouts

#### Binvox Format
*   **Header**:
    1.  Line 1: `#binvox 1` [S1, unofficial].
    2.  Line 2: `dim bx by bz` (integer dimensions) [S1, unofficial].
    3.  Line 3: `translate tx ty tz` (normalized translation coordinates) [S1, unofficial].
    4.  Line 4: `scale s` (normalized scale factor) [S1, unofficial].
    5.  Line 5: `data` [S1, unofficial].
*   **Data**:
    *   Run-Length Encoded (RLE) pairs of `(value, count)` [S1, unofficial].
    *   `value`: 1 byte (`0` or `1` for occupancy) [S1, unofficial].
    *   `count`: 1 byte (number of consecutive voxels, max 255) [S1, unofficial].
    *   Total bytes written is even (2 bytes per run) [S1, unofficial].

#### Rawvox Format
*   **Header**:
    *   Text line 1: `size_x_ size_y_ size_z_` [S1, unofficial].
    *   Text line 2: `lb_[0] lb_[1] lb_[2]` (lower bounds as doubles) [S1, unofficial].
    *   Text line 3: `unit_[0] unit_[1] unit_[2]` (voxel sizes as doubles) [S1, unofficial].
*   **Data**:
    *   **Sparse (Text)**: Lines containing `x y z` for each filled voxel [S1, unofficial].
    *   **Dense (Binary)**: Byte-wise material values (`uint8_t`) [S3, unofficial].
    *   **Layout**: Can be `row_major` or `column_major` (Fortran order) [S3, unofficial]. Column-major is common for NumPy compatibility [S3, unofficial].

#### Raw Compressed (Squared) Format
*   **Header**:
    *   If not squared: `_numDivs` (`glm::uvec3`) followed by raw grid data [S2, unofficial].
    *   If squared: `end` (`glm::uvec3`, max dimension) followed by padded grid data [S2, unofficial].
*   **Data**:
    *   `uint16_t` voxel values [S2, unofficial].
    *   Empty space filled with `VOXEL_EMPTY` constant [S2, unofficial].

### Pseudocode

#### Binvox Writer
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_binvox(path, size_x, size_y, size_z, lb, bound, filled_func):
    """
    Writes a voxel grid to the binvox binary format.
    """
    # Calculate normalization parameters
    norm_translate = lb  # Assuming lb_ provides translation
    norm_scale = bound.norm()
    
    with open(path, 'wb') as f:
        # Write Header
        f.write(b'#binvox 1\n')
        f.write(f'dim {size_x} {size_y} {size_z}\n'.encode())
        f.write(f'translate {-norm_translate[0]} {-norm_translate[2]} {-norm_translate[1]}\n'.encode())
        f.write(f'scale {norm_scale}\n'.encode())
        f.write(b'data\n')
        
        # Write Data using RLE
        x = 0
        y = 0
        z = 0
        total_bytes = 0
        
        while x < size_x:
            value = 1 if filled_func(x, y, z) else 0
            count = 0
            
            # Inner loop for RLE
            while x < size_x and count < 255 and (1 if filled_func(x, y, z) else 0) == value:
                z += 1
                if z >= size_z:
                    z = 0
                    y += 1
                    if y >= size_y:
                        y = 0
                        x += 1
                        if x >= size_x:
                            break
                count += 1
            
            # Write value and count
            f.write(bytes([value]))
            f.write(bytes([count]))
            total_bytes += 2
            
    return total_bytes
```

#### Rawvox Sparse Writer
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_rawvox_sparse(path, size_x, size_y, size_z, lb, unit, filled_func):
    """
    Writes a voxel grid to the rawvox text format (sparse).
    """
    with open(path, 'w') as f:
        # Write Header
        f.write(f'{size_x} {size_y} {size_z}\n')
        f.write(f'{lb[0]} {lb[1]} {lb[2]}\n')
        f.write(f'{unit[0]} {unit[1]} {unit[2]}\n')
        
        # Write Voxel Coordinates
        count = 0
        for x in range(size_x):
            for y in range(size_y):
                for z in range(size_z):
                    if filled_func(x, y, z):
                        f.write(f'{x} {y} {z}\n')
                        count += 1
```

#### Rawvox Dense Writer (Column-Major)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_rawvox_dense(path, size_x, size_y, size_z, grid_data, byte_order='column_major'):
    """
    Writes a voxel grid to the rawvox binary format (dense).
    grid_data: 1D array of uint8_t.
    """
    with open(path, 'wb') as f:
        # Write Header (Dimensions only, or full header if required by specific variant)
        # Note: S3 implies the .info file holds metadata, but S1 implies header in file.
        # Assuming standard rawvox header for compatibility with S1 logic if needed,
        # but S3 example focuses on the binary data itself.
        # We write the binary data directly as per S3 example.
        
        if byte_order == 'column_major':
            # Reshape logic handled by reader, but writer must ensure
            # the 1D array corresponds to column-major layout.
            # If grid_data is already in column-major order:
            f.write(grid_data.tobytes())
        else:
            # Row-major: standard C order
            f.write(grid_data.tobytes())
```

#### Raw Compressed Squared Writer
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def write_rawvox_squared(path, num_divs, grid_data, squared=True):
    """
    Writes voxel grid, optionally padding to a cube.
    grid_data: 1D array of uint16_t.
    """
    with open(path, 'wb') as f:
        if not squared:
            # Write dimensions
            f.write(num_divs.x.to_bytes(4, 'little'))
            f.write(num_divs.y.to_bytes(4, 'little'))
            f.write(num_divs.z.to_bytes(4, 'little'))
            # Write raw grid
            f.write(grid_data.tobytes())
        else:
            # Calculate end dimension
            end_val = max(num_divs.x, num_divs.y, num_divs.z)
            end = (end_val, end_val, end_val)
            
            # Calculate start offset
            start_x = (end_val - num_divs.x) // 2
            start_y = (end_val - num_divs.y) // 2
            start_z = (end_val - num_divs.z) // 2
            
            # Write end dimensions
            f.write(end[0].to_bytes(4, 'little'))
            f.write(end[1].to_bytes(4, 'little'))
            f.write(end[2].to_bytes(4, 'little'))
            
            # Write padded data
            for x in range(end[0]):
                for y in range(end[1]):
                    for z in range(end[2]):
                        current_pos = (x - start_x, y - start_y, z - start_z)
                        if (0 <= current_pos[0] < num_divs.x and
                            0 <= current_pos[1] < num_divs.y and
                            0 <= current_pos[2] < num_divs.z):
                            # Write voxel from grid
                            # Index calculation depends on grid layout (assume row-major for simplicity)
                            idx = (current_pos[0] + 
                                   current_pos[1] * num_divs.x + 
                                   current_pos[2] * num_divs.x * num_divs.y)
                            voxel = grid_data[idx]
                            f.write(voxel.to_bytes(2, 'little'))
                        else:
                            # Write VOXEL_EMPTY
                            f.write(VOXEL_EMPTY.to_bytes(2, 'little'))
```

### Parameters and Edge Cases
*   **Binvox RLE Limit**: The count byte is limited to 255. If a run exceeds 255 voxels, it must be split into multiple entries [S1, unofficial].
*   **Coordinate Order**: Binvox uses a specific coordinate system where Z is the vertical axis, and the translate values are negated in the header [S1, unofficial].
*   **Rawvox Byte Order**: The `rawvox` format does not enforce a single byte order; the consumer (e.g., NumPy) must know if the data is `row_major` or `column_major` (Fortran order) to reshape correctly [S3, unofficial].
*   **Squared Padding**: When padding to a cube, the offset calculation `(end - num_divs) / 2` centers the original grid within the new cube [S2, unofficial].

## Subtopics

### [MagicaVoxel File Output](1752-magicavoxel-file-output.md)
Exports voxel particles to the MagicaVoxel `.vox` binary format, handling grid sizing and chunk structure. It differs from generic voxel I/O by enforcing the specific chunk-based binary layout and resolution limits required by the MagicaVoxel application.

### [MagicaVoxel File Parsing](2405-magicavoxel-file-parsing.md)
A loader for MagicaVoxel `.vox` files (versions 150 and 200) that parses models, palette, MATL materials, and the nTRN/nGRP/nSHP scene graph, converting visible faces into quads within a BVH. It structures data into chunks containing tags, sizes, and child chunks. Unlike [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md) which uses dense or simple sparse grids, this method handles the hierarchical scene graph and chunk-based binary format specific to Magica Voxel.

### [PLY File Parsing](2781-ply-file-parsing.md)
This method parses the ASCII header of a Polygon File Format (PLY) file to determine vertex counts, property types, and the byte offset of the subsequent data section. It serves as the initial step in extracting structured mesh geometry (vertices and faces) for voxelization pipelines, specifically targeting the ASCII format used in outputs from 3D Gaussian Splatting tutorials. Unlike binary PLY parsers or OBJ loaders, this implementation assumes triangular topology and ignores quad faces, providing the raw data required for subsequent normal calculation and voxelization.

### [Voxel File I/O](2786-voxel-file-i-o.md)
This method handles the serialization and deserialization of voxel grid data into various file formats for storage, transport, or interoperability with external tools. It solves the problem of persisting 3D volumetric data generated by voxelization or volume rendering pipelines in a structured, recoverable format. Unlike [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md), which focuses on specific legacy binary encodings, this article covers a broader set of generic and specialized formats including ASCII text, binary grids, and specialized image stacks.

### Voxel Metadata File Output
Derives a metadata file path by stripping the `.rawvox` or `.binvox` extension and appending `.meta` [S1, unofficial]. It writes a header `#voxmeta 1`, followed by grid dimensions, and then iterates through all voxels to write the voxel index and flags [S1, unofficial].

### Rawvox File Writing
Opens the output file in binary mode and writes the header consisting of `size_x_ size_y_ size_z_`, followed by the lower bound components `lb_`, and then the unit components `unit_` [S1, unofficial].

### Raw Compressed Export (Squared)
Exports the voxel grid to a binary file, optionally padding the grid to a cube (squared) by filling empty space with `VOXEL_EMPTY` [S2, unofficial]. If not squared, it writes `_numDivs` and the raw grid. If squared, it calculates the max dimension `end`, writes `end`, and then writes the grid data padded with `VOXEL_EMPTY` for out-of-bounds coordinates [S2, unofficial].

### Raw Voxel Export Format
Exports voxels as a byte-wise binary file where each byte corresponds to a material value defined in the XML configuration (`material_interior` or `material_shell`) [S3, unofficial]. The data is typically loaded in Python/NumPy by reshaping the 1D `uint8` array into 3D using column-major (`order='F'`) order [S3, unofficial].

## Operating characteristics
*   **Binvox**:
    *   **Compression**: RLE compression is effective for sparse or blocky data but adds overhead for random noise.
    *   **Performance**: Writing is $O(N)$ where $N$ is the number of voxels. RLE encoding requires a single pass.
    *   **Limitations**: Limited to binary occupancy (no color/alpha in standard binvox). Max run length 255.
*   **Rawvox**:
    *   **Compression**: None (unless wrapped in external compression).
    *   **Performance**: Very fast I/O. Dense binary write is $O(N)$. Sparse text write is $O(N_{filled})$ but larger file size.
    *   **Limitations**: No standard compression. Requires external metadata for interpretation (bounds, units).
*   **Memory**: Minimal auxiliary memory required during write, except for buffering if RLE is implemented with lookahead.

## Background
The `binvox` format is a widely used standard for storing 3D voxel data, particularly in computer graphics and robotics for occupancy grids [S1, unofficial]. The `rawvox` format is a simpler, often custom binary/text format used in various voxelization tools (e.g., VoxelMagick) for easy parsing by scientific computing libraries like NumPy [S3, unofficial]. These formats predate more modern hierarchical structures like OpenVDB or Draco, serving as lightweight interchange formats for dense and sparse voxel grids.

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S2] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
[S3] 3DStuff/voxelizer, https://github.com/3DStuff/voxelizer, fetched 2026-10-27 (unofficial)
