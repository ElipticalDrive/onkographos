# Quantization

Also called: Value Quantization, Data Quantization

## Overview
Quantization is a lossy compression technique that maps a continuous or high-precision range of data values to a limited set of discrete levels, thereby reducing the memory footprint and storage requirements of volume data [S1][S2, unofficial]. It solves the problem of excessive memory bandwidth and storage consumption inherent in high-precision scientific and medical volume datasets. It differs from its siblings by serving as the fundamental discretization step applied to scalar fields, attributes, or neural network weights, whereas methods like [Vector Quantization](907-vector-quantization.md) operate on groups of values and [Transform-based Block Coding](6882-transform-based-block-coding.md) operates on transformed coefficients.

## Shared principles
Quantization in volume rendering generally involves mapping a source value $x$ from a range $[x_{min}, x_{max}]$ to a discrete integer index $q$ in the range $[0, M-1]$, where $M$ is the number of quantization levels (e.g., $2^8=256$ for 8-bit). The mapping is typically linear, defined by a step size $\phi = (x_{max} - x_{min}) / M$. The reconstruction is performed via linear interpolation: $x' = x_{min} + q \cdot \phi$. The primary trade-off is between the bit-depth of the quantized representation and the resulting reconstruction error, which is often measured as Mean Squared Error (MSE) [S2, unofficial].

## Subtopics

### [Attribute Quantization](1912-attribute-quantization.md)
Attribute quantization specializes in reducing the precision of voxel attributes, specifically colors and normals, rather than scalar field values [1912-attribute-quantization.md]. It differs from [Uniform Quantization](5855-uniform-quantization.md) by using perceptually uniform color spaces (CIELAB) and geometric mappings (Octahedron Normal Vectors) to maintain visual quality [1912-attribute-quantization.md]. It is chosen when color fidelity and normal accuracy are critical for shading, such as in specular reflections [1912-attribute-quantization.md].

### [Scalar Quantization](5726-scalar-quantization.md)
Scalar quantization (per-block) is a random-access decompression method that maps a block of voxels to a local range defined by block-specific minimum and maximum values [5726-scalar-quantization.md]. It differs from [Uniform Quantization](5855-uniform-quantization.md) by computing range parameters locally per block rather than globally, allowing for parallel GPU decompression without loading full-resolution data [5726-scalar-quantization.md]. It is chosen when random access to specific voxel blocks is required without decoding the entire volume [5726-scalar-quantization.md].

### [Uniform Quantization](5855-uniform-quantization.md)
Uniform quantization maps raw data values to discrete levels using constant spacing between levels [5855-uniform-quantization.md]. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by using fixed step sizes regardless of data distribution [5855-uniform-quantization.md]. It is chosen as a baseline compression method, typically achieving a 4:1 compression ratio by representing 32-bit floats with 8-bit integers [5855-uniform-quantization.md].

### [Chroma Subsampling](6512-chroma-subsampling.md)
Chroma subsampling reduces the bitrate of volumetric data by lowering the spatial resolution of chrominance components (U, V) while retaining luminance (Y) at full resolution [6512-chroma-subsampling.md]. It differs from direct subsampling by aggregating color information from eight adjacent voxel corners using a median filter, which preserves sharpness [6512-chroma-subsampling.md]. It is chosen to prevent color loss at low bitrates [6512-chroma-subsampling.md].

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Differentiable quantization reduces the memory footprint of neural field representations (e.g., NeRF, 3D Gaussian Splatting) by compressing stored values to single bytes during the optimization process [10195-differentiable-quantization-for-neural-fields.md]. It differs from [Scalar Quantization](5726-scalar-quantization.md) by using a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation [10195-differentiable-quantization-for-neural-fields.md]. It is chosen to enable training of low-precision neural fields without the quality degradation associated with post-training quantization [10195-differentiable-quantization-for-neural-fields.md].

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Neural Implicit Network (INR) Compression replaces explicit volumetric grids with a coordinate-based neural network, using quantization to reduce model size [10414-neural-implicit-network-compression.md]. It differs from traditional lossy compressors by allowing direct evaluation at arbitrary resolutions [10414-neural-implicit-network-compression.md]. It is chosen for high compression ratios on scalar fields where tensor decomposition fails [10414-neural-implicit-network-compression.md].

### Lossy Quantization for Coarsened Patches
Lossy quantization for coarsened patches applies quantization error $q$ proportional to coarsening error $c$ ($q = qc$) within a bintree hierarchy [S1]. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else 0. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S1].

### HSV Colour Sorting
HSV colour sorting is a technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S2, unofficial]. The effectiveness depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S2, unofficial].

### Interpolation Symmetry Quantization
Interpolation symmetry quantization improves the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S3]. It quantizes both $w$ and $1 - w$ using the codebook generated by the scalar quantizer and keeps the representation resulting in the lower error [S3]. It stores either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream, where linear interpolation is $(1 - w)s_i + w s_j$ [S3].

### Position Lattice Quantization
Position lattice quantization is an attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S4]. It fails to compress positions without introducing significant rendering error [S4]. It is identified as a future research direction for end-to-end positional compression [S4].

### Quantization and Entropy Encoding/Decoding Pipeline
The quantization and entropy encoding/decoding pipeline involves quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression [S5]. It converts compressed data to non-negative values to enable compression into int8 format, then reverts to the original range during decompression [S5]. The formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $E$ is entropy encoder, $D$ is entropy decoder, $Q$ is quantization, and $q$ is the quantization parameter [S5]. Adjusting $q$ trades off reconstruction quality and model storage size [S5].

## Background
Quantization is a fundamental concept in signal processing and computer graphics, widely used in image and video compression standards (e.g., JPEG, MPEG) and adapted for volume rendering to handle large datasets. In volume rendering, it is often combined with transform-based methods (like wavelets) or used directly on scalar fields to enable efficient storage and transmission. The specific applications in neural fields and implicit networks represent a recent development, integrating quantization into the training loop to maintain differentiability.

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S3] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S4] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S5] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
