# Compression Domain Rendering

Also called: Hardware accelerated texture decoding, On-chip decoding

## Overview
Compression Domain Rendering is a technique that reconstructs volumetric data on the GPU per-fragment by fetching indices from a compressed RGB index texture, using dependent texture fetches to retrieve difference vectors from codebook textures, and adding them to the mean value [S1, unofficial]. It differs from [Stripe-based On-the-fly Decoding](5363-stripe-based-on-the-fly-decoding.md) by relying on hardware-accelerated 3D texture mapping with dependent fetches rather than explicit CPU/GPU block decompression loops, and from [CPU-Based Block Decompression](7197-cpu-based-block-decompression.md) by performing the reconstruction entirely within the GPU shader pipeline without transferring decompressed blocks to main memory [S1, unofficial].

## How it works
**Inputs:**
*   Compressed volumetric data represented as a hierarchical quantization scheme, generating an RGB index set of 1/4 the original size in every dimension [S1, unofficial].
*   Codebook textures $C_1$ and $C_2$ storing difference vectors [S1, unofficial].
*   Address texture $A$ storing relative addresses for codeword components [S1, unofficial].
*   Color table (1D dependent texture map) [S1, unofficial].

**Outputs:**
*   Color and opacity values drawn to the framebuffer [S1, unofficial].

**Data Structures:**
*   **Index Texture $I$**: 3D texture map. Each texel is RGB.
    *   $R$: Mean value of the block [S1, unofficial].
    *   $G$: Index for difference vectors in $C_1$ [S1, unofficial].
    *   $B$: Index for difference vectors in $C_2$ [S1, unofficial].
*   **Codebook Textures $C_1, C_2$**: 2D textures storing difference vectors. $C_1$ requires 64 distinct addresses; $C_2$ requires 8 distinct addresses [S1, unofficial].
*   **Address Texture $A$**: 3D texture, same size as the encoded texel region (e.g., $4^3$ in the example).
    *   $R$: Stores 64 different addresses to index $C_1$ [S1, unofficial].
    *   $G$: Stores 8 different addresses to index $C_2$. Blocks of $2^3$ adjacent texels are assigned the same address to yield 8 different values [S1, unofficial].
    *   Interpolation: Nearest neighbor [S1, unofficial].

**Steps:**
1.  **Slicing**: Render the 3D index texture sliced in back-to-front or front-to-back order via planes orthogonal to the viewing direction [S1, unofficial]. Care must be taken to perform slicing with as many slices as necessary to render the original data, as each texel of the index texture expands to a $4^3$ block during rendering [S1, unofficial].
2.  **Index Fetch**: Fetch the respective texture sample from the index texture $I$ [S1, unofficial].
3.  **Address Calculation**: Use the address texture $A$ to determine the relative address ($t$ texture coordinate) for each fragment within a codeword [S1, unofficial]. The $G$- and $B$-components of $I$ together with the $R$- and $G$-components of $A$ uniquely determine the addresses into $C_1$ and $C_2$ [S1, unofficial].
4.  **Difference Vector Fetch**: Index the difference vectors via the $G$- and $B$-components of the fetched sample from $I$, and fetch them from the dependent textures $C_1$ and $C_2$ [S1, unofficial].
5.  **Reconstruction**: Add the $R$-component of $I$ (mean value) to the fetched difference information [S1, unofficial].
6.  **Shading**: Map the final scalar value to color and opacity via a 1D dependent texture map (color table) [S1, unofficial].
7.  **Rasterization**: Draw the color and opacity to the framebuffer [S1, unofficial].

**Parameters:**
*   Texture coordinate wrap mode for the index texture: `GL_REPEAT` [S1, unofficial].
*   Address texture $A$ interpolation: Nearest neighbor [S1, unofficial].
*   Address Texture $A$ repetition: The address texture is periodically repeated over the domain to yield correct addresses at every fragment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader for Compression Domain Rendering
# Inputs:
#   I: 3D Index Texture (RGB)
#   C1: 2D Codebook Texture for difference vectors (64 entries)
#   C2: 2D Codebook Texture for difference vectors (8 entries)
#   A: 3D Address Texture (RGB)
#   ColorTable: 1D Texture (Scalar -> RGBA)
#   ViewDirection: Vector defining the slicing planes
#   SliceIndex: Integer index of the current slice being rendered

def render_fragment(I, C1, C2, A, ColorTable, ViewDirection, SliceIndex):
    # 1. Calculate texture coordinates for the current slice
    # The slice is orthogonal to ViewDirection.
    # Coordinates range from 0 to Nx/4, Ny/4, Nz/4 with GL_REPEAT
    tex_coords = calculate_slice_coordinates(SliceIndex, ViewDirection)
    
    # 2. Fetch index from I
    # Wrap mode is GL_REPEAT
    index_texel = sample_texture(I, tex_coords)
    mean_value = index_texel.r
    idx_c1 = index_texel.g
    idx_c2 = index_texel.b
    
    # 3. Fetch address from A
    # Interpolation is Nearest Neighbor
    addr_texel = sample_texture(A, tex_coords)
    
    # 4. Determine s coordinates for dependent fetches
    # G and B of I determine the base index (s) for C1 and C2
    # R and G of A determine the relative address (t) within the codeword
    s_c1 = idx_c1
    t_c1 = addr_texel.r
    
    s_c2 = idx_c2
    t_c2 = addr_texel.g
    
    # 5. Fetch difference vectors from dependent textures
    diff_c1 = sample_texture(C1, vec2(s_c1, t_c1))
    diff_c2 = sample_texture(C2, vec2(s_c2, t_c2))
    
    # 6. Reconstruct scalar value
    # Add mean value to difference information
    # Note: The exact combination of diff_c1 and diff_c2 depends on the 
    # specific hierarchical quantization scheme. Based on the text, 
    # they are fetched and added to the mean.
    reconstructed_value = mean_value + diff_c1 + diff_c2
    
    # 7. Map to color and opacity
    color_rgba = sample_texture(ColorTable, vec1(reconstructed_value))
    
    # 8. Draw to framebuffer
    return color_rgba
```

**Edge Cases and Pitfalls:**
*   **Slicing Granularity**: Because each texel of the index texture expands to a $4^3$ block during rendering, care must be taken to perform the slicing with as many slices as necessary to render the original data [S1, unofficial].
*   **Memory Efficiency**: The address texture $A$ is not enlarged to the size of the original dataset to avoid losing memory savings; instead, it is periodically repeated over the domain using `GL_REPEAT` [S1, unofficial].

## Subtopics

### [GPU Decoder with Texture Mapping](1050-gpu-decoder-with-texture-mapping.md)
This method reconstructs volumetric data values on the GPU by mapping output coordinates to texture memory indices, retrieving codebook vectors via dependent texture fetches, and combining them with mean values. It differs from CPU-based decoding by minimizing data transfer between main memory and GPU memory, leveraging the GPU's texture hardware for parallel reconstruction of compressed vector-quantized data.

### [Pure Random-Access Decompression](5361-pure-random-access-decompression.md)
Pure Random-Access Decompression is a compression-domain rendering strategy where volumetric data is transmitted to the GPU in compressed form and decompressed on-demand for each individual voxel access during the rendering process. Unlike methods that decompress a full working set into GPU memory before rendering, this approach never fully decompresses the dataset in GPU memory, thereby better exploiting memory bandwidth during transmission. It is distinguished from [Compression Domain Rendering](960-compression-domain-rendering.md) by its lack of pre-fetching or block-based reconstruction; instead, it relies on immediate, spatially independent decompression for every requested sample.

## Operating characteristics
*   **Performance**: The simultaneous decoding and rendering of compressed data sets results in a loss in performance of about a factor of 2-3 compared to the rendering of uncompressed data [S1, unofficial].
*   **Memory**: The index set is 1/4 the original size in every dimension [S1, unofficial]. The address texture is periodically repeated to avoid storing a full-sized address map [S1, unofficial].
*   **Strengths**: Hardware accelerated 3D texture mapping allows for parallel reconstruction [S1, unofficial]. Minimizes data transfer between main memory and GPU memory [S1, unofficial].
*   **Limitations**: Performance loss compared to uncompressed rendering [S1, unofficial].

## Background
Compression Domain Rendering was introduced in a 2003 paper by researchers at the Technical University of Munich (TUM) [S1, unofficial]. It utilizes a hierarchical quantization scheme and hardware-accelerated texture mapping to perform on-chip decoding [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
