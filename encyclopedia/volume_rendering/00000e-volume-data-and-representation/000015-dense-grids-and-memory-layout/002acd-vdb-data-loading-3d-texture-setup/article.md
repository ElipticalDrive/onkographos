# VDB Data Loading & 3D Texture Setup

Also called: VDB Loader, VDB to 3D Texture, getVDBData

## Overview
This method loads a VDB-formatted binary file, parses its sparse voxel coordinates and tile data, and uploads the resulting dense volume as a 3D texture for GPU rendering [S1, unofficial]. It bridges the gap between sparse VDB storage and the dense grid requirements of hardware-accelerated ray marching [S1, unofficial]. Unlike [Bricking](914-bricking.md) which manages hierarchical tiling on the GPU, this method flattens the sparse data into a single dense 3D texture, relying on the parent [Dense grids and memory layout](3-dense-grids-and-memory-layout.md) for indexing [S1, unofficial].

## How it works
**Inputs:** A VDB binary file (ArrayBuffer) and a WebGL context.
**Outputs:** A bound 3D texture object containing the volume data.

**Data Structures:**
1.  **File Header:** Fixed-size binary structure.
    *   Offset 0-1: `width` (uint16)
    *   Offset 2-3: `height` (uint16)
    *   Offset 4-5: `depth` (uint16)
    *   Offset 6-7: `count` (uint16) [S1, unofficial]
    *   Offset 8+: `coordinates` (uint16 array, size $3 \times count$) [S1, unofficial]
    *   Offset $8 + 6 \times count$: `data` (uint8 array, size $512 \times count$) [S1, unofficial]
2.  **Cloud Data:** A `Uint8Array` of size $X \times Y \times Z$, initialized to zero.
3.  **3D Texture:** GPU resource with format `gl.RED`, type `gl.UNSIGNED_BYTE` [S1, unofficial].

**Steps:**
1.  **Cache Check:** Check `dataRepository` for an existing texture. If found, bind and return [S1, unofficial].
2.  **Texture Initialization:** Create a 3D texture, bind to `gl.TEXTURE0`, and initialize with a $1 \times 1 \times 1$ `R8` texture to reserve memory [S1, unofficial].
3.  **File Parsing:** Download the file as an `ArrayBuffer`. Parse the header to extract `resolution` ($X, Y, Z$) and `nodeCount` [S1, unofficial].
4.  **Dimension Calculation:** Compute grid dimensions in tiles: $D_x = X/8, D_y = Y/8, D_z = Z/8$. Note: The source implies $X, Y, Z$ are multiples of 8 [S1, unofficial].
5.  **Memory Allocation:** Allocate `cloudData` as a `Uint8Array` of size $X \times Y \times Z$ [S1, unofficial].
6.  **Data Extraction:** Parse `coordinates` as a `Uint16Array` starting at offset 8. Parse `nodeData` as a `Uint8Array` starting immediately after the coordinates block [S1, unofficial].
7.  **Tile Placement:** Iterate $i$ from 0 to `nodeCount` - 1:
    *   Extract tile coordinates $(tx, ty, tz)$ from `coordinates` at index $i \times 3$.
    *   Extract tile data from `nodeData` at index $i \times 512$ (512 bytes = $8 \times 8 \times 8$ uint8s).
    *   Call `setTile(cloudData, dimensions, [tx, ty, tz], tileData)` to place the 8x8x8 block into the dense grid [S1, unofficial].
8.  **Upload:** Upload `cloudData` to the 3D texture using `gl.texImage3D` with dimensions $X, Y, Z$, format `gl.RED`, and type `gl.UNSIGNED_BYTE` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_vdb_to_texture(file_buffer, gl, data_repository):
    # 1. Cache Check
    if data_repository.has_texture(file_buffer):
        return data_repository.get_texture(file_buffer)

    # 2. Parse Header
    # Offset 0-1: width, 2-3: height, 4-5: depth, 6-7: count
    width = file_buffer.get_uint16(0)
    height = file_buffer.get_uint16(2)
    depth = file_buffer.get_uint16(4)
    node_count = file_buffer.get_uint16(6)

    # 3. Calculate Dimensions
    # Dimensions in voxels
    dim_x = width
    dim_y = height
    dim_z = depth
    
    # Dimensions in tiles (8x8x8)
    tile_dim_x = dim_x // 8
    tile_dim_y = dim_y // 8
    tile_dim_z = dim_z // 8

    # 4. Allocate Cloud Data
    # Total voxels
    total_voxels = dim_x * dim_y * dim_z
    cloud_data = Uint8Array(total_voxels) # Initialized to 0

    # 5. Extract Coordinates and Data
    # Coordinates start at offset 8, each node has 3 uint16s
    coords_offset = 8
    coords_byte_length = node_count * 3 * 2 # 3 coords * 2 bytes
    coordinates = file_buffer.get_uint16_array(coords_offset, node_count * 3)
    
    # Data starts immediately after coordinates
    data_offset = coords_offset + coords_byte_length
    # Each node has 512 uint8s (8x8x8)
    node_data = file_buffer.get_uint8_array(data_offset, node_count * 512)

    # 6. Place Tiles
    for i in range(node_count):
        # Extract tile position (tx, ty, tz)
        tx = coordinates[i * 3]
        ty = coordinates[i * 3 + 1]
        tz = coordinates[i * 3 + 2]
        
        # Extract tile data (512 bytes)
        tile_data_start = i * 512
        tile_data = node_data[tile_data_start : tile_data_start + 512]
        
        # Place tile into cloud_data
        set_tile(cloud_data, tile_dim_x, tile_dim_y, tile_dim_z, 
                 tx, ty, tz, tile_data)

    # 7. Upload to GPU
    texture = gl.create_texture(gl.TEXTURE_3D)
    gl.bind_texture(gl.TEXTURE_3D, texture)
    
    # Initialize with 1x1x1 to reserve memory (implementation detail [G])
    gl.tex_image_3d(gl.TEXTURE_3D, 0, gl.R8, 1, 1, 1, 0, gl.RED, gl.UNSIGNED_BYTE, None)
    
    # Upload actual data
    gl.tex_image_3d(gl.TEXTURE_3D, 0, gl.R8, dim_x, dim_y, dim_z, 0, gl.RED, gl.UNSIGNED_BYTE, cloud_data)
    
    gl.bind_texture(gl.TEXTURE_3D, 0)
    
    # Cache and return
    data_repository.set_texture(file_buffer, texture)
    return texture

def set_tile(cloud_data, dim_x, dim_y, dim_z, tx, ty, tz, tile_data):
    """
    Places an 8x8x8 tile into the dense cloud_data array.
    Assumes tile_data is a Uint8Array of length 512.
    """
    # Base index for the tile in the dense grid
    # Using row-major mapping from parent [G]
    # i = tx, j = ty, k = tz
    # But cloud_data is dim_x * dim_y * dim_z
    # So stride_y = dim_x, stride_z = dim_x * dim_y
    
    stride_y = dim_x
    stride_z = dim_x * dim_y
    
    base_index = (tx * stride_z) + (ty * stride_y) + tz
    
    # Copy 512 bytes
    # tile_data[k] goes to cloud_data[base_index + k]
    # This assumes the tile data is stored in row-major order within the 8x8x8 block
    # matching the global grid's row-major order.
    for k in range(512):
        cloud_data[base_index + k] = tile_data[k]
```

**Parameters:**
*   `gl.R8` / `gl.RED` / `gl.UNSIGNED_BYTE`: GPU texture format for single-channel 8-bit data [S1, unofficial].

**Pitfalls:**
*   **Offset Calculation:** Coordinates and data offsets must be calculated based on `coordinates.byteLength` exactly as defined in the header layout. Mismatched offsets result in reading garbage data [S1, unofficial].
*   **Dimension Alignment:** The source implies dimensions are multiples of 8. If not, the tile placement logic may fail or require padding [S1, unofficial].

## Subtopics

### VDB 16-bit Tile Loading
Loads a VDB file format consisting of 8^3 leaf tiles, parsing the binary header and tile data, and uploading it as a 3D texture [S1, unofficial]. It differs from the standard loader by handling 16-bit data and compressing it to 8-bit for texture upload [S1, unofficial].

**Definition:** Parses a VDB binary file with 16-bit tile data and uploads it as an 8-bit 3D texture [S1, unofficial].

**Steps:**
1.  Parse header to get resolution (rounded up to multiples of 8), dimensions (resolution/8), and nodeCount [S1, unofficial].
2.  Extract coordinates and nodeData arrays from the binary buffer [S1, unofficial].
3.  For each node $i$, call `setTile(cloudData, dimensions, coordinates[i], nodeData[i*512])` [S1, unofficial].
4.  Compress 16-bit data to 8-bit: if `cloudData[i] > 0`, `compressedData[i] = 255 * (cloudData[i] / 65535)` [S1, unofficial].
5.  Upload `compressedData` to a 3D texture with format R8 [S1, unofficial].

**Layout:**
*   Offset 0-1: width (uint16, little endian)
*   Offset 2-3: height (uint16)
*   Offset 4-5: depth (uint16)
*   Offset 6-7: format (uint16)
*   Offset 8-9: sparse (uint16)
*   Offset 10-11: block size (uint16)
*   Offset 12-15: nodeCount (uint32)
*   Offset 16-19: normalization (float32)
*   Offset 20-31: offsets (int32[3])
*   Offset 32+: coordinates (int32[3] * nodeCount)
*   Followed by nodeData (uint16[512] * nodeCount) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes, as each node is processed once. Parsing and upload are linear in file size.
*   **Memory Complexity:** $O(X \times Y \times Z)$ for the dense cloud data, where $X, Y, Z$ are the full resolution dimensions. This can be significantly larger than the sparse VDB file size if the volume is sparse [S1, unofficial].
*   **Performance:** Fast for small to medium volumes. For large sparse volumes, the dense texture upload may be memory-intensive and slow due to the $O(XYZ)$ allocation [S1, unofficial].
*   **Strengths:** Simple implementation, leverages hardware texture sampling, compatible with standard ray marching shaders.
*   **Limitations:** Inefficient for highly sparse data due to dense memory allocation. No hierarchical culling.
*   **When to Use:** Small to medium volumes, or when the volume is dense enough that sparse overhead is negligible.
*   **When Not to Use:** Large, highly sparse volumes where [Bricking](914-bricking.md) or [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) would be more memory-efficient.

## Background
Originated from the `webgl-3d-math` library and the `volumetrics` project by al-ro [S1, unofficial]. It provides a straightforward way to load VDB data into WebGL for real-time volume rendering.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
