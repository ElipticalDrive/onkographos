# Gaussian Splat Texture Packing

Also called: 2D Texture Array Packing, RGBA32UI Splat Packing

## Overview
Gaussian Splat Texture Packing is a method for storing 3D Gaussian Splat attributes into a 2D texture array to enable efficient GPU-based rendering and reading. It solves the problem of transferring large volumes of Gaussian data (position, scale, rotation, color, SH coefficients) from CPU memory to the GPU by packing multiple attributes into a single `GL_RGBA32UI` texture. It sets itself apart from sibling methods like [SPZ Packed Format](3926-spz-packed-format.md) by using a fixed-width 2048-pixel texture with a specific row-major layout, and from [PNG-based Gaussian Compression](4834-png-based-gaussian-compression.md) by using uncompressed 32-bit unsigned integer formats for high precision.

## How it works
**Inputs:**
- `packedData`: A flat array of 8-bit or 32-bit integers containing the packed Gaussian attributes. Each Gaussian $i$ occupies 8 consecutive elements (representing 2 `uvec4` values, i.e., 32 bytes or 8 uint32s).
- `splatCount`: The total number of Gaussians.

**Outputs:**
- `splatTexture`: A 2D texture object (`GLuint`) containing the packed data.

**Data Structures:**
- **Texture Format:** `GL_RGBA32UI` (internal format), `GL_RGBA_INTEGER` (type), `GL_UNSIGNED_INT` (data type).
- **Texture Dimensions:**
  - Width: 2048 pixels.
  - Height: $\lceil \text{splatCount} / 1024 \rceil$ pixels.
- **Layout:**
  - Each Gaussian occupies 2 horizontal columns (2 texels).
  - The x-coordinate (column) for Gaussian $i$ is $(i \pmod{1024}) \times 2$.
  - The y-coordinate (row) for Gaussian $i$ is $\lfloor i / 1024 \rfloor$.
  - Each texel stores 4 `uint32` values.
  - Gaussian $i$'s first 4 attributes are stored in texel $(x, y)$.
  - Gaussian $i$'s next 4 attributes are stored in texel $(x+1, y)$.

**Parameters:**
- Texture Filters: `GL_NEAREST` for minification and magnification.
- Wrap Mode: `GL_CLAMP_TO_EDGE` for both S and T axes.

**Steps:**
1. Calculate texture width $W = 2048$.
2. Calculate texture height $H = \lceil \text{splatCount} / 1024 \rceil$.
3. Create a 2D texture with format `GL_RGBA32UI`, dimensions $W \times H$, and type `GL_UNSIGNED_INT`.
4. For each Gaussian $i$ from $0$ to $\text{splatCount} - 1$:
   a. Compute row $r = \lfloor i / 1024 \rfloor$.
   b. Compute column index $c = (i \pmod{1024}) \times 2$.
   c. Calculate the byte offset for the first texel: $\text{idx1} = (r \times W + c) \times 4$.
   d. Calculate the byte offset for the second texel: $\text{idx2} = (r \times W + c + 1) \times 4$.
   e. Copy `packedData[i*8 + 0..3]` to `textureData[idx1..idx1+3]`.
   f. Copy `packedData[i*8 + 4..7]` to `textureData[idx2..idx2+3]`.
5. Upload `textureData` to the GPU texture using `glTexSubImage2D` or `glTexImage2D`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_gaussians_to_texture(packed_data: list[int], splat_count: int) -> tuple[int, int, list[int]]:
    """
    Packs Gaussian data into a 2D texture layout.
    
    Args:
        packed_data: List of integers representing packed attributes. 
                     Each Gaussian has 8 integers (32 bytes).
        splat_count: Number of Gaussians.
        
    Returns:
        texture_width: Width of the texture (2048).
        texture_height: Height of the texture.
        texture_data: Flattened list of integers representing the texture data.
    """
    texture_width = 2048
    texture_height = (splat_count + 1023) // 1024  # ceil(splat_count / 1024)
    
    # Each texel has 4 uint32 values. Total texels = width * height.
    total_texels = texture_width * texture_height
    texture_data = [0] * (total_texels * 4)
    
    for i in range(splat_count):
        # Calculate row and column
        row = i // 1024
        col = (i % 1024) * 2
        
        # Calculate indices in the flattened texture data array
        # Each texel is 4 integers
        idx1 = (row * texture_width + col) * 4
        idx2 = (row * texture_width + col + 1) * 4
        
        # Copy first 4 attributes to first texel
        for k in range(4):
            texture_data[idx1 + k] = packed_data[i * 8 + k]
            
        # Copy next 4 attributes to second texel
        for k in range(4):
            texture_data[idx2 + k] = packed_data[i * 8 + 4 + k]
            
    return texture_width, texture_height, texture_data
```

**Edge Cases and Pitfalls:**
- **Precision:** Using `GL_RGBA32UI` ensures no quantization loss for integer-packed attributes. If floating-point attributes are used, `GL_RGBA32F` should be used instead, but the packing logic remains similar.
- **Boundaries:** The texture height is calculated using ceiling division to ensure all Gaussians fit. Extra texels at the end of the texture are unused and should be ignored during reading.
- **Alignment:** The layout assumes 4-byte alignment for `uint32`. Ensure the `packedData` array is properly aligned.

## Subtopics

### [Packed Splat Reading](6819-packed-splat-reading.md)
Packed Splat Reading is a method to read a single 3D Gaussian Splat from a packed texture representation, decoding its geometric and color properties into a `Gsplat` struct. It distinguishes itself from extended variants by utilizing a single `usampler2DArray` texture and a standard unpacking routine for the packed data.

### [Texture-Based Splat Storage Preprocessing](7101-texture-based-splat-storage-preprocessing.md)
This method preprocesses parsed 3D Gaussian splat data into eight separate RGBA32F textures, one for each attribute channel (Position, Scale, Rotation, Color, and three Spherical Harmonics levels), to enable efficient on-demand loading by the Unreal Engine Niagara rendering system. It sets itself apart from sibling methods by focusing on the creation of uncompressed, high-precision floating-point texture assets rather than byte-level compression or Morton-ordering.

### Adaptive Texture Allocation for 2D Gaussians
A method to assign texture dimensions to 2D Gaussian primitives based on their spatial extent, ensuring uniform texel size in world space and adaptive level-of-detail [S2]. It computes a uniform texel size $T_{size}$ using a hyperparameter $T_{total}$ and the sum of scaled areas of all Gaussians. For each Gaussian $i$, texture width $U_i$ and height $V_i$ are determined by its scaling factors $(s_u)_i$ and $(s_v)_i$:
$$ U_i = \lceil 6 \times (s_u)_i / T_{size} \rceil $$
$$ V_i = \lceil 6 \times (s_v)_i / T_{size} \rceil $$
where $T_{size} = T_{total} / \sum_{i \in N} (36 \times (s_u)_i (s_v)_i)$. $T_{total}$ is a hyperparameter controlling the total number of texels, typically set to $1 \times 10^7$ [S2].

## Operating characteristics
- **Time Complexity:** Packing is $O(N)$ where $N$ is the number of Gaussians. Reading is $O(1)$ per Gaussian.
- **Memory Complexity:** The texture size is $2048 \times \lceil N/1024 \rceil \times 16$ bytes (for `RGBA32UI`). This is approximately $32N$ bytes, which is efficient compared to storing separate arrays for each attribute.
- **Performance:** GPU texture sampling is highly optimized. Using `GL_NEAREST` avoids interpolation overhead.
- **Strengths:** Simple layout, efficient GPU access, high precision.
- **Limitations:** Fixed width limits the number of Gaussians per row. Not suitable for dynamic resizing without reallocation.

## Background
Originating from implementations like `gsplat-opengl` [S1, unofficial], this method is used in real-time 3D Gaussian Splatting renderers to efficiently manage Gaussian data on the GPU. It is a foundational step for rendering pipelines that require fast access to Gaussian attributes.

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S2] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
