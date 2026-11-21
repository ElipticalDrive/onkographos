# Attribute Quantization

Also called: CIELAB color quantization, Perceptual quantization

## Overview
Attribute quantization is a lossy compression technique that reduces the precision of voxel attributes (primarily colors and normals) to lower memory footprint while maintaining perceptual quality [S1]. It addresses the problem where detailed scenes contain millions of distinct attribute values with small variations, allowing the reduction of distinct values by leveraging human perception limits [S1]. It differs from [Scalar Quantization](5726-scalar-quantization.md) and [Uniform Quantization](5855-uniform-quantization.md) by specializing the quantization method based on data type (e.g., using perceptually uniform color spaces or geometric mappings for normals) rather than applying a generic linear mapping [S1].

## How it works
**Inputs:** Attribute array containing colors (RGB) and/or normals (3D vectors) [S1].
**Outputs:** Quantized attribute values mapped to a reduced discrete set [S1].

**Data Structures:**
*   **Color:** Mapped to CIELAB space. The number of distinct colors is determined by the bit depth [S1]. Exact field layouts, sizes, and bit packing are Not given in the sources.
*   **Normals:** Mapped to Octahedron Normal Vectors (ONVs) [S1]. Exact field layouts, sizes, and bit packing are Not given in the sources.

**Steps:**
1.  **Color Quantization (Perceptual):**
    1.  Preprocess the attribute array to analyze the distribution of colors [S1].
    2.  Convert RGB colors to the CIELAB color space, which is (locally) perceptually uniform [S1].
    3.  Apply a data-driven clustering algorithm in the CIELAB space to reduce the number of distinct colors [S1].
    4.  Map original colors to the nearest cluster center [S1].
    5.  *Note:* The original method by Xiang relied on clustering in a scaled RGB space, but CIELAB yields higher quality results for the same number of colors [S1].
2.  **Normal Quantization (ONV):**
    1.  Convert 3D normal vectors $\mathbf{n} = (n_x, n_y, n_z)$ to Octahedron Normal Vectors (ONVs) [S1].
    2.  Quantize the ONV coordinates uniformly [S1].
    3.  *Benefit:* ONVs provide higher precision for the same number of bits compared to storing one value per dimension [S1].

**Formulas:**
*   **Color Difference:** Visual comparison uses the Euclidean distance in RGB space for error metrics:
    $$ \Delta E_{RGB} = \sqrt{dR^2 + dG^2 + dB^2} $$
    where $dR$, $dG$, and $dB$ are the differences in the red, green, and blue color channels, respectively [S1].
*   **CIE94 Delta-E:** Used for perceptual quality assessment with parameters $k_L = 1$, $K_1 = 0.045$, $K_2 = 0.015$, and D65 illuminant as reference white [S1].

**Parameters:**
*   **Color Bit Depth:** Typically 12-bit (4096 colors) is used [S1].
*   **Normal Bit Depth:** 12-bit ONVs produce minor errors; 10-bit may suffice for diffuse shading but causes visible banding; 16-bit is preferred for specular reflections [S1].

**Pitfalls:**
*   Color quantization requires preprocessing to analyze colors [S1].
*   Using too few bits for normals (e.g., 10-bit) can cause visible banding on smoothly varying surfaces [S1].
*   Lossless schemes are overly conservative; allowing lossy quantization significantly improves compression rates [S1].

## Subtopics

### [Thresholding and Quantization](5722-thresholding-and-quantization.md)
A lossy compression technique applied to the coefficients of a compact volume model (such as wavelets or tensors) to further reduce data size by neglecting insignificant values. It distinguishes itself from sibling methods like [Quantization with Noise Injection](2090-quantization-with-noise-injection.md) by focusing on coefficient sparsity and precision reduction rather than adding stochastic noise to preserve statistical properties. It allows the user to define the maximum allowed error, making it a controllable trade-off between compression ratio and fidelity.

### [Quantization with Noise Injection](2090-quantization-with-noise-injection.md)
Quantization of continuous anchor attributes to discrete values for entropy coding, using noise injection during training for backpropagation and rounding during inference. The sources do not provide algorithmic details for this variant.

## Operating characteristics
*   **Memory Reduction:** Attribute arrays often exceed the geometry (DAG) memory size [S1]. Quantization combined with palette compression can reduce the attribute memory footprint to a similar order of magnitude as the geometry [S1].
*   **Compression Rates:** Significant compression rates are achieved (e.g., 20-69% reduction in attribute array size depending on the scene) [S1].
*   **Perceptual Quality:** 12-bit colors and normals generally maintain high visual quality. 10-bit normals are acceptable for diffuse shading but show banding [S1].
*   **Non-Linear Scaling:** Combining multiple attributes (colors, normals, reflectance) does not result in a linear increase in memory due to shared compression benefits [S1].
*   **Preprocessing Cost:** Color quantization requires an initial analysis pass over the data [S1].

## Background
Attribute quantization for voxel scenes was proposed to address the large memory footprint of attribute arrays in hierarchical voxel representations (like SVOs) [S1]. The approach builds on the observation that human perception is not flawless, allowing lossy compression of attributes like colors and normals without significant visual degradation [S1]. The specific use of CIELAB for colors improves upon earlier RGB-based clustering methods [S1]. The use of Octahedron Normal Vectors (ONVs) for normals leverages prior work in normal vector quantization [MSS* 10, CDE* 14] to achieve better precision per bit [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
