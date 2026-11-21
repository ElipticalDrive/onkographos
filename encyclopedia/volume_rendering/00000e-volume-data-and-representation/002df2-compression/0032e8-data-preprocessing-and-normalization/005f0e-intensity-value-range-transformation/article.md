# Intensity/Value Range Transformation

Also called: Intensity Transformation, Value Range Transformation, Density Range Restriction

## Overview
Intensity/Value Range Transformation is a family of preprocessing techniques that map raw scalar voxel values to a specific target range or distribution before compression or rendering. It solves the problem of wide dynamic ranges, irrelevant background data, and numerical instability by filtering or compressing the intensity spectrum. It differs from sibling methods like [Geometric Normalization](12018-geometric-normalization.md) (which handles spatial coordinates) and [Byte/Short Conversion](11742-byte-short-conversion.md) (which handles quantization) by operating directly on the continuous or high-precision scalar values to shape their statistical distribution or physical relevance.

## Shared principles
The family operates on a volume tensor $V$ containing scalar voxel values. The core mechanism involves applying a transformation function $T(v)$ to each voxel $v \in V$. This function is typically piecewise linear, logarithmic, or affine, designed to map the input domain to a target codomain (e.g., $[0, 1]$ or $[0, 255]$). Common goals include:
1.  **Range Restriction:** Setting values outside a region of interest to zero or a background value.
2.  **Dynamic Range Compression:** Reducing the span of values to fit storage or rendering constraints.
3.  **Discontinuity Avoidance:** Using ramps to prevent high-frequency artifacts during subsequent compression (e.g., wavelet transforms).

## Subtopics

### [Volume Normalization](1823-volume-normalization.md)
Maps a volume tensor's scalar values to the range $[0, 1]$ while preserving constant positive masks as full foreground ($1.0$). It is chosen when a uniform $[0, 1]$ range is required for downstream neural networks or transfer functions. See [Volume Normalization](1823-volume-normalization.md) for implementation details.

### Density Range Restriction and Ramp Function
A preprocessing function applied to volume data to restrict the range of density values, improving compression ratio by removing irrelevant data and avoiding discontinuities that generate high frequencies [S1, unofficial]. It is chosen when using wavelet-based compression (e.g., Haar Wavelet) where discontinuities degrade compression efficiency.

**Definition:**
The function $T$ filters density values to keep only those representing regions of interest. It keeps densities intact between a selected range $[d_2, d_3]$ and adds ramps before $d_2$ and after $d_3$ to avoid discontinuities [S1, unofficial].

**Steps:**
1.  Apply function $T$ to each density value:
    *   If density $\le d_1$, set density $= 0$.
    *   If density $\ge d_4$, set density $= 0$.
    *   If $d_1 \le \text{density} \le d_2$, set density $= (\text{density} - d_1) \times \frac{d_2}{d_2 - d_1}$.
    *   If $d_3 \le \text{density} \le d_4$, set density $= (d_4 - \text{density}) \times \frac{d_4}{d_4 - d_3}$.
    *   *Note: The source text states "if density $\ge d_3 \rightarrow density = 0$" in the summary but uses $d_4$ in the formula block. The formula block is more precise for a ramp ending at $d_4$.* [S1, unofficial].
2.  Normalize models with histograms outside the $[0, 255]$ range to achieve the same output value range after applying $T$ [S1, unofficial].

**Parameters:**
*   $d_1, d_2, d_3, d_4$: Threshold parameters defining the density range $[d_2, d_3]$ to keep intact and the ramps $[d_1, d_2]$ and $[d_3, d_4]$ to avoid discontinuities [S1, unofficial].

### Logarithmic Intensity Compression
Compresses volume intensity data logarithmically to handle wide dynamic ranges typical in diffraction data [S2, unofficial]. It is chosen when raw linear mapping leaves most data transparent due to orders-of-magnitude intensity spans (e.g., Bragg peaks in crystallography).

**Definition:**
Applies a logarithmic transformation to clamp negatives and compress high values [S2, unofficial].

**Steps:**
1.  Cast input to float32.
2.  Apply $\max(x, 0.0)$ to clamp negatives.
3.  Apply $\log(1 + x)$ to compute the compressed output.

**Formula:**
$$y = \log(1 + \max(x, 0))$$
where $x$ is the input intensity array and $y$ is the compressed output [S2, unofficial].

**Pitfalls:**
Raw linear mapping leaves everything but the brightest Bragg peak fully transparent due to orders-of-magnitude intensity span [S2, unofficial].

### Triplane Feature Normalization
Normalizes feature channels to zero-mean, clips outliers, and scales to a fixed range. It is chosen for 3D neural field generation (e.g., Triplane Diffusion) where unbounded features skew the distribution [S3].

**Steps:**
1.  Normalize feature channels to zero-mean [S3].
2.  Clip each channel to be within $S = 16$ standard deviations of the mean [S3].
3.  Scale each channel to be within the range $[-1, 1]$ [S3].

## Background
Intensity transformation is a fundamental step in volume data pipelines, particularly in medical imaging and scientific visualization. Early implementations focused on simple windowing and leveling. The specific "Density Range Restriction and Ramp Function" was introduced in the context of mobile medical image visualization to improve compression ratios using Haar Wavelets [S1, unofficial]. Logarithmic compression is standard in diffraction data analysis (e.g., X-ray crystallography) to handle the vast dynamic range of pixel intensities [S2, unofficial]. Triplane normalization is a modern technique used in generative 3D models (e.g., Triplane Diffusion) to stabilize training by bounding feature distributions [S3].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S2] stacy-zhang/voxel, https://github.com/stacy-zhang/voxel, fetched 2026-10-28 (unofficial)
[S3] 3D Neural Field Generation Using Triplane Diffusion, https://doi.org/10.1109/cvpr52729.2023.02000, fetched 2026-10-26
