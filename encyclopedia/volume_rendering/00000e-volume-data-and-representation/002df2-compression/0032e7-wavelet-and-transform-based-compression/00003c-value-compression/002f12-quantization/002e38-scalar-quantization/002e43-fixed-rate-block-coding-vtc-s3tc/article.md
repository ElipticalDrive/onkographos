# Fixed-rate Block-coding (VTC/S3TC)

Also called: 3D S3TC, S3TC, VTC (Volume Texture Compression), VTC 3D texture compression formats

## Overview
Fixed-rate block-coding, specifically Volume Texture Compression (VTC), is a hardware-accelerated compression method that extends the 2D S3TC algorithm to 3D volumes by retiling in the Z direction [S3]. It solves the problem of storing large volumetric datasets in limited GPU memory by approximating voxel values through linear interpolation from a small set of representative values, enabling real-time rendering without decompressing the entire volume into RAM [S1, S2]. Unlike general-purpose codecs, VTC is a fixed-rate format with specific block sizes (4x4x1, 4x4x2, or 4x4x4) and is distinct from its sibling [S3TC](52-s3tc.md) by its native support for 3D textures [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume of $w \times h \times d$ voxels.
*   **Output:** A compressed byte stream of size $\lceil w/4 \rceil \times \lceil h/4 \rceil \times d \times \text{blocksize}$ bytes [S3].

**Data Structures and Layout:**
The volume is divided into blocks. The block dimensions depend on the image depth $d$ [S3]:
*   If $d \ge 4$: Blocks are $4 \times 4 \times 4$ texels.
*   If $d = 1$ or $2$: Blocks are $4 \times 4 \times 1$ or $4 \times 4 \times 2$ texels, respectively [S3].

The storage layout maps these 3D blocks to a linear byte array [S3]:
*   **Block Size:** 8 bytes for DXT1-analogous formats, 16 bytes for DXT3/DXT5-analogous formats [S3].
*   **4x4x1 Blocks:** Stored identically to a standard 4x4 S3TC block [S3].
*   **4x4x2 and 4x4x4 Blocks:** Stored as two or four consecutive 4x4 S3TC blocks, one for each Z-layer within the block [S3]. For example, a 4x4x2 DXT1 block is 16 bytes: the first 8 bytes encode texels $(0,0,0)$ to $(3,3,0)$, and the second 8 bytes encode $(0,0,1)$ to $(3,3,1)$ [S3].

**Decoding Formula and Steps:**
Decoding is performed on-the-fly by hardware [S1, S2]. For a given voxel coordinate $(x,y,z)$, the decoder identifies the containing block and reconstructs the value [S1, S2]:

1.  **Calculate Block Offset:** Determine the byte offset of the block containing texel $(x,y,z)$ in the compressed stream [S3]:
    $$ \text{offset} = \text{blocksize} \times \min(d,4) \times \left( \lfloor x/4 \rfloor + \lceil w/4 \rceil \times \left( \lfloor y/4 \rfloor + \lceil h/4 \rceil \times \lfloor z/4 \rfloor \right) \right) $$
    where $w, h, d$ are the volume dimensions and $\text{blocksize}$ is 8 or 16 [S3].

2.  **Select Interpolation Mode:** The block header contains metadata that selects one of several linear interpolation modes from a lookup table [S1, S2].

3.  **Retrieve Representative Values:** The block contains a precomputed subset of representative values (e.g., endpoints and intermediate weights) [S1, S2].

4.  **Reconstruct Voxel Value:** Apply the selected interpolation formula over the representative values to approximate the voxel value [S1, S2]. This interpolation is performed independently for each block, exploiting spatial coherence in small blocks [S1, S2].

**Parameters:**
*   **Error Tolerance:** In the underlying Block Truncation Coding (BTC) model, an error tolerance (threshold level) determines truncation, which can vary per block [S2]. VTC adapts this for hardware acceleration [S2].

**Edge Cases and Pitfalls:**
*   **Boundary Alignment:** Sub-image updates (`CompressedTexSubImage2D`/`3D`) require width, height, depth, and offsets to be multiples of 4, or equal to the texture dimensions [S3].
*   **Borders:** S3TC and VTC do not support borders; `CompressedTexImage2D`/`3D` generates `INVALID_OPERATION` if `border != 0` [S3].
*   **Format Mismatch:** `CompressedTexImage1DARB` generates `INVALID_ENUM` if the internal format is an S3TC/VTC format, as these formats only support 2D (S3TC) or 3D (VTC) images [S3].

## Operating characteristics
*   **Compression Ratio:** Fixed. 4:1 for DXT1-analogous (8 bytes per 16 texels) and 8:1 for DXT3/DXT5-analogous (16 bytes per 16 texels) [S3].
*   **Performance:** Decoding is hardware-accelerated and fast, allowing real-time rendering without decompressing the entire volume into system memory [S1, S2].
*   **Strengths:** Low memory footprint, random access to any voxel via on-the-fly decoding, and efficient use of GPU texture units [S1, S2].
*   **Limitations:** Lossy compression; quality depends on the linear interpolation approximation [S1, S2]. Limited to specific block sizes and formats [S3].
*   **When to use:** Real-time direct volume rendering on hardware supporting VTC (e.g., NVIDIA GPUs with `GL_NV_texture_compression_vtc`) [S1, S3].
*   **When not to use:** When lossless compression is required, or when the hardware does not support VTC [S1, S3].

## Background
VTC was introduced by NVIDIA as the OpenGL extension `GL_NV_texture_compression_vtc` in April 2004 [S3]. It is an adaptation of Block Truncation Coding (BTC) that supports hardware-accelerated random access, specifically extending the 2D S3TC format to 3D volumes [S2]. It is part of the family of hardware-supported fixed-rate block-coding techniques, alongside S3TC and ASTC [S1, S2].

## Sources

[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] OpenGL Extension GL_NV_texture_compression_vtc, https://people.freedesktop.org/~marcheu/extensions/NV/texture_compression_vtc.html, fetched 2026-10-29
