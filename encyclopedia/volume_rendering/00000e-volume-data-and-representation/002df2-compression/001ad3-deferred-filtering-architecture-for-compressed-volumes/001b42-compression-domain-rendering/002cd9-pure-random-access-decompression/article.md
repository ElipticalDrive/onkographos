# Pure Random-Access Decompression

Also called: Literal compression-domain rendering, On-demand voxel decompression

## Overview
Pure Random-Access Decompression is a compression-domain rendering strategy where volumetric data is transmitted to the GPU in compressed form and decompressed on-demand for each individual voxel access during the rendering process [S1][S2, unofficial]. Unlike methods that decompress a full working set into GPU memory before rendering, this approach never fully decompresses the dataset in GPU memory, thereby better exploiting memory bandwidth during transmission [S1][S2, unofficial]. It is distinguished from [Compression Domain Rendering](960-compression-domain-rendering.md) by its lack of pre-fetching or block-based reconstruction; instead, it relies on immediate, spatially independent decompression for every requested sample [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed volume data (transmitted from CPU to GPU) [S1].
*   **Output:** Uncompressed scalar values for individual voxels as needed by the ray-caster or fragment shader [S1].

**Data Structures:**
The specific compressed data layout depends on the underlying codec (e.g., VQ, Huffman) [S1][S2, unofficial]. The sources do not specify a universal bit-packing layout, only that data is transmitted in compressed form [S1][S2, unofficial].

**Steps:**
1.  **Transmission:** Transmit the compressed volume data from the CPU to the GPU memory [S1].
2.  **On-Demand Access:** When the rendering pipeline requires a voxel value at a specific coordinate, trigger decompression [S1].
3.  **Decompression:** Perform fast, spatially independent decompression for the single requested voxel [S1]. This may involve shader-based decoding or GPGPU programming if hardware support is absent [S1].
4.  **Rendering:** Use the decompressed value for shading, gradient calculation, or transfer function mapping [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast(compressed_volume, origin, direction, step_size, bounds_min, bounds_max):
    t = 0.0
    samples = []
    while step_size > 0:
        pos = [origin[i] + direction[i] * t for i in range(3)]
        if not (bounds_min[0] <= pos[0] <= bounds_max[0] and
                bounds_min[1] <= pos[1] <= bounds_max[1] and
                bounds_min[2] <= pos[2] <= bounds_max[2]):
            break
        value = decompress_voxel(compressed_volume, pos)
        samples.append(value)
        t += step_size
        if value >= 1.0:
            break
    return samples
```
*Note: The sources do not specify the codec internals; `decompress_voxel` is a black-box call that returns the scalar value for the given coordinate [S1].*

**Edge Cases and Pitfalls:**
*   **Performance Cost:** GPU implementations of random-access decompression (e.g., for VQ or HVQ) are costly in performance [S1].
*   **Quality Limitations:** Pure random-access rarely implements multisampling or high-quality shading effectively without the aid of deferred filtering techniques [S1].
*   **Interpolation:** If interpolation is required, it must be implemented via graphics card interpolation features or GPU fragment programming, as the raw decompressed data is point-wise [S2, unofficial].

## Subtopics

### GPU On-Demand Decompression
This variant is used by Vector Quantization (VQ) methods [36-37] and by Wang et al. [48, 67] [S2, unofficial]. It decompresses multiple related blocks or slices on-demand rather than just a single voxel, and employs delayed filtering to exploit data space correlations [S2, unofficial]. This approach improves GPU memory utilization compared to pure single-voxel access but still avoids full pre-decompression [S2, unofficial].

## Operating characteristics
*   **Memory Usage:** Low GPU memory footprint for the volume data itself, as the full dataset is never stored in uncompressed form [S1][S2, unofficial].
*   **Performance:** Decompression is computationally expensive per voxel because it lacks spatial coherence exploitation (like block-based reconstruction) [S1].
*   **Bandwidth:** Better exploitation of memory bandwidth during transmission from CPU to GPU due to compression [S1][S2, unofficial].
*   **Limitations:** Difficult to implement high-quality shading, multisampling, or gradient-based operations without additional techniques like deferred filtering [S1].
*   **When to use:** Suitable when GPU memory is severely constrained and the volume is too large to fit in uncompressed form, or for time-varying visualizations where frame data fits in GPU memory but pre-decompression is too slow [S1].
*   **When not to use:** When high-quality shading, multisampling, or fast rendering is required, as pure random-access is rarely sufficient for these without deferred filtering [S1].

## Background
The concept emerged as an alternative to architectures that require prior decompression of the full working set in GPU memory, such as those using FPGA decoders for wavelet coefficients [WSKW05] or GPU block-based decoders for multiscale tensor reconstruction [SIM∗ 11] [S1]. The latter approach limits the maximum working set size by its uncompressed size, which is problematic for large volumes [S1]. Pure random-access was proposed to overcome this limitation by keeping data compressed during transmission and decompressing only what is needed [S1]. It is often implemented using shaders or GPGPU programming when hardware support (like OpenGL VTC [Cra04, NIH08] or ASTC [NLP∗ 12]) is insufficient for the specific data format [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
