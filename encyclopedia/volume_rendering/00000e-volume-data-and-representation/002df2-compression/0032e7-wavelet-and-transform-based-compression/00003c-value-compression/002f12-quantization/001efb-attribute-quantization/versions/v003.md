# Attribute Quantization

Also called: CIELAB color quantization, Perceptual quantization

## Overview
Attribute Quantization is a lossy compression technique that reduces the precision of voxel attributes, such as colors and normals, to lower memory footprint while maintaining perceptual quality [S1]. It solves the problem of excessive memory usage in attribute arrays by leveraging human perception limits to reduce the number of distinct values [S1]. It differs from [Scalar Quantization](5726-scalar-quantization.md) and [Uniform Quantization](5855-uniform-quantization.md) by specializing the quantization method based on the data type (e.g., using CIELAB for colors and Octahedron Normal Vectors for normals) to achieve higher precision and better compression ratios than generic uniform approaches [S1].

## How it works
**Inputs:** A set of continuous attribute values (colors in RGB, normals in 3D vectors).
**Outputs:** Quantized discrete attribute values suitable for palette-based compression or entropy coding.

**Data Structures:**
*   **Color:** A quantized index into a perceptually uniform codebook (e.g., CIELAB space).
*   **Normal:** A 2D coordinate on an octahedron surface, stored as fixed-point integers.

**Steps:**

1.  **Color Quantization (CIELAB):**
    *   Convert input RGB colors to the CIELAB color space. This space is locally perceptually uniform, ensuring that Euclidean distance correlates with perceived difference [S1].
    *   Perform data-driven clustering (e.g., k-means) on the CIELAB values to generate a codebook of representative colors. The user specifies the number of colors, typically 12-bit (4096 colors) [S1].
    *   Replace each voxel's color with the index of the nearest codebook entry.
    *   *Note:* This method requires preprocessing to analyze the colors in the scene [S1].

2.  **Normal Quantization (Octahedron Normal Vectors - ONVs):**
    *   For each normal vector $\mathbf{n} = (n_x, n_y, n_z)$, map it to the surface of a regular octahedron.
    *   Let $s = |n_x| + |n_y| + |n_z|$. If $s = 0$, the normal is undefined (handle as a default). Otherwise, compute the projected coordinates:
        $$ u = \frac{n_x}{s}, \quad v = \frac{n_y}{s} $$
    *   The vector $(u, v)$ lies within the unit square $[-1, 1] \times [-1, 1]$.
    *   Quantize $u$ and $v$ to fixed-point integers with the desired bit depth (e.g., 10-bit or 12-bit).
    *   Store the pair $(u, v)$. This yields higher precision for the same number of bits compared to storing one value per dimension [S1].

**Formulas:**
*   **CIE94 Delta-E:** Used to assess perceptual quality. Parameters: $k_L = 1$, $K_1 = 0.045$, $K_2 = 0.015$, with D65 illuminant as reference white [S1].
*   **Color Difference Magnitude:** $\sqrt{dR^2 + dG^2 + dB^2}$, used for visual comparison [S1].

**Parameters:**
*   **Color Bit Depth:** Typically 12-bit (4096 colors) [S1].
*   **Normal Bit Depth:** 10-bit or 12-bit. 10-bit produces minor banding on smooth surfaces but is barely perceivable for diffuse shading; 12-bit is preferred for specular reflections [S1].

**Edge Cases and Pitfalls:**
*   **Preprocessing:** Color quantization requires a preprocessing step to analyze the scene's colors and build the codebook [S1].
*   **Zero Normals:** Handle zero-length normal vectors explicitly during ONV mapping [G].
*   **Specular Artifacts:** Using low bit depths (e.g., 10-bit) for normals may cause visible banding in specular highlights, even if diffuse shading appears fine [S1].

## Subtopics

### [Quantization with Noise Injection](2090-quantization-with-noise-injection.md)
Enables gradient flow during training by simulating quantization error via noise addition. Choose when differentiable training is required.

### [Thresholding and Quantization](5722-thresholding-and-quantization.md)
Reduces model coefficients by truncation or rounding based on an error bound. Choose when reducing hierarchical transform coefficients is the primary goal.

## Operating characteristics
*   **Memory:** Significantly reduces the memory footprint of the attribute array, potentially bringing it to the same order of magnitude as the geometry (DAG) [S1].
*   **Compression Rate:** Combining quantization with palette compression yields high compression rates (e.g., 20-69% reduction depending on the scene) [S1].
*   **Performance:** The preprocessing step for color clustering adds initial computational cost, but encoding is fast [S1].
*   **Quality:** Maintains high visual quality as assessed by CIE94 Delta-E and SSIM metrics [S1].
*   **Limitations:** Requires preprocessing for color quantization; not suitable for real-time attribute generation without pre-computed codebooks [S1].

## Background
Originating from work on geometry and attribute compression for voxel scenes, this method addresses the large memory footprint of attribute arrays in structured voxel representations [S1]. It builds on the observation that human perception is not flawless, allowing for lossy compression of attributes like colors and normals without significant visual degradation [S1]. The use of CIELAB for colors and ONVs for normals was adopted to improve upon standard RGB quantization and per-dimension storage, respectively [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
