# Gaussian Splat Texture Packing

Also called: Texture Packing

## Overview
Gaussian Splat Texture Packing stores 3D Gaussian Splat attributes into a dense 2D texture to optimize GPU memory bandwidth and cache locality during rasterization [S1, unofficial]. It packs 8 unsigned integers per Gaussian into a `GL_RGBA32UI` texture, arranging 1024 Gaussians per row [S1, unofficial]. This layout differs from SPZ Packed Format by using a fixed-width texture with a specific row-major bit-packing scheme rather than a variable-length or sparse format, and from PNG-based compression by using raw integer textures rather than lossy or lossless image codecs [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A list of $N$ Gaussians, where each Gaussian $i$ has 8 packed integer attributes [S1, unofficial].
**Outputs:** A 2D texture `textureData` of format `GL_RGBA32UI` with dimensions $W \times H$ [S1, unofficial].

**Data Structures:**
*   `packedData`: A flat array of $N \times 8$ unsigned 32-bit integers. Each Gaussian $i$ occupies indices $[i \times 8, i \times 8 + 7]$ [S1, unofficial].
*   `textureData`: A 2D texture buffer [S1, unofficial].
*   Texture Dimensions:
    *   Width $W = 2048$ [S1, unofficial].
    *   Height $H = \lceil N / 1024 \rceil$ [S1, unofficial].

**Layout:**
Each Gaussian $i$ is mapped to two horizontal texels (columns) in the texture [S1, unofficial].
*   Row index: $r = \lfloor i / 1024 \rfloor$ [S1, unofficial].
*   Column index: $c = (i \pmod{1024}) \times 2$ [S1, unofficial].
*   The first `uvec4` of Gaussian $i$ is stored at texture coordinates $(c, r)$ [S1, unofficial].
*   The second `uvec4` of Gaussian $i$ is stored at texture coordinates $(c+1, r)$ [S1, unofficial].

**Parameters:**
*   Texture Internal Format: `GL_RGBA32UI` [S1, unofficial].
*   Texture Type: `GL_RGBA_INTEGER` [S1, unofficial].
*   Data Type: `GL_UNSIGNED_INT` [S1, unofficial].
*   Min/Mag Filter: `GL_NEAREST` [S1, unofficial].
*   Wrap Mode: `GL_CLAMP_TO_EDGE` for both S and T axes [S1, unofficial].

**Steps:**
1.  Initialize `textureData` with dimensions $2048 \times \lceil N / 1024 \rceil$ [S1, unofficial].
2.  For each Gaussian $i$ from $0$ to $N-1$:
    a. Calculate row $r = i / 1024$ [S1, unofficial].
    b. Calculate column $c = (i \% 1024) \times 2$ [S1, unofficial].
    c. Calculate base index for the first texel: $idx1 = (r \times 2048 + c) \times 4$ [S1, unofficial].
    d. Calculate base index for the second texel: $idx2 = (r \times 2048 + c + 1) \times 4$ [S1, unofficial].
    e. Copy `packedData[i*8+0..3]` to `textureData[idx1..idx1+3]` [S1, unofficial].
    f. Copy `packedData[i*8+4..7]` to `textureData[idx2..idx2+3]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_gaussians_to_texture(packed_data: list, splat_count: int) -> list:
    """
    packed_data: List of uint32, length splat_count * 8.
    Returns: List of uint32 representing the RGBA32UI texture data.
    """
    texture_width = 2048
    texture_height = (splat_count + 1023) // 1024  # ceil(splat_count / 1024)
    total_components = texture_width * texture_height * 4
    
    # Initialize texture data with zeros
    texture_data = [0] * total_components
    
    for i in range(splat_count):
        # Calculate row and column
        row = i // 1024
        col = (i % 1024) * 2
        
        # Calculate indices in the flat texture array (RGBA order)
        idx1 = (row * texture_width + col) * 4
        idx2 = (row * texture_width + col + 1) * 4
        
        # Copy first uvec4 (indices 0-3)
        for k in range(4):
            texture_data[idx1 + k] = packed_data[i * 8 + k]
            
        # Copy second uvec4 (indices 4-7)
        for k in range(4):
            texture_data[idx2 + k] = packed_data[i * 8 + 4 + k]
            
    return texture_data
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If $N$ is not a multiple of 1024, the last row contains unused texels [S1, unofficial].
*   **Memory Usage:** The texture size is fixed at $2048 \times \lceil N/1024 \rceil \times 16$ bytes [S1, unofficial].

## Subtopics

### Texture-Based Splat Storage Preprocessing
A variant used in Unreal Engine Niagara rendering, which preprocesses Gaussian splat data into 8 separate `RGBA32F` textures (Position, Scale, Rotation, Color, HarmonicsL1, HarmonicsL2, HarmonicsL31, HarmonicsL32) [S2, unofficial]. Unlike the standard packed format which interleaves data in a single texture, this variant separates attributes into distinct textures for on-demand loading by Niagara [S2, unofficial].
*   **Definition:** Preprocesses parsed Gaussian splat data into 8 separate RGBA32F textures [S2, unofficial].
*   **Steps:** CreateAndSaveTexture function creates a UTexture2D asset. It sets SRGB=false, MipGenSettings=TMGS_NoMipmaps, CompressionNone=true, Filter=TF_Nearest. It initializes the source with ETextureSourceFormat::TSF_RGBA32F, locks mip 0, copies InPixelData (FLinearColor array) via FMemory::Memcpy, unlocks, and updates resources [S2, unofficial].
*   **Parameters:** Texture format is TSF_RGBA32F (32-bit floating point RGBA) [S2, unofficial].
*   **Layout:** FGaussianSplattingTextureData struct fields: PositionTextureData (TArray<FLinearColor>), ScaleTextureData (TArray<FLinearColor>), RotationTextureData (TArray<FLinearColor>), ColorTextureData (TArray<FLinearColor>), harmonicsL1TextureData (TArray<FLinearColor>), harmonicsL2TextureData (TArray<FLinearColor>), harmonicsL31TextureData (TArray<FLinearColor>), harmonicsL32TextureData (TArray<FLinearColor>) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians [S1, unofficial].
*   **Memory Complexity:** The texture requires $2048 \times \lceil N/1024 \rceil \times 16$ bytes [S1, unofficial].
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The method originates from implementations of 3D Gaussian Splatting in OpenGL, specifically the `gsplat-opengl` project, which aims to provide a high-performance, real-time renderer for 3D Gaussians [S1, unofficial]. It is designed to integrate with standard graphics pipelines by leveraging texture-based storage for efficient GPU access [S1, unofficial].

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S2] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
