# Quantization

Also called: Quantization with Matrix

## Overview
Quantization is the process of mapping a large set of input values to a smaller, discrete set of output values, serving as the primary lossy compression step in volume rendering pipelines. It reduces the precision of volumetric data (voxel values, neural field weights, or attributes) to minimize memory footprint and bandwidth requirements. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which focuses on spatial frequency separation, or [Scalar Quantization](5726-scalar-quantization.md) which handles per-block linear interpolation, this family encompasses diverse strategies including data-driven codebooks, perceptual color spaces, and differentiable integration into neural training loops.

## Shared principles
Not given in the sources.

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
[Vector Quantization](907-vector-quantization.md) maps $n$-dimensional input vectors to a fixed-length index referencing a codebook entry. It exploits spatial redundancy in local data blocks (e.g., $4 \times 4 \times 4$) to achieve high compression ratios with extremely fast GPU decompression via texture lookups.

### [Attribute Quantization](1912-attribute-quantization.md)
[Attribute Quantization](1912-attribute-quantization.md) reduces the precision of voxel attributes like colors and normals. It differs from generic approaches by using specialized representations such as CIELAB for colors and Octahedron Normal Vectors for normals to maintain perceptual quality.

### [Scalar Quantization](5726-scalar-quantization.md)
[Scalar Quantization](5726-scalar-quantization.md) is a random-access method that reconstructs voxel values on-the-fly using block-specific minimum and maximum values. It enables parallel GPU decompression without loading full-resolution data, relying on software-defined linear interpolation rather than fixed hardware tables.

### [Uniform Quantization](5855-uniform-quantization.md)
[Uniform Quantization](5855-uniform-quantization.md) maps continuous data to discrete levels with fixed spacing (e.g., 32-bit to 8-bit). It is simple to implement but less effective than non-uniform methods for data with wide dynamic ranges or specific feature importance.

### [Chroma Subsampling](6512-chroma-subsampling.md)
[Chroma Subsampling](6512-chroma-subsampling.md) reduces the bitrate by lowering the spatial resolution of chrominance components (U, V) while maintaining full resolution for luminance (Y). It uses median aggregation over $2 \times 2 \times 2$ blocks to prevent "grey object" artifacts in sparse data.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
[Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md) compresses neural radiance field grid values into single bytes during training. It uses a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation, preventing the quality degradation associated with post-training quantization.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
[Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) replaces large voxel grids with compact coordinate-based neural network weights. It serves as both compressed storage and a queryable data source, allowing direct evaluation at arbitrary resolutions.

### Lossy Quantization for Coarsened Patches
A method for lossy compression of coarsened patches in a bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$) [S2]. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else 0 [S2]. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S2].

### HSV Colour Sorting
A technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S3, unofficial]. The effectiveness depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S3, unofficial].

### Interpolation Symmetry Quantization
A method to improve the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S4]. It quantizes both $w$ and $1 - w$ using the scalar quantizer's codebook and keeps the representation resulting in the lower error. It stores either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream, corresponding to the equivalent linear interpolation $(1 - w)s_i + w s_j$ [S4].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S5]. It fails to compress positions without introducing significant rendering error and is identified as a future research direction [S5].

### Quantization and Entropy Encoding/Decoding Pipeline
A compression procedure involving quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression [S6]. The procedure converts compressed data to non-negative values to enable compression into int8 format, then reverts to the original range during decompression [S6]. The formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $E$ is the entropy encoder, $D$ is the entropy decoder, $Q$ is quantization, and $q$ is the quantization parameter [S6]. Adjusting $q$ trades off reconstruction quality and model storage size [S6].

### Post-finetune Refinement
A refinement step that uses a small number of bits to enhance the reconstruction quality of important voxels, compensating for losses introduced by coarse quantization [S7]. An ablation study on the Synthetic-NeRF dataset shows that disabling post-finetune causes a -1.72 dB $\Delta$PSNR, -0.07 $\Delta$SSIM, and -0.079 MB $\Delta$Size [S7]. Disabling it saves bits but inevitably deteriorates rendering quality [S7].

## Background
Quantization is a fundamental technique in data compression, widely used in image and video coding standards (e.g., JPEG, MPEG) and increasingly in neural radiance fields (NeRFs) and volumetric rendering. In the context of volume rendering, it addresses the high storage and bandwidth requirements of large-scale 3D datasets. Early applications involved scalar quantization of DCT coefficients [S1]. More recent developments integrate quantization directly into the training of neural representations to ensure differentiability and perceptual fidelity [S1, S6, S7].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S3] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S4] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S5] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S6] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S7] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
