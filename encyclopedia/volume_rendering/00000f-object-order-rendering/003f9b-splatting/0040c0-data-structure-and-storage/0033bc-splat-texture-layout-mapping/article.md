# Splat Texture Layout & Mapping

Also called: Splat Data Texture Mapping

## Overview
Splat Texture Layout & Mapping maps per-primitive splat data into two 3D textures backed by `Uint32Array` buffers for efficient GPU sampling [S1, unofficial]. Each splat corresponds to two 16-byte texture records within this layout [S1, unofficial].

## How it works
**Inputs:** Splat data array and `inputCapacity` (number of splats).
**Outputs:** Two 3D textures containing mapped splat data, and logic to calculate 3D texture coordinates for sampling.

**Data Structures:**
1. **Splat Buffer:** A flat array of `Uint32` values. Each splat occupies 4 `Uint32` slots (16 bytes) [S1, unofficial].
2. **3D Textures:** Two 3D textures backed by `Uint32Array` objects. Each splat corresponds to two 16-byte texture records within these textures [S1, unofficial].

**Algorithm:**
1. **Calculate Capacity:** Determine the maximum splat capacity using `getTextureSize(inputCapacity).maxSplats` [S1, unofficial].
2. **Allocate Memory:** If `inputCapacity` differs from `maxSplats`, allocate new `Uint32Array` buffers of size `maxSplats * 4` and copy the original data [S1, unofficial].
3. **Calculate Layer Bits:** Compute the bit width for the texture layer index: `layerBits = SPLAT_TEX_WIDTH_BITS + Math.ceil(Math.log2(height))` [S1, unofficial].
4. **Create Mask:** Generate a source layer mask: `sourceLayerMask = 2 ** layerBits - 1` [S1, unofficial].
5. **Map to Textures:** Distribute the splat data into the two 3D textures. The sources do not specify the exact packing scheme across the two textures [S1, unofficial].
6. **Calculate Coordinates:** Compute the 3D texture coordinate (`ivec3`) for a given splat index. The sources mention a `splatTexCoord` function that handles paging or tiling for large splat counts, but do not provide the implementation details [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def splat_texture_layout_and_mapping(splats, input_capacity, splat_tex_width_bits, height):
    # Step 1: Calculate capacity
    max_splats = get_texture_size(input_capacity).max_splats

    # Step 2: Allocate and copy
    buffer_size = max_splats * 4
    data_buffer = [0] * buffer_size
    if input_capacity != max_splats:
        for i in range(input_capacity):
            for j in range(4):
                data_buffer[i * 4 + j] = splats[i][j]
    else:
        data_buffer = list(splats)

    # Step 3 & 4: Layer bits and mask
    layer_bits = splat_tex_width_bits + math.ceil(math.log2(height))
    source_layer_mask = (2 ** layer_bits) - 1

    # Step 5: Map to two 3D textures
    # Sources do not specify the exact packing scheme across the two textures [S1, unofficial].
    texture_1 = create_3d_texture(max_splats, height, splat_tex_width_bits)
    texture_2 = create_3d_texture(max_splats, height, splat_tex_width_bits)

    for i in range(max_splats):
        record = data_buffer[i*4 : i*4+4]
        coord = calculate_splat_tex_coord(i, layer_bits, source_layer_mask, height, splat_tex_width_bits)
        # [G] Standard interleaved packing assumed for demonstration
        write_to_texture(texture_1, coord, record[:2])
        write_to_texture(texture_2, coord, record[2:])

    return texture_1, texture_2, data_buffer

def get_texture_size(capacity):
    # [G] Returns capacity info; implementation details not provided
    return type('obj', (object,), {'max_splats': capacity})()

def create_3d_texture(depth, height, width_bits):
    width = 1 << width_bits
    # [G] Returns a 3D array structure
    return [[[0 for _ in range(width)] for _ in range(height)] for _ in range(depth)]

def write_to_texture(texture, coord, data):
    x, y, z = coord
    texture[z][y][x] = data

def calculate_splat_tex_coord(splat_index, layer_bits, source_layer_mask, height, splat_tex_width_bits):
    # Sources do not provide the paging/tiling logic for coordinate calculation [S2, unofficial].
    # [G] Standard linear mapping to 3D coordinates
    width = 1 << (layer_bits - math.ceil(math.log2(height)))
    x = splat_index % width
    y = (splat_index // width) % height
    z = (splat_index // (width * height)) & source_layer_mask
    return (x, y, z)
```

**Edge Cases and Pitfalls:**
*   **Capacity Mismatch:** If `inputCapacity` does not match `maxSplats`, memory must be reallocated and data copied [S1, unofficial].
*   **Coordinate Calculation:** For large splat counts, the coordinate calculation must handle paging or tiling, but the specific logic is not provided in the sources [S2, unofficial].

## Subtopics

### [Hardware Accelerated Texture Atlases](8597-hardware-accelerated-texture-atlases.md)
This method organizes per-primitive texture data into large 2D atlas textures to leverage GPU hardware filtering and caching, thereby avoiding multiple texture bindings and software-based sampling overhead. It sets itself apart from software-based interpolation by utilizing dedicated texture units for bilinear filtering, which is faster than manual interpolation from global memory arrays.

### Splat Coordinate Calculation
A function to calculate the 3D texture coordinate (`ivec3`) for a given splat index, likely handling paging or tiling for large splat counts [S2, unofficial]. A variant explicitly handles paged storage [S2, unofficial].

## Operating characteristics
**Time Complexity:** O(N) for allocation and mapping, where N is the number of splats [S1, unofficial].
**Memory Complexity:** O(N) for the `Uint32Array` buffers and 3D textures [S1, unofficial].
**Strengths:** Efficient GPU sampling via hardware filtering [S1, unofficial].
**Limitations:** Fixed texture size limits the maximum number of splats without complex paging [S2, unofficial].

## Background
This method is part of the Gaussian Splatting rendering pipeline, specifically for efficient data transfer and sampling on the GPU. It is implemented in libraries such as `gaussian-splat-lite` [S1, unofficial].

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S2] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
