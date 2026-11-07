# VDB Data Loading & 3D Texture Setup

Also called: VDB Loader, VDB to 3D Texture, getVDBData

## Overview
This method loads a VDB-formatted binary file, parses its sparse voxel coordinates and tile data, and uploads the resulting dense volume as a 3D texture for GPU rendering [S1, unofficial]. It bridges the gap between the sparse, chunk-based VDB representation and the dense, hardware-sampled 3D texture required by standard ray marchers, building on [Dense grids and memory layout](3-dense-grids-and-memory-layout.md). Unlike [Bricking](914-bricking.md) which handles large datasets via tiling, this method assumes the entire volume fits into a single 3D texture object, prioritizing simplicity and direct GPU upload over memory efficiency for massive scenes.

## How it works
**Inputs:** A VDB binary file (or `ArrayBuffer`), a WebGL context, and an optional cache key.
**Outputs:** A bound WebGL 3D texture (`gl.TEXTURE_3D`) containing the volume data.

**Data Structures:**
1.  **VDB Header:** Fixed-size binary structure.
    *   `width` (uint16, offset 0)
    *   `height` (uint16, offset 2)
    *   `depth` (uint16, offset 4)
    *   `nodeCount` (uint16, offset 6)
2.  **Coordinates:** Array of `nodeCount` vectors, each stored as three `uint16` values (x, y, z). Total size: $3 \times \text{nodeCount} \times 2$ bytes. Starts at offset 8.
3.  **Node Data:** Array of `nodeCount` tiles, each containing $8 \times 8 \times 8 = 512$ `uint8` values. Total size: $512 \times \text{nodeCount}$ bytes. Starts immediately after coordinates.
4.  **Cloud Data:** A dense `Uint8Array` of size $W \times H \times D$, initialized to zero.

**Steps:**
1.  **Cache Check:** Check `dataRepository` for an existing texture bound to `gl.TEXTURE0`. If found, return it [S1, unofficial].
2.  **Texture Initialization:** Create a 3D texture object. Bind it to `gl.TEXTURE0`. Initialize with a $1 \times 1 \times 1$ `R8` texture to allocate GPU memory space [S1, unofficial].
3.  **File Parsing:** Download the file as an `ArrayBuffer`. Parse the header to extract `resolution` ($W, H, D$) and `nodeCount` [S1, unofficial].
4.  **Dimension Calculation:** Compute the grid dimensions in tiles: $\text{dims} = [\lfloor W/8 \rfloor, \lfloor H/8 \rfloor, \lfloor D/8 \rfloor]$. Note: The source implies integer division, assuming resolutions are multiples of 8 [S1, unofficial].
5.  **Memory Allocation:** Allocate `cloudData` as a `Uint8Array` of size $W \times H \times D$ [S1, unofficial].
6.  **Data Extraction:**
    *   Parse coordinates from offset 8 as a `Uint16Array`.
    *   Parse node data from offset $8 + \text{coordinates.byteLength}$ as a `Uint8Array` [S1, unofficial].
7.  **Tile Placement:** Iterate $i$ from $0$ to $\text{nodeCount} - 1$:
    *   Extract tile coordinates: $\mathbf{c} = \text{coordinates}[i \times 3 : i \times 3 + 3]$.
    *   Extract tile data: $\mathbf{d} = \text{nodeData}[i \times 512 : i \times 512 + 512]$.
    *   Call `setTile(cloudData, dims, \mathbf{c}, \mathbf{d})` to place the 512-byte tile into the dense array at the correct global voxel indices [S1, unofficial].
8.  **GPU Upload:** Upload `cloudData` to the 3D texture using `gl.texImage3D` with dimensions $W \times H \times D$, format `gl.RED`, and type `gl.UNSIGNED_BYTE` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_vdb_to_texture(buffer, gl, data_repository):
    # 1. Cache check
    cache_key = compute_hash(buffer)
    if cache_key in data_repository:
        return data_repository[cache_key]

    # 2. Parse Header
    # Offset 0-1: width (uint16), 2-3: height (uint16), 4-5: depth (uint16), 6-7: nodeCount (uint16)
    width = buffer.getUint16(0, true)  # little endian
    height = buffer.getUint16(2, true)
    depth = buffer.getUint16(4, true)
    node_count = buffer.getUint16(6, true)

    # 3. Calculate Dimensions
    # Assuming resolution is a multiple of 8 for tile alignment
    tile_dims_x = width // 8
    tile_dims_y = height // 8
    tile_dims_z = depth // 8

    # 4. Allocate Dense Cloud Data
    # Size = W * H * D bytes
    cloud_data = Uint8Array(width * height * depth)

    # 5. Parse Coordinates and Data
    # Coordinates start at offset 8, each node has 3 uint16s
    coords_offset = 8
    coords_array = Uint16Array(buffer, coords_offset, node_count * 3)
    
    # Data starts immediately after coordinates
    data_offset = coords_offset + (node_count * 3 * 2) # 2 bytes per uint16
    data_array = Uint8Array(buffer, data_offset, node_count * 512)

    # 6. Place Tiles
    for i in range(node_count):
        # Extract tile coordinates (x, y, z) in tile space
        tx = coords_array[i * 3]
        ty = coords_array[i * 3 + 1]
        tz = coords_array[i * 3 + 2]
        
        # Extract tile data (512 bytes)
        tile_data = data_array.subarray(i * 512, i * 512 + 512)
        
        # Place tile into dense array
        set_tile(cloud_data, width, height, depth, tx, ty, tz, tile_data)

    # 7. Create and Upload Texture
    texture = gl.createTexture()
    gl.bindTexture(gl.TEXTURE_3D, texture)
    
    # Initialize with 1x1x1 to allocate
    gl.texImage3D(gl.TEXTURE_3D, 0, gl.R8, 1, 1, 1, 0, gl.RED, gl.UNSIGNED_BYTE, None)
    
    # Upload actual data
    gl.texImage3D(gl.TEXTURE_3D, 0, gl.R8, width, height, depth, 0, gl.RED, gl.UNSIGNED_BYTE, cloud_data)
    
    # Set sampling parameters (standard for volume rendering)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_R, gl.CLAMP_TO_EDGE)

    # 8. Cache and Return
    data_repository[cache_key] = texture
    return texture

def set_tile(cloud_data, width, height, depth, tx, ty, tz, tile_data):
    """
    Places an 8x8x8 tile into the dense cloud_data array.
    tx, ty, tz are tile indices (0 to dims-1).
    """
    # Convert tile indices to global voxel start indices
    start_x = tx * 8
    start_y = ty * 8
    start_z = tz * 8
    
    # Iterate over the 8x8x8 tile
    for k in range(8):
        for j in range(8):
            for i in range(8):
                # Global voxel coordinates
                gx = start_x + i
                gy = start_y + j
                gz = start_z + k
                
                # Linear index in 1D array
                index = (gz * height + gy) * width + gx
                
                # Copy data
                # Tile data is stored in row-major order: x varies fastest, then y, then z
                # Assuming standard VDB tile layout: index = k*64 + j*8 + i
                tile_index = k * 64 + j * 8 + i
                cloud_data[index] = tile_data[tile_index]
```

**Parameters:**
*   `gl.R8` / `gl.RED` / `gl.UNSIGNED_BYTE`: The GPU texture format used for single-channel 8-bit data [S1, unofficial].

**Pitfalls:**
*   **Offset Calculation:** Coordinates and data offsets must be calculated precisely based on `coordinates.byteLength`. Mismatched offsets result in reading garbage data [S1, unofficial].
*   **Resolution Alignment:** The method assumes resolutions are multiples of 8. If not, the tile placement logic may fail or leave gaps [S1, unofficial].

## Subtopics

### VDB 16-bit Tile Loading
Loads a VDB file format consisting of 8^3 leaf tiles, parsing the binary header and tile data, and uploading it as a 3D texture [S1, unofficial]. It differs from the standard 8-bit loader by handling 16-bit precision data and compressing it to 8-bit for texture upload.
*   **Definition:** Parses a VDB binary file with 16-bit node data and uploads it as a 3D texture [S1, unofficial].
*   **Steps:**
    1.  Parse header to get resolution (rounded up to multiples of 8), dimensions (resolution/8), and nodeCount [S1, unofficial].
    2.  Extract coordinates and nodeData arrays from the binary buffer [S1, unofficial].
    3.  For each node $i$, call `setTile(cloudData, dimensions, coordinates[i], nodeData[i*512])` [S1, unofficial].
    4.  Compress 16-bit data to 8-bit: if `cloudData[i] > 0`, `compressedData[i] = 255 * (cloudData[i] / 65535)` [S1, unofficial].
    5.  Upload `compressedData` to a 3D texture with format R8 [S1, unofficial].
*   **Layout:** Binary file layout: Offset 0-1: width (uint16, little endian). Offset 2-3: height (uint16). Offset 4-5: depth (uint16). Offset 6-7: format (uint16). Offset 8-9: sparse (uint16). Offset 10-11: block size (uint16). Offset 12-15: nodeCount (uint32). Offset 16-19: normalization (float32). Offset 20-31: offsets (int32[3]). Offset 32+: coordinates (int32[3] * nodeCount). Followed by nodeData (uint16[512] * nodeCount) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes, as each node's tile is processed once. Parsing and upload are linear with respect to file size.
*   **Memory Complexity:** $O(W \times H \times D)$ for the dense `cloudData` array, where $W, H, D$ are the volume dimensions. This is significantly larger than the sparse VDB file size for sparse volumes.
*   **Performance:** Fast for small to medium volumes that fit in GPU memory. Upload time depends on bus bandwidth and texture size.
*   **Strengths:** Simple implementation, direct GPU upload, no complex tiling logic required.
*   **Limitations:** Does not support volumes larger than the maximum 3D texture size of the GPU. High memory overhead for sparse data.
*   **When to use:** Small volumes, prototyping, or when the entire volume fits in a single 3D texture.
*   **When not to use:** Large-scale volumes requiring out-of-core rendering or bricking.

## Background
Originating from the `webgl-3d-math` library and the `volumetrics` project by al-ro, this method provides a straightforward way to load VDB data into WebGL contexts [S1, unofficial]. It is commonly used in web-based volume visualization tools where simplicity and ease of integration are prioritized over handling massive datasets.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
