# Splat Texture Layout & Mapping

Also called: Splat Data Texture Mapping

## Overview
Splat Texture Layout & Mapping is a method for storing and accessing 3D Gaussian splat data within GPU 3D textures. It solves the problem of efficiently mapping a large, unstructured set of splats (each with position, covariance, color, opacity, etc.) into a fixed-size, hardware-sampled 3D texture volume. It differs from proxy geometry approaches by keeping data in texture memory for direct sampling, and from sparse volume textures by using a dense, pre-allocated texture structure with specific bit-packing for coordinates and layer indices.

## How it works
The method maps splat data to two 3D textures backed by `Uint32Array` buffers. Each splat corresponds to two 16-byte texture records. The layout is determined by calculating a maximum splat capacity and deriving bit-masks for coordinate indexing.

**Inputs and Outputs:**
- Input: A list of splats with attributes (position, scale, rotation, color, opacity, SH coefficients).
- Output: Two 3D textures (or their underlying `Uint32Array` buffers) containing the mapped splat data.

**Data Structures:**
1.  **Splat Buffer:** A `Uint32Array` of size `maxSplats * 4`. This likely stores packed 32-bit words representing splat attributes.
2.  **Texture Records:** Each splat occupies two 16-byte (4-word) records in the 3D texture.
3.  **3D Texture:** A 3D texture object backed by the `Uint32Array`. The dimensions are derived from `maxSplats`.

**Parameters:**
- `inputCapacity`: The number of splats provided by the user.
- `maxSplats`: The maximum number of splats the texture can hold, calculated via `getTextureSize(inputCapacity).maxSplats`.
- `SPLAT_TEX_WIDTH_BITS`: A constant defining the width bits of the texture.

**Steps:**
1.  **Calculate Capacity:** Determine `maxSplats` using `getTextureSize(inputCapacity).maxSplats`.
2.  **Allocate Memory:** If `inputCapacity` differs from `maxSplats`, allocate new `Uint32Array`s of size `maxSplats * 4`. Copy the original data into these new arrays.
3.  **Calculate Layer Bits:** Compute the number of bits required for the layer index:
    $$ \text{layerBits} = \text{SPLAT\_TEX\_WIDTH\_BITS} + \lceil \log_2(\text{height}) \rceil $$
    Where `height` is likely the depth or height dimension of the texture volume relevant to the splat distribution.
4.  **Create Mask:** Compute the source layer mask:
    $$ \text{sourceLayerMask} = 2^{\text{layerBits}} - 1 $$
5.  **Map Splat Coordinates:** For each splat index $i$, calculate its 3D texture coordinate $(x, y, z)$ using a function `splatTexCoord(i)`. This function handles paging or tiling for large splat counts to map the linear splat index to the 3D texture coordinates.
6.  **Write Data:** Write the splat's 16-byte data (packed into 4 `Uint32` words) into the texture record at the calculated coordinate. Since each splat uses two records, write to both `(x, y, z)` and `(x, y, z+1)` or similar adjacent locations, depending on the specific packing scheme.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_texture_layout_and_mapping(splats, splat_tex_width_bits):
    """
    Maps splats to two 3D textures backed by Uint32Arrays.
    splats: List of splat objects with attributes.
    splat_tex_width_bits: Integer constant for texture width bits.
    """
    # Step 1: Calculate max capacity
    # Note: getTextureSize is a helper function not defined here.
    # Assume it returns an object with 'maxSplats' field.
    max_splats = get_texture_size(len(splats)).max_splats
    
    # Step 2: Allocate buffers
    # Each splat maps to 4 uint32 words (16 bytes) per record, 2 records per splat.
    # Total words = max_splats * 4 words per splat * 2 records? 
    # The note says "allocate new Uint32Arrays of size maxSplats * 4".
    # This implies the buffer size is max_splats * 4 words, which might be per record or total.
    # Given "Each splat corresponds to two 16-byte texture records", 
    # and 16 bytes = 4 uint32s. 
    # If buffer size is max_splats * 4, it might be storing one record per splat 
    # in a flattened array, or the 4 refers to words per splat total?
    # Let's assume the buffer holds the data for one texture record per splat,
    # and the second record is handled similarly or interleaved.
    # The note says "allocate new Uint32Arrays of size maxSplats * 4".
    # Let's create two such arrays for the two textures/records.
    buffer1 = Uint32Array(max_splats * 4)
    buffer2 = Uint32Array(max_splats * 4)
    
    # Copy original data if capacity differs
    if len(splats) != max_splats:
        for i in range(len(splats)):
            # Pack splat i data into 4 uint32 words
            words = pack_splat_to_words(splats[i])
            buffer1[i * 4 : i * 4 + 4] = words
            buffer2[i * 4 : i * 4 + 4] = words # Assuming identical or related second record
    
    # Step 3: Calculate layer bits
    # Height is needed. Assume it's derived from max_splats or texture dimensions.
    # For now, assume height is a known parameter or derived from texture size.
    # Let's assume height is the depth of the texture.
    height = get_texture_height(max_splats) # Placeholder for actual height calculation
    layer_bits = splat_tex_width_bits + math.ceil(math.log2(height))
    
    # Step 4: Create mask
    source_layer_mask = (1 << layer_bits) - 1
    
    # Step 5 & 6: Map and write
    for i in range(len(splats)):
        # Calculate 3D texture coordinate
        coord = splat_tex_coord(i) # Returns ivec3 (x, y, z)
        
        # Write to texture 1
        write_to_texture_3d(buffer1, coord, splat_tex_width_bits, source_layer_mask)
        
        # Write to texture 2 (adjacent or specific second record location)
        # The note says "two 16-byte texture records". 
        # This might mean two separate textures or two records in the same texture.
        # Assuming two separate textures for simplicity based on "two 3D textures".
        write_to_texture_3d(buffer2, coord, splat_tex_width_bits, source_layer_mask)

def pack_splat_to_words(splat):
    """
    Packs a splat's attributes into 4 uint32 words.
    Implementation details depend on the specific attribute packing scheme.
    [G] This is a placeholder for the actual packing logic.
    """
    # Example: Position (3 floats -> 3 uint32s?), Scale, Rotation, Color, Opacity
    # This needs to be defined by the specific implementation.
    # Since the source doesn't provide the packing format, we leave it as a comment.
    # TODO: Define exact bit-packing for splat attributes into 4 uint32 words.
    return [0, 0, 0, 0]

def splat_tex_coord(splat_index):
    """
    Calculates the 3D texture coordinate for a given splat index.
    Handles paging or tiling for large splat counts.
    """
    # [G] This function is described in [S2, unofficial] as handling paged storage.
    # The exact algorithm is not provided in the notes.
    # It likely maps the linear index to (x, y, z) coordinates in the 3D texture.
    # Example logic (placeholder):
    # x = splat_index % width
    # y = (splat_index // width) % height
    # z = splat_index // (width * height)
    # This is a guess. The actual implementation may involve more complex paging.
    return ivec3(0, 0, 0)

def write_to_texture_3d(buffer, coord, width_bits, layer_mask):
    """
    Writes data from buffer to the 3D texture at the given coordinate.
    """
    # [G] This function maps the buffer data to the texture memory.
    # The exact mapping depends on the texture format and layout.
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point attributes must be carefully packed into `Uint32` words to avoid precision loss.
- **Boundaries:** Ensure splat coordinates fit within the 3D texture dimensions. Paging/tiling must handle splats that exceed the initial texture bounds.
- **Degenerate Input:** Empty splat lists or splats with zero scale should be handled gracefully.

## Subtopics

### Splat Coordinate Calculation
A function to calculate the 3D texture coordinate (ivec3) for a given splat index. It handles paging or tiling for large splat counts, ensuring that splats are distributed across the 3D texture volume efficiently. A variant explicitly handles paged storage, allowing for dynamic expansion of the texture as more splats are added. [S2, unofficial]

## Operating characteristics
- **Time Complexity:** O(N) for mapping N splats, where N is the number of splats.
- **Memory Complexity:** O(N) for storing the splat data in the 3D textures.
- **Performance:** Efficient for GPU rendering as data is directly accessible in texture memory. Sampling is fast using hardware texture sampling.
- **Strengths:** Simple mapping, efficient GPU access, supports large scenes with paging.
- **Limitations:** Fixed texture size may require reallocation and copying if capacity is exceeded. Packing/unpacking of attributes may be complex.
- **When to Use:** When rendering large numbers of Gaussian splats on the GPU with efficient texture sampling.
- **When Not to Use:** When memory is extremely constrained and sparse representations are more suitable, or when dynamic resizing is frequent and costly.

## Background
This method is part of the Gaussian Splatting rendering pipeline, specifically for efficient data storage and access on the GPU. It is used in implementations like `gaussian-splat-lite` [S1, unofficial] and `spark` [S2, unofficial]. The approach leverages 3D textures for their hardware-accelerated sampling capabilities, mapping the 3D Gaussian splats into a structured volume for efficient rendering.

## Sources
[S1] Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
[S2] spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
