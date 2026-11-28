# Quantization

Also called: Value quantization, Precision reduction

## Overview
Quantization reduces the precision of volume data values to lower memory footprint and bandwidth requirements, serving as the primary lossy compression step in transform-based and neural volume rendering pipelines [S1][S2, unofficial][S3, S4, S5]. It differs from its siblings by focusing on the discretization of scalar or vector values (colors, normals, densities) rather than the structural encoding of the data hierarchy or the entropy coding of the resulting symbols.

## Shared principles
Not given in the sources.

## Subtopics

### [Attribute Quantization](1912-attribute-quantization.md)
Attribute quantization specializes in reducing the precision of voxel attributes (colors and normals) by leveraging perceptual uniformity and geometric mappings, rather than applying generic linear mappings. It is chosen when visual fidelity of surface attributes is critical, using CIELAB clustering for colors and Octahedron Normal Vectors for normals.

### [Scalar Quantization](5726-scalar-quantization.md)
Scalar quantization (per-block) maps a block of voxels to a local range defined by block-specific minimum and maximum values, enabling parallel GPU decompression without loading full-resolution data. It differs from global scalar quantization by computing ranges locally per block and from hardware-accelerated block-coding by using software-defined linear interpolation.

### [Uniform Quantization](5855-uniform-quantization.md)
Uniform quantization maps data values to a limited set of discrete levels using constant spacing, typically achieving 4:1 compression by representing 32-bit floats with 8-bit integers. It is chosen for its simplicity and speed, though it may be ineffective for data with wide dynamic ranges compared to non-uniform methods.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Chroma subsampling reduces the resolution of chrominance components (U, V) while maintaining luminance (Y) at full resolution, storing one color value for every eight adjacent corners. It is specifically designed to prevent the complete loss of chroma information (grey objects) in sparse volumetric data, using median filtering to preserve sharpness.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Differentiable quantization integrates quantization directly into the training loop of neural fields (e.g., NeRF, 3D Gaussian Splatting) using a straight-through estimator, reducing memory footprint to single bytes. It sets itself apart from post-training quantization by allowing gradients to flow through the rounding operation, ensuring the model adapts to the low-precision representation.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Neural Implicit Network Compression replaces explicit grids with a coordinate-based neural network (e.g., SIREN) where weights are quantized (e.g., 9-bit via k-means) to serve as compressed storage. It differs from traditional compressors by allowing direct evaluation at arbitrary resolutions, though it lacks interactive rendering speed for large volumes.

### Lossy Quantization for Coarsened Patches
Lossy compression for coarsened patches in a bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$) [S1]. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else 0. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision, and quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S1].

### HSV Colour Sorting
A technique to preserve color quality during DXT1 compression by reordering colors based on HSV values before storage [S2, unofficial]. Its effectiveness depends on which HSV component (H, S, or V) is the most frequent in the voxel data's color set [S2, unofficial].

### Interpolation Symmetry Quantization
A method to improve quantization accuracy by exploiting the symmetry of linear interpolation between segments $s_i$ and $s_j$ [S3]. It quantizes both weight $w$ and $1-w$ using the same codebook, keeping the representation with lower error, and stores either $(i, j, w)$ or $(j, i, 1-w)$ [S3]. The interpolation is defined as $(1-w)s_i + w s_j$ [S3].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into training [S4]. It fails to compress positions without introducing significant rendering error and is identified as a future research direction [S4].

### Quantization and Entropy Encoding/Decoding Pipeline
A pipeline where quantization is followed by entropy encoding (e.g., range-coder) for compression [S5]. To enable int8 compression, data is converted to non-negative values and reverted during decompression [S5]. The reconstruction formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $q$ is the quantization parameter that expands the data range to enhance precision [S5].

## Background
Quantization is a fundamental technique in data compression, widely used in volume rendering to manage the high memory bandwidth and storage costs associated with high-precision scientific and medical data. Early applications focused on uniform and scalar quantization for terrain and voxel data [S1][S2, unofficial]. With the advent of neural rendering, differentiable quantization has become essential for compressing neural fields and Gaussian splats [S1, S4]. The method is used in various formats and pipelines, including DXT1 compression for voxel animations [S2, unofficial], Mixture Graphs for segmentation histograms [S3], and end-to-end joint optimization for dynamic NeRFs [S5].

## Sources
[S1] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S2] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S3] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S4] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S5] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
