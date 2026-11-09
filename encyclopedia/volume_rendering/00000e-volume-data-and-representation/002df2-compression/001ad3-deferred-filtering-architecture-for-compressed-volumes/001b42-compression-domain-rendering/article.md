# Compression Domain Rendering

Also called: Hardware accelerated texture decoding, On-chip decoding, Render from Compressed Data

## Overview
Compression Domain Rendering is a technique that reconstructs volumetric data on the GPU per-fragment by fetching indices from a compressed texture and using dependent texture fetches to retrieve difference vectors from codebook textures, adding them to a mean value [S1, unofficial]. It solves the problem of high storage, computation, and transmission overhead associated with massive volume datasets by rendering directly from compressed scalar data without decompressing the entire dataset into GPU memory [S2, unofficial]. It differs from [Stripe-based On-the-fly Decoding](5363-stripe-based-on-the-fly-decoding.md) and [CPU-Based Block Decompression](7197-cpu-based-block-decompression.md) by performing the reconstruction entirely within the GPU's texture hardware pipeline via dependent fetches, minimizing data transfer between main memory and GPU memory [S1, unofficial].

## How it works
**Inputs:**
*   Compressed volumetric data, typically encoded using Vector Quantization (VQ) [S1, unofficial].
*   An RGB index texture $I$ containing the compressed indices [S1, unofficial].
*   Two dependent codebook textures $C_1$ and $C_2$ containing difference vectors [S1, unofficial].
*   An address texture $A$ storing relative addresses within codewords [S1, unofficial].
*   A 1D color/opacity table texture [S1, unofficial].

**Outputs:**
*   Color and opacity values drawn to the framebuffer [S1, unofficial].

**Data Structures:**
*   **Index Texture $I$ (RGB):**
    *   Size: $1/4$ the original data size in every dimension (e.g., for a $4^3$ block, the index texture is smaller) [S1, unofficial].
    *   R-component: Mean value of the block [S1, unofficial].
    *   G-component: Index into $C_1$ [S1, unofficial].
    *   B-component: Index into $C_2$ [S1, unofficial].
*   **Codebook Textures $C_1, C_2$:**
    *   $C_1$: Stores difference vectors indexed by the G-component of $I$ [S1, unofficial].
    *   $C_2$: Stores difference vectors indexed by the B-component of $I$ [S1, unofficial].
*   **Address Texture $A$:**
    *   Size: Same as the encoded texel region (e.g., $4^3$ in the example) [S1, unofficial].
    *   R-component: Stores 64 different addresses to index $C_1$ [S1, unofficial].
    *   G-component: Stores 8 different addresses to index $C_2$ [S1, unofficial].
    *   Layout: Blocks of $2^3$ adjacent texels in the original data space are assigned the same address in the G-component of $A$ to yield 8 different values for $C_2$ [S1, unofficial].

**Parameters:**
*   Texture coordinate wrap mode for index texture: `GL_REPEAT` [S1, unofficial].
*   Address Texture $A$ interpolation: Nearest neighbor [S1, unofficial].

**Steps:**
1.  **Slicing:** Render the 3D index texture sliced in back-to-front or front-to-back order via planes orthogonal to the viewing direction [S1, unofficial]. The number of slices must be sufficient to render the original data, as each texel of the index texture expands to a $4^3$ block during rendering [S1, unofficial].
2.  **Fragment Shader Execution (Per Fragment):**
    a.  Fetch the texture sample from the index texture $I$ using the fragment's texture coordinates [S1, unofficial].
    b.  Extract the G- and B-components of the fetched sample from $I$ [S1, unofficial].
    c.  Use the G-component to determine the $s$ texture coordinate for $C_1$ and the B-component for $C_2$ [S1, unofficial].
    d.  Fetch the relative address (t texture coordinate) for the current fragment within the codeword from the address texture $A$ using nearest neighbor interpolation [S1, unofficial].
    e.  Fetch the difference vector from $C_1$ using the $s$ coordinate from $I$'s G-component and the $t$ coordinate from $A$'s R-component [S1, unofficial].
    f.  Fetch the difference vector from $C_2$ using the $s$ coordinate from $I$'s B-component and the $t$ coordinate from $A$'s G-component [S1, unofficial].
    g.  Add the R-component of $I$ (mean value) to the fetched difference information from $C_1$ and $C_2$ to reconstruct the scalar value [S1, unofficial].
    h.  Map the final scalar value to color and opacity via a 1D dependent texture map (color table) [S1, unofficial].
    i.  Draw the color and opacity to the framebuffer [S1, unofficial].
3.  **Addressing Logic:** The G- and B-components of $I$ together with the R- and G-components of $A$ uniquely determine the addresses into $C_1$ and $C_2$ respectively [S1, unofficial]. The address texture $A$ is periodically repeated over the domain (via `GL_REPEAT` on $I$) to yield correct addresses at every fragment without requiring an enlarged texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for Compression Domain Rendering
# Inputs:
#   I: Index Texture (RGB)
#   C1: Codebook Texture 1
#   C2: Codebook Texture 2
#   A: Address Texture
#   ColorTable: 1D Texture

def render_fragment(tex_coords):
    # 1. Fetch index from I
    # I is sampled with tex_coords. Wrap mode is GL_REPEAT.
    index_sample = sample_texture(I, tex_coords)
    
    # 2. Extract components
    mean_value = index_sample.r
    index_c1_s = index_sample.g
    index_c2_s = index_sample.b
    
    # 3. Fetch relative address from A
    # A is sampled with tex_coords. Interpolation is Nearest Neighbor.
    # A's content is repeated over the domain.
    address_sample = sample_texture(A, tex_coords, interpolation=NEAREST)
    
    # 4. Determine coordinates for C1
    # R-component of A provides the t-coordinate (relative address) for C1
    c1_t = address_sample.r
    c1_sample = sample_texture(C1, vec2(index_c1_s, c1_t))
    
    # 5. Determine coordinates for C2
    # G-component of A provides the t-coordinate (relative address) for C2
    # Blocks of 2^3 adjacent texels share the same G-address in A
    c2_t = address_sample.g
    c2_sample = sample_texture(C2, vec2(index_c2_s, c2_t))
    
    # 6. Reconstruct scalar value
    # Add mean value to difference vectors
    scalar_value = mean_value + c1_sample + c2_sample
    
    # 7. Map to color and opacity
    color_opacity = sample_texture(ColorTable, scalar_value)
    
    # 8. Output to framebuffer
    return color_opacity
```

**Edge Cases and Pitfalls:**
*   **Slice Count:** Care must be taken to perform slicing with as many slices as necessary to render the original data, because each texel of the index texture expands to a $4^3$ block during rendering [S1, unofficial].
*   **Performance Loss:** Simultaneous decoding and rendering may result in a performance loss of about a factor of 2-3 compared to rendering uncompressed data [S1, unofficial].
*   **Address Texture Size:** The address texture must be large enough to cover one single block (e.g., $4^3$) but is mapped via repetition to cover the whole domain, avoiding memory overhead of an enlarged texture [S1, unofficial].

## Subtopics

### [GPU Decoder with Texture Mapping](1050-gpu-decoder-with-texture-mapping.md)
This method reconstructs volumetric data values on the GPU by mapping output coordinates to texture memory indices, retrieving codebook vectors via dependent texture fetches, and combining them with mean values. It differs from CPU-based decoding by minimizing data transfer between main memory and GPU memory, leveraging the GPU's texture hardware for parallel reconstruction of compressed vector-quantized data.

### [Pure Random-Access Decompression](5361-pure-random-access-decompression.md)
Pure Random-Access Decompression is a compression-domain rendering strategy where volumetric data is transmitted to the GPU in compressed form and decompressed on-demand for each individual voxel access during the rendering process. Unlike methods that decompress a full working set into GPU memory before rendering, this approach never fully decompresses the dataset in GPU memory, thereby better exploiting memory bandwidth during transmission. It is distinguished from [Compression Domain Rendering](960-compression-domain-rendering.md) by its lack of pre-fetching or block-based reconstruction; instead, it relies on immediate, spatially independent decompression for every requested sample.

## Operating characteristics
*   **Time Complexity:** The method performs per-fragment reconstruction, which incurs a performance loss of about a factor of 2-3 compared to rendering uncompressed data [S1, unofficial].
*   **Memory Complexity:** Reduces storage and transmission overhead by using an index texture of $1/4$ the original size in every dimension [S1, unofficial]. The address texture is small (one block size) and repeated, avoiding memory overhead of an enlarged texture [S1, unofficial].
*   **Strengths:** Reduces storage, computation, and transmission overhead of large volume data [S2, unofficial]. Minimizes data transfer between main memory and GPU memory [S1, unofficial].
*   **Limitations:** Performance loss compared to uncompressed rendering [S1, unofficial]. Requires careful slicing to handle the expansion of index texels to blocks [S1, unofficial].
*   **When to Use:** When rendering massive datasets where storage and transmission bandwidth are bottlenecks, and hardware-accelerated texture mapping is available [S1, unofficial][S2, unofficial].
*   **When Not to Use:** When maximum rendering speed is critical and uncompressed data fits in memory [S1, unofficial].

## Background
Compression Domain Rendering was proposed to address the lack of attenuation and occlusion in frequency domain rendering while achieving high compression ratios and fast rendering speed [S2, unofficial]. Chiueh et al. applied 3D Hartley transform to extend JPEG for subcubes, performing frequency domain rendering on subcubes before compositing sub-images in the spatial domain [S2, unofficial]. Ning and Hesselink applied vector quantization in the spatial domain to compress volume and directly rendered quantized blocks [S2, unofficial]. Fowler and Yagel combined differential pulse-code modulation and Huffman coding for lossless compression [S2, unofficial]. Yeo and Liu applied discrete cosine transform based compression on overlapping blocks [S2, unofficial]. The specific hardware-accelerated texture decoding method described in [S1, unofficial] utilizes RGB index textures and dependent texture fetches to reconstruct VQ-encoded data on the GPU.

## Sources
[S1] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
