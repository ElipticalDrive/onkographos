# Texture Creation from Palette

Also called: FVox::CreateTexture, Palette Texture Setup with Endianness Reversal

## Overview
This method constructs a 2D RGBA8 texture from a 1D palette array for use in volume rendering, specifically handling the byte-order conversion required when the source data is stored as little-endian 32-bit integers. It differs from general volume texture generation by treating the palette as a 1D lookup table mapped to a 256x1 texture, and explicitly addresses the endianness reversal pitfall where shader sRGB interpretation relies on the specific R, G, B, A byte layout resulting from the little-endian interpretation of the reversed uint.

## How it works
**Inputs:**
- `Palette`: A 1D array of 32-bit unsigned integers (`uint[]`) representing color indices or RGBA values.
- `OutTexture`: A texture object instance to be populated.

**Outputs:**
- `OutTexture`: A fully initialized 2D texture ready for GPU binding.

**Data Structures:**
- `Palette`: `uint[]` of size $N=256$.
- `OutTexture`: 2D texture with dimensions $W=256, H=1, D=1, Layers=1$.
- Format: `TSF_BGRA8` (or equivalent RGBA8 with specific byte ordering).

**Steps:**

1. **Initialize Texture Properties:**
   Set the texture metadata to ensure correct rendering and memory management:
   - `LODGroup`: Set to `TEXTUREGROUP_World`.
   - `CompressionSettings`: Set to `TC_Default`.
   - `MipGenSettings`: Set to `TMGS_NoMipmaps`.
   - `DeferCompression`: Set to `true` [S1, unofficial].

2. **Convert and Copy Memory:**
   Convert the `uint[]` palette data into a `byte[]` array suitable for the texture format. This step involves two critical operations:
   - **Endianness Reversal:** The source `uint` values must be interpreted with reversed endianness relative to the native system or shader expectation.
   - **Byte Packing:** The resulting bytes must be arranged in the specific order R, G, B, A. This relies on the little-endian byte order interpretation of the reversed uint [S2, unofficial].
   
   In C#, this is achieved using `MemoryMarshal.AsBytes` on the reversed or appropriately cast data. The raw byte copy must be performed such that the shader reads the texture as sRGB and sees the correct channel order [S2, unofficial].

3. **Initialize Texture Source:**
   Configure the texture source with the following parameters:
   - Width: 256
   - Height: 1
   - Depth: 1
   - Layers: 1
   - Format: `TSF_BGRA8`
   - Data: The byte array from Step 2, derived from `Palette.GetData()` [S1, unofficial].

4. **Create and Bind Texture:**
   Create the 256x1 RGBA8 texture from the byte data. Bind this texture to the shader under the uniform name `'palette'` [S2, unofficial].

5. **Finalize Texture:**
   Call `OutTexture->UpdateResource()` to upload the data to the GPU. Call `OutTexture->PostEditChange()` to notify the engine of the modification [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_palette_texture(palette_uints: list[int]) -> Texture:
    """
    Creates a 256x1 RGBA8 texture from a 256-element uint palette,
    handling endianness reversal and byte packing for sRGB shader reading.
    """
    # Step 1: Initialize Texture Properties
    texture = Texture()
    texture.LODGroup = "TEXTUREGROUP_World"
    texture.CompressionSettings = "TC_Default"
    texture.MipGenSettings = "TMGS_NoMipmaps"
    texture.DeferCompression = True
    
    # Step 2: Convert and Copy Memory
    # The palette is 256 uints. We need to convert to bytes with specific endianness.
    # Source [S2, unofficial] states: "Convert the uint[] array to a byte[] array using MemoryMarshal.AsBytes.
    # This relies on the little-endian byte order being R, G, B, A."
    # And: "The endianness reversal and the raw byte copy must be changed together..."
    
    # Assuming the uints are in native endianness and need to be reversed for the shader's
    # little-endian R,G,B,A interpretation.
    reversed_uints = [reverse_endianness(u) for u in palette_uints]
    
    # Pack into bytes. In little-endian, the first byte is the least significant.
    # If the uint is 0xAABBGGRR (reversed), little-endian bytes are [RR, GG, BB, AA].
    # This matches the required R, G, B, A order for the shader.
    byte_data = []
    for u in reversed_uints:
        byte_data.append(u & 0xFF)          # R
        byte_data.append((u >> 8) & 0xFF)   # G
        byte_data.append((u >> 16) & 0xFF)  # B
        byte_data.append((u >> 24) & 0xFF)  # A
        
    # Step 3: Initialize Texture Source
    texture.Source.Width = 256
    texture.Source.Height = 1
    texture.Source.Depth = 1
    texture.Source.Layers = 1
    texture.Source.Format = "TSF_BGRA8"
    texture.Source.Data = bytes(byte_data)
    
    # Step 4: Create and Bind Texture
    # Create the 256x1 RGBA8 texture from the byte data
    texture.create_2d_texture(256, 1, "RGBA8")
    
    # Bind to shader as 'palette'
    # This is a runtime binding step, not part of the texture object creation itself,
    # but implied by the method's purpose.
    
    # Step 5: Finalize Texture
    texture.UpdateResource()
    texture.PostEditChange()
    
    return texture

def reverse_endianness(value: int) -> int:
    """Reverse the byte order of a 32-bit unsigned integer."""
    return ((value & 0xFF) << 24) | \
           ((value & 0xFF00) << 8) | \
           ((value & 0xFF0000) >> 8) | \
           ((value & 0xFF000000) >> 24)
```

**Parameters:**
- Palette Size: Fixed at 256 entries [S1, unofficial].
- Texture Dimensions: 256x1 [S1, unofficial].
- Format: BGRA8 (internal storage) / RGBA8 (logical) [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Endianness Mismatch:** The endianness reversal and the raw byte copy must be changed together. If the shader reads the texture as sRGB, it relies on the specific byte order (R, G, B, A) resulting from the little-endian interpretation of the reversed uint [S2, unofficial]. Failure to align these results in incorrect color channels.
- **sRGB Interpretation:** The shader expects sRGB data. Ensure the texture format and shader sampling account for this [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the palette size (256). The conversion and copy are linear.
- **Memory Complexity:** $O(N)$ for the byte array. The texture memory is $256 \times 1 \times 4$ bytes = 1 KB.
- **Performance:** Negligible for a single palette. Suitable for frequent updates if the palette changes often.
- **Strengths:** Simple, fast setup for 256-color palettes. Explicit handling of endianness ensures correctness on different hardware architectures.
- **Limitations:** Fixed size (256 colors). Not suitable for high-color-depth or large palettes. Requires careful attention to byte order.
- **When to use:** When using a 256-entry palette for volume rendering with sRGB shaders.
- **When not to use:** For palettes larger than 256 entries, or when high-color fidelity (e.g., 16-bit per channel) is required.

## Background
This method originates from the VOX4U and Voxel2Pixel projects, which deal with voxel data formats and rendering. The specific handling of endianness and byte packing is critical for ensuring that the palette texture is interpreted correctly by the GPU shader, particularly when dealing with sRGB color spaces and little-endian architectures. The method is part of the broader volume texture creation pipeline, building on [Volume Texture Creation](1857-volume-texture-creation.md).

## Sources
[S1] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
[S2] GitHub - BenMcLean/Voxel2Pixel: Graphics library for rendering voxel ..., https://github.com/BenMcLean/Voxel2Pixel, fetched 2026-10-29 (unofficial)
