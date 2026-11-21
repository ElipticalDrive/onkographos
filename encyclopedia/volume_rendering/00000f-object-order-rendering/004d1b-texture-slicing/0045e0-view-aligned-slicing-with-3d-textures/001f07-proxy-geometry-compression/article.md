# Proxy Geometry Compression

Also called: Retained-Mode Rendering with Proxy Geometry Compression, Retained-mode scheme with proxy geometry compression

## Overview
Proxy Geometry Compression is a technique for storing volume data and proxy geometry locally in graphics hardware to avoid CPU-GPU bandwidth bottlenecks associated with immediate mode rendering [S1]. It sets itself apart from immediate-mode splatting by pre-loading compressed proxy geometry templates into GPU memory, leveraging the regularity of rectilinear volumes to achieve high compression ratios (e.g., 171:1) [S1]. This allows the entire volume representation to reside in local graphics memory, enabling interactive rendering speeds that are significantly faster than sending each splat separately [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D rectilinear volume dataset and a traversal order (X, Y, or Z axis) [S1].
*   **Output:** Compressed 32-bit vertex data stored in GPU memory, and decompressed texture coordinates for splatting [S1].

**Data Structures:**
The method uses three proxy geometry templates, one for each of the three possible traversal orders (X, Y, Z) [S1].
*   **Vertex Template:** Represents a single slice of the volume [S1]. For a $256^3$ volume, this requires 768k vertices instead of 192M [S1].
*   **Packed Vertex Format:** Each vertex is packed into a 32-bit integer [S1].
    *   **Volume Texture Coordinates ($t_x, t_y$):** Stored as integers [S1].
        *   $t_x = m_x \times 256 + i_x$
        *   $t_y = m_y \times 256 + i_y$
        *   $m_x, m_y$ (Major): 7 bits each [S1]. Supports slices up to $2^{15} \times 2^{15}$ voxels [S1].
        *   $i_x, i_y$ (Minor): 8 bits each [S1].
    *   **Gaussian Texture Coordinates ($f_x, f_y$):** 1 bit each [S1]. Values are strictly 0 or 1 [S1].
    *   **Total:** $7+8+7+8+1+1 = 32$ bits [S1].
*   **Index Buffer:** Uses 2-byte indices (unsigned short) to keep patch sizes under 65,535 vertices, reducing index buffer memory usage [S1].

**Steps:**
1.  **Pre-computation (CPU):**
    *   Omit vertex positions because they can be calculated from volume texture coordinates [S1].
    *   Maintain three proxy geometry templates for the three traversal orders [S1].
    *   Pack the vertex attributes into 32-bit integers using the formula above [S1].
    *   Upload the packed vertex data and index buffers to GPU memory [S1].
2.  **Rendering (GPU Vertex Shader):**
    *   For each vertex in the proxy geometry template, retrieve the packed 32-bit integer [S1].
    *   Decompress the integer to extract $m_x, m_y, i_x, i_y, f_x, f_y$ [S1].
    *   Use a lookup table (stored in constant registers) to efficiently extract these components [S1].
    *   Recover the full volume texture coordinates: $t_x = m_x \times 256 + i_x$ and $t_y = m_y \times 256 + i_y$ [S1].
    *   Recover the Gaussian texture coordinates $f_x, f_y$ (0 or 1) [S1].
    *   Pass $t_x, t_y, f_x, f_y$ to the fragment shader [S1].
3.  **Fragment Shader:**
    *   Use the passed texture coordinates to sample the volume texture and the 2D Gaussian texture for EWA splatting [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_vertex(mx, my, ix, iy, fx, fy):
    """Pack vertex attributes into a 32-bit integer."""
    return (mx << 25) | (my << 18) | (ix << 10) | (iy << 2) | (fy << 1) | fx

def decompress_vertex(packed_int):
    """Decompress a 32-bit integer to recover texture coordinates."""
    fx = (packed_int >> 0) & 1
    fy = (packed_int >> 1) & 1
    iy = (packed_int >> 2) & 0xFF
    ix = (packed_int >> 10) & 0xFF
    my = (packed_int >> 18) & 0x7F
    mx = (packed_int >> 25) & 0x7F
    tx = mx * 256 + ix
    ty = my * 256 + iy
    return tx, ty, fx, fy

def render_volume(compressed_vertices, indices):
    """Simulate vertex shader processing for each vertex in the index buffer."""
    for idx in indices:
        packed = compressed_vertices[idx]
        tx, ty, fx, fy = decompress_vertex(packed)
        # Pass tx, ty, fx, fy to fragment shader for splatting [S1]
```

**Parameters:**
*   **Slice Size:** The method supports slices up to $2^{15} \times 2^{15}$ voxels due to the 7-bit major coordinate [S1].
*   **Patch Size:** Index buffers are sized to contain no more than 65,535 vertices to allow 2-byte indexing [S1].

**Edge Cases and Pitfalls:**
*   **Voxel Count:** The compression assumes rectilinear/regular volumes [S1].
*   **Memory Usage:** While vertex memory is compressed, index buffers can still be large [S1]. For most datasets, only 5-20% of voxels are non-transparent, limiting index buffer size to ~115.2M bytes in typical cases [S1].
*   **Precision:** The 7-bit major coordinate limits the maximum slice dimension [S1]. If a volume exceeds $2^{15}$ in a dimension, this compression scheme cannot represent it directly without modification [S1].

## Operating characteristics
*   **Compression Ratio:** 171:1 for a $256^3$ volume dataset (12M bytes vs 2048M bytes for vertices) [S1].
*   **Vertex Count Reduction:** Reduces vertex count from 192M to 768k for a $256^3$ volume by sharing templates across slices [S1].
*   **Performance:** Significant improvement over immediate mode rendering [S1].
    *   Bonsai (274,866 splats): 7.53 fps (Retained) vs 0.53 fps (Immediate) [S1].
    *   Engine (247,577 splats): 10.28 fps (Retained) vs 1.40 fps (Immediate) [S1].
    *   Checkerboard (786,432 splats): 6.88 fps (Retained) vs 0.47 fps (Immediate) [S1].
    *   Lobster (555,976 splats): 10.60 fps (Retained) vs 1.19 fps (Immediate) [S1].
    *   UNC Head (2,955,242 splats): 2.86 fps (Retained) vs 0.12 fps (Immediate) [S1].
*   **Strengths:** Eliminates CPU-GPU bandwidth bottleneck; fits large volumes in local GPU memory; fast decompression in vertex shader [S1].
*   **Limitations:** Requires regular/rectilinear volumes; limited slice size ($2^{15}$); index buffer memory can still be significant for very large transparent-heavy volumes [S1].
*   **When to use:** Interactive volume rendering on hardware with limited CPU-GPU bandwidth and sufficient local GPU memory [S1].
*   **When not to use:** Irregular grids; extremely large slices exceeding $2^{15}$; scenes where index buffer memory exceeds available GPU memory [S1].

## Background
Originated in the paper "Hardware-Accelerated Adaptive EWA Volume Splatting" by Wei Chen et al. (2004) [S1]. The method was developed to address the memory bandwidth bottleneck of immediate-mode splatting, which sends each quad separately to the pipeline [S1]. It builds on [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md) by adding compression to the proxy geometry representation [S1].

## Sources
[S1] Hardware-Accelerated Adaptive EWA Volume Splatting, https://publications.ri.cmu.edu/storage/publications/pub_files/pub4/chen_wei_2004_1/chen_wei_2004_1.pdf, fetched 2026-10-27
