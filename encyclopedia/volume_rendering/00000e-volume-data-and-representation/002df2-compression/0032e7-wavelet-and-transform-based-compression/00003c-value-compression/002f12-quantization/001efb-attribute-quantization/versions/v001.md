# Attribute Quantization

Also called: CIELAB color quantization, Perceptual quantization

## Overview
Attribute quantization is a lossy compression technique that reduces the precision of voxel attributes (primarily colors and normals) to lower memory footprint while maintaining perceptual quality [S1]. It addresses the problem where detailed scenes contain millions of distinct attribute values with small variations, allowing the reduction of distinct values by leveraging human perception limits [S1]. It differs from [Scalar Quantization](5726-scalar-quantization.md) and [Uniform Quantization](5855-uniform-quantization.md) by specializing the quantization method based on data type (e.g., using perceptually uniform color spaces or geometric mappings for normals) rather than applying a generic linear mapping [S1].

## How it works
**Inputs:** Attribute array containing colors (RGB) and/or normals (3D vectors).
**Outputs:** Quantized attribute values mapped to a reduced discrete set.

**Data Structures:**
*   **Color:** Mapped to CIELAB space. The number of distinct colors is determined by the bit depth.
*   **Normals:** Mapped to Octahedron Normal Vectors (ONVs).

**Steps:**

1.  **Color Quantization (Perceptual):**
    1.  Preprocess the attribute array to analyze the distribution of colors [S1].
    2.  Convert RGB colors to the CIELAB color space, which is (locally) perceptually uniform [S1].
    3.  Apply a data-driven clustering algorithm in the CIELAB space to reduce the number of distinct colors [S1].
    4.  Map original colors to the nearest cluster center.
    5.  *Note:* The original method by Xiang relied on clustering in a scaled RGB space, but CIELAB yields higher quality results for the same number of colors [S1].

2.  **Normal Quantization (ONV):**
    1.  Convert 3D normal vectors $\mathbf{n} = (n_x, n_y, n_z)$ to Octahedron Normal Vectors (ONVs) [S1].
    2.  Quantize the ONV coordinates uniformly.
    3.  *Benefit:* ONVs provide higher precision for the same number of bits compared to storing one value per dimension [S1].

**Formulas:**
*   **Color Difference:** Visual comparison uses the Euclidean distance in RGB space for error metrics:
    $$ \Delta E_{RGB} = \sqrt{dR^2 + dG^2 + dB^2} $$
    where $dR, dG, dB$ are the differences in color channels [S1].
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
A technique to further reduce data by neglecting coefficients of a compact model, making the model lossy. This is often applied after initial quantization to remove insignificant values [5722-thresholding-and-quantization.md].

### Quantization with Noise Injection
A training-specific quantization method used for continuous anchor attributes in neural volume representations to enable backpropagation through the rounding operation [S2, unofficial].

**Definition:** Quantizes continuous attributes to discrete values for entropy coding. During training, noise is injected to approximate the gradient of the rounding function; during inference, standard rounding is used [S2, unofficial].

**Steps:**
1.  Identify anchor attributes $f_i \in \{f_a, l_s, l_r, \Delta x_g\}$ [S2, unofficial].
2.  Compute the quantization step size $q_i$ using a refinement factor $r_i$ from a context model [S2, unofficial].
3.  **Training:** Substitute the original attribute $f_i$ with the noisy quantized version $\hat{f}_i$ for quantization-aware training [S2, unofficial].
4.  **Inference:** Round the attribute to the nearest discrete value [S2, unofficial].

**Formulas:**
*   **Training Quantization:**
    $$ \hat{f}_i = f_i + V(-1/2, 1/2) \times q_i $$
    where $V(-1/2, 1/2)$ represents uniform sampling from the interval $[-0.5, 0.5]$ [S2, unofficial].
*   **Inference Quantization:**
    $$ \hat{f}_i = \text{round}(f_i / q_i) \times q_i $$
*   **Adaptive Step Size:**
    $$ q_i = Q \times (1 + \tanh(r_i)) $$
    where $Q$ is the predefined quantization step and $r_i$ is the refinement factor [S2, unofficial].

**Parameters:**
*   **Predefined Step Size ($Q$):** Varies by component [S2, unofficial]:
    *   $Q = 1$ for $f_a$
    *   $Q = 0.001$ for $l_s$
    *   $Q = 0.001$ for $l_r$
    *   $Q = 0.2$ for $\Delta x_g$
*   **Attributes:** $f_a$ (anchor attributes), $l_s$ (scaling factor), $l_r$ (scaling factor), $\Delta x_g$ (geometry offset) [S2, unofficial].

## Operating characteristics
*   **Memory Reduction:** Attribute arrays often exceed the geometry (DAG) memory size. Quantization combined with palette compression can reduce the attribute memory footprint to a similar order of magnitude as the geometry [S1].
*   **Compression Rates:** Significant compression rates are achieved (e.g., 20-69% reduction in attribute array size depending on the scene) [S1].
*   **Perceptual Quality:** 12-bit colors and normals generally maintain high visual quality. 10-bit normals are acceptable for diffuse shading but show banding [S1].
*   **Non-Linear Scaling:** Combining multiple attributes (colors, normals, reflectance) does not result in a linear increase in memory due to shared compression benefits [S1].
*   **Preprocessing Cost:** Color quantization requires an initial analysis pass over the data [S1].

## Background
Attribute quantization for voxel scenes was proposed to address the large memory footprint of attribute arrays in hierarchical voxel representations (like SVOs) [S1]. The approach builds on the observation that human perception is not flawless, allowing lossy compression of attributes like colors and normals without significant visual degradation [S1]. The specific use of CIELAB for colors improves upon earlier RGB-based clustering methods [S1]. The use of Octahedron Normal Vectors (ONVs) for normals leverages prior work in normal vector quantization [MSS* 10, CDE* 14] to achieve better precision per bit [S1]. The noise injection variant is derived from techniques in neural field compression to enable differentiable quantization [S2, unofficial].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
[S2] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
