# VDB Data Loading & 3D Texture Setup

Also called: VDB Loader, VDB to 3D Texture, getVDBData

## Overview
This method loads a VDB-formatted binary file, parses its sparse voxel coordinates and tile data, and uploads the resulting density field as a 3D texture for GPU rendering [S1, unofficial]. It bridges the gap between CPU-side sparse data structures and the GPU's dense 3D texture sampling capabilities, serving as the initialization step for [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md). Unlike [Bricking](914-bricking.md) which manages hierarchical tiling for streaming, this method typically loads the entire sparse dataset into a single dense volume texture, making it suitable for scenes where the total voxel count fits within GPU memory limits.

## How it works
**Inputs:** A VDB binary file (ArrayBuffer) and a WebGL context.
**Outputs:** A WebGL 3D texture object containing the density data.

**Data Structures:**
1.  **File Header:** Fixed-size binary structure.
    *   `width` (uint16, offset 0)
    *   `height` (uint16, offset 2)
    *   `depth` (uint16, offset 4)
    *   `nodeCount` (uint16, offset 6)
2.  **Coordinates:** `Uint16Array` of size `3 * nodeCount`, starting at offset 8. Each node has 3 coordinates (x, y, z).
3.  **Node Data:** `Uint8Array` of size `512 * nodeCount`, starting immediately after coordinates. Each node contains 512 bytes (8x8x8 tiles of 8-bit data).
4.  **Cloud Data:** `Uint8Array` of size `width * height * depth`, allocated in CPU memory to hold the dense volume.

**Parameters:**
*   Texture Format: `gl.R8` / `gl.RED`
*   Texture Type: `gl.UNSIGNED_BYTE`
*   Texture Unit: `gl.TEXTURE0`

**Steps:**
1.  **Cache Check:** Check if a texture for this specific VDB file already exists in the `dataRepository`. If yes, return the cached texture.
2.  **Texture Initialization:** Create a new 3D texture object. Bind it to `gl.TEXTURE0`. Initialize it as a minimal `1x1x1` texture with format `gl.RED` and type `gl.UNSIGNED_BYTE` to satisfy WebGL state requirements before upload.
3.  **File Parsing:**
    *   Read `resolution` array: `[getUint16(0), getUint16(2), getUint16(4)]`.
    *   Read `nodeCount`: `getUint16(6)`.
    *   Calculate `dimensions`: `[resolution[0]/8, resolution[1]/8, resolution[2]/8]`. Note: This assumes resolution is a multiple of 8.
    *   Allocate `cloudData` as a `Uint8Array` of size `resolution[0] * resolution[1] * resolution[2]`.
4.  **Data Extraction:**
    *   Parse `coordinates` from offset 8 as a `Uint16Array`.
    *   Parse `nodeData` from offset `8 + coordinates.byteLength` as a `Uint8Array`.
5.  **Tile Placement:** Iterate `i` from `0` to `nodeCount - 1`:
    *   Extract tile coordinates: `coords = coordinates.subarray(i*3, i*3+3)`.
    *   Extract tile data: `tile = nodeData.subarray(i*512, i*512+512)`.
    *   Call `setTile(cloudData, dimensions, coords, tile)` to place the 8x8x8 tile into the correct position in `cloudData`.
6.  **GPU Upload:** Upload `cloudData` to the bound 3D texture using `gl.texImage3D` with dimensions `resolution`, format `gl.RED`, type `gl.UNSIGNED_BYTE`, and data `cloudData`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_vdb_to_texture(file_buffer, gl, data_repository):
    # 1. Cache Check
    texture_key = file_buffer.id # Assuming unique ID for caching
    if texture_key in data_repository:
        return data_repository[texture_key]

    # 2. Texture Initialization
    texture = gl.createTexture()
    gl.bindTexture(gl.TEXTURE_3D, texture)
    # Initialize with 1x1x1 to satisfy WebGL requirements
    gl.texImage3D(
        gl.TEXTURE_3D, 0, gl.RED, 1, 1, 1, 0, gl.RED, gl.UNSIGNED_BYTE, None
    )

    # 3. File Parsing
    # Read header
    width = file_buffer.getUint16(0, little_endian=True)
    height = file_buffer.getUint16(2, little_endian=True)
    depth = file_buffer.getUint16(4, little_endian=True)
    node_count = file_buffer.getUint16(6, little_endian=True)

    resolution = [width, height, depth]
    dimensions = [width // 8, height // 8, depth // 8]
    
    # Allocate dense cloud data
    cloud_data_size = width * height * depth
    cloud_data = bytearray(cloud_data_size)

    # 4. Data Extraction
    # Coordinates start at offset 8
    coords_offset = 8
    coords_byte_length = node_count * 3 * 2 # 3 uint16 per node
    coordinates = file_buffer.getUint16Array(coords_offset, node_count * 3)
    
    # Node data starts immediately after coordinates
    data_offset = coords_offset + coords_byte_length
    node_data = file_buffer.getUint8Array(data_offset, node_count * 512)

    # 5. Tile Placement
    for i in range(node_count):
        # Extract 3 coordinates for this node
        x = coordinates[i * 3]
        y = coordinates[i * 3 + 1]
        z = coordinates[i * 3 + 2]
        
        # Extract 512 bytes for this tile
        tile_data = node_data[i * 512 : (i + 1) * 512]
        
        # Place tile into cloud_data
        set_tile(cloud_data, dimensions, [x, y, z], tile_data)

    # 6. GPU Upload
    gl.texImage3D(
        gl.TEXTURE_3D, 0, gl.RED, width, height, depth, 0, gl.RED, gl.UNSIGNED_BYTE, cloud_data
    )
    
    # Set texture parameters (e.g., filtering)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
    gl.texParameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_R, gl.CLAMP_TO_EDGE)

    # Cache and return
    data_repository[texture_key] = texture
    return texture

def set_tile(cloud_data, dimensions, coords, tile_data):
    """
    Places an 8x8x8 tile into the dense cloud_data array.
    coords: [x, y, z] in tile coordinates (not voxel coordinates)
    tile_data: 512 bytes (8x8x8)
    """
    x, y, z = coords
    tile_width = 8
    tile_height = 8
    tile_depth = 8
    
    # Calculate base index in the dense array
    # Assuming row-major order: x is fastest, then y, then z
    # Or column-major? VDB typically uses x,y,z order. 
    # Standard dense grid indexing: index = x + y*width + z*width*height
    base_index = x + y * dimensions[0] + z * dimensions[0] * dimensions[1]
    
    # Copy tile data
    # Note: The source implies direct copy. The internal layout of the 8x8x8 tile 
    # within the 512 bytes must match the dense array's memory layout.
    # Assuming the tile data is already in the correct linear order for the dense array.
    for i in range(512):
        cloud_data[base_index + i] = tile_data[i]
```

**Edge Cases and Pitfalls:**
*   **Offset Calculation:** Coordinates and data offsets must be calculated precisely based on `coordinates.byteLength`. Mismatched offsets result in reading garbage data [S1, unofficial].
*   **Resolution Alignment:** The method assumes resolution is a multiple of 8. If not, `dimensions` calculation may fail or produce incorrect tile counts.
*   **Memory Usage:** The `cloudData` array is dense. For large volumes with high sparsity, this can exceed GPU memory limits.

## Subtopics

### VDB 16-bit Tile Loading
Loads a VDB file format consisting of 8^3 leaf tiles, parsing the binary header and tile data, and uploading it as a 3D texture [S1, unofficial]. It differs from the standard 8-bit loader by handling 16-bit data and compressing it to 8-bit for texture upload.

**Definition:** Parses a more complex VDB header (including format, sparse, block size, normalization, and offsets) and loads 16-bit tile data, compressing it to 8-bit for GPU storage [S1, unofficial].

**Steps:**
1.  **Parse Header:** Extract resolution (rounded up to multiples of 8), dimensions (resolution/8), and nodeCount from the extended header.
2.  **Extract Data:** Extract coordinates and nodeData arrays from the binary buffer. Coordinates are `int32[3] * nodeCount`. Node data is `uint16[512] * nodeCount`.
3.  **Tile Placement:** For each node `i`, call `setTile(cloudData, dimensions, coordinates[i], nodeData[i*512])`.
4.  **Compression:** Compress 16-bit data to 8-bit: if `cloudData[i] > 0`, `compressedData[i] = 255 * (cloudData[i] / 65535)`.
5.  **Upload:** Upload `compressedData` to a 3D texture with format `R8`.

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
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes, as each node is processed once. Parsing and upload are linear in the size of the sparse data.
*   **Memory Complexity:** $O(W \times H \times D)$ for the dense cloud data, where $W, H, D$ are the volume dimensions. This is the primary limitation; memory usage is independent of sparsity.
*   **Performance:** Upload time depends on the size of the dense volume. For large volumes, this can be a bottleneck.
*   **Strengths:** Simple implementation; leverages GPU hardware filtering; fast sampling during rendering.
*   **Limitations:** High memory consumption for large or sparse volumes; not suitable for very large datasets that exceed GPU memory.
*   **When to Use:** Small to medium volumes that fit in GPU memory; scenes requiring fast, hardware-accelerated ray marching.
*   **When Not to Use:** Very large volumes; highly sparse volumes where memory efficiency is critical.

## Background
This method is derived from the `webgl-3d-math` library and the `volumetrics` project by al-ro, which provides utilities for 3D math and volume rendering in WebGL [S1, unofficial]. It represents a common approach in WebGL-based volume rendering where sparse data structures are converted to dense textures for compatibility with GPU texture sampling units.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
