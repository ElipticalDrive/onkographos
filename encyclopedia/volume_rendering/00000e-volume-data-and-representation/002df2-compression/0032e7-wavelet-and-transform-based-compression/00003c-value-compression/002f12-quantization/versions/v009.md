# Quantization

Also called: Value Quantization, Precision Reduction

## Overview
Quantization is a lossy compression technique that maps a continuous or high-precision range of data values to a limited set of discrete levels, thereby reducing memory bandwidth and storage requirements for volume data [S1][S2, unofficial]. It differs from general lossless compression by intentionally discarding precision, a trade-off justified by human perception limits or the redundancy in high-precision scientific data [S1]. While [Uniform Quantization](5855-uniform-quantization.md) applies fixed step sizes and [Scalar Quantization](5726-scalar-quantization.md) uses local block ranges, this family encompasses specialized methods for attributes, neural fields, and geometric structures.

## Shared principles
Not given in the sources.

## Subtopics

### [Attribute Quantization](1912-attribute-quantization.md)
Attribute quantization reduces the precision of voxel attributes, specifically colors and normals, to lower the memory footprint while maintaining perceptual quality. It differs from [Scalar Quantization](5726-scalar-quantization.md) by specializing the quantization method based on data type, using perceptually uniform color spaces or geometric mappings for normals rather than generic linear mappings. See [Attribute Quantization](1912-attribute-quantization.md).

### [Scalar Quantization](5726-scalar-quantization.md)
Scalar quantization (per-block) is a random-access decompression method that reconstructs voxel values on-the-fly from compressed data. It solves the problem of storing high-precision volume data by mapping a block of voxels to a local range defined by block-specific minimum and maximum values, allowing for parallel GPU decompression without loading full-resolution data into memory. It differs from general scalar quantization by computing range parameters locally per block and from hardware-accelerated block-coding by relying on software-defined linear interpolation formulas. See [Scalar Quantization](5726-scalar-quantization.md).

### [Uniform Quantization](5855-uniform-quantization.md)
Uniform quantization maps a large range of raw data values to a limited set of discrete levels using constant spacing between levels. It typically achieves a 4:1 compression ratio by representing 32-bit floating-point data with 8-bit integers. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by using fixed step sizes regardless of data distribution. See [Uniform Quantization](5855-uniform-quantization.md).

### [Chroma Subsampling](6512-chroma-subsampling.md)
Chroma subsampling reduces the bitrate of volumetric data by lowering the spatial resolution of chrominance components (U, V) while retaining luminance (Y) at full resolution. It differs from direct subsampling by aggregating color information from eight adjacent voxel corners using a median filter, which preserves sharpness and reduces outlier influence compared to averaging.

### [TetSplat Pre-processing and Data Layout](7221-tetsplat-pre-processing-and-data-layout.md)
TetSplat Pre-processing and Data Layout compresses scalar field values on tetrahedral meshes into a compact binary format for efficient storage and interactive visualization. It differs from general scalar quantization by integrating field value quantization directly into a hierarchical tree structure (Solid octree and Shell quadtree) that also encodes geometric topology, using specific bit-packing schemes for field values and normal vectors.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by compressing stored values to single bytes during the optimization process. Unlike post-training quantization, which applies discretization after convergence and causes significant quality degradation, this method integrates quantization into the training loop. It sets itself apart from standard [Scalar Quantization](5726-scalar-quantization.md) by using a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Neural Implicit Network (INR) Compression replaces explicit volumetric grids with a coordinate-based neural network that maps spatial coordinates directly to field values. Unlike traditional lossy compressors that produce bitstreams requiring full decoding, INRs allow direct evaluation at arbitrary resolutions. The primary method, **neurcomp**, distinguishes itself by using a SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering, achieving high compression ratios on scalar fields where tensor decomposition methods fail. It builds on [Quantization](5935-quantization.md) principles to reduce model size.

#### Lossy Quantization for Coarsened Patches
Lossy compression for coarsened patches in the bintree hierarchy. Quantization error $q$ is proportional to coarsening error $c$: $q = qc$, where $q$ is a fixed constant usually $< 1$. A single unified error is recomputed and nested for each patch. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. Shift $k$ is calculated as: $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else $0$. All finest level vertices have zero shift. Vertices classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S1].

#### HSV Colour Sorting
A technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S2, unofficial]. The effectiveness of HSV sorting depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S2, unofficial].

#### Interpolation Symmetry Quantization
A method to improve the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S3]. Quantize both $w$ and $1 - w$ using the codebook generated by the scalar quantizer. Keep the representation resulting in the lower error. Store either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream. Linear interpolation: $(1 - w)s_i + w s_j$, where $(i, j, w)$ is equivalent to $(j, i, 1 - w)$ [S3].

#### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S4]. Fails to compress positions without introducing significant rendering error [S4]. Identified as a future research direction for end-to-end positional compression [S4].

## Background
Quantization is a fundamental technique in data compression, widely used in image and video processing before being adapted for volume rendering. In volume rendering, the high precision of scientific data (often 32-bit floats) necessitates reduction to fit within memory bandwidth constraints or to accelerate rendering. Early applications focused on uniform quantization for scalar fields [S1][S2, unofficial]. Later developments introduced perceptual quantization for attributes like color and normals [S1], and more recently, differentiable quantization for neural implicit representations [S1]. The technique is integral to various compression standards and custom codecs for volumetric data.

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S3] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S4] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
