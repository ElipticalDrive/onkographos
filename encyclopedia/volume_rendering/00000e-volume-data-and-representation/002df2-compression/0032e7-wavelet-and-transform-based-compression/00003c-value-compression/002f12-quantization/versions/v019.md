# Quantization

Also called: Quantization with Matrix

## Overview
Quantization reduces the precision of volumetric data coefficients or values to decrease storage and transmission requirements, serving as the primary lossy step in transform-based compression pipelines. It sets itself apart from sibling methods like [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by operating on the magnitude of coefficients or values directly rather than on prediction residuals, and from [Vector Quantization](907-vector-quantization.md) by typically mapping values to a fixed uniform grid rather than a learned codebook.

## Shared principles
The family shares the mechanism of mapping a continuous or high-precision input domain to a discrete, lower-precision output domain. This is achieved through a quantization function $Q(\cdot)$, often parameterized by a scaling factor or step size. The general form for scalar quantization of a value $x$ with step size $\Delta$ is:
$$ Q(x) = \text{round}\left(\frac{x}{\Delta}\right) $$
The inverse operation (dequantization) reconstructs the value:
$$ \hat{x} = Q(x) \cdot \Delta $$
In transform-based contexts, the quantization step is often combined with a quantization matrix $Q(u,v,w)$ to apply non-uniform precision across frequency bands, as seen in [S1].

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
Maps $n$-dimensional input vectors to indices in a pre-defined codebook. It exploits spatial redundancy in local data blocks (e.g., $4 \times 4 \times 4$) to achieve high compression ratios for data with high spatial coherence. It is chosen when extremely fast decompression is required, as it can be implemented via GPU texture lookups. Unlike scalar quantization, it does not rely on linear interpolation between bounds but on the closest match in a learned dictionary.

### [Scalar Quantization](5726-scalar-quantization.md)
Maps voxel values within a localized block to a linear interpolation range defined by the block's local minimum and maximum. It sets itself apart by enabling random-access decompression without storing fully decompressed data, making it suitable for on-the-fly rendering. It is simpler to implement than vector quantization but may be less accurate for complex data distributions.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Reduces the spatial resolution of chrominance components (U, V) while maintaining full resolution for luminance (Y). It addresses color loss in sparse volume representations at low bitrates by defining one color value for every eight adjacent corners, preserving color quality better than direct subsampling.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Integrates quantization into the training loop of neural fields using a straight-through estimator. It reduces memory footprint by mapping grid values to single bytes during optimization, ensuring consistency between the trained model and the render-time representation. It differs from post-training quantization by avoiding quality drops caused by the mismatch between optimized high-precision values and discrete render-time values.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Compresses the weights of neural implicit networks (e.g., SIREN) using techniques like k-means clustering. It replaces explicit volumetric grids with a compact neural network, allowing direct rendering from network weights. It distinguishes itself by embedding quantization logic into the network architecture rather than post-processing data.

### [Unquantization Functions](13872-unquantization-functions.md)
Maps integer quantized values back to half-precision float range using specific fixed-point scaling and rounding constants. It is used in contexts like BC6H texture compression to reconstruct floating-point color data from compressed integer indices.

### Lossy Quantization for Coarsened Patches
A method for lossy compression of coarsened patches in a bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$) [S2]. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else 0 [S2]. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S2].

### HSV Colour Sorting
A technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S3, unofficial]. The effectiveness depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S3, unofficial].

### Interpolation Symmetry Quantization
A method to improve the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S4]. It quantizes both $w$ and $1 - w$ using the codebook generated by the scalar quantizer and keeps the representation resulting in the lower error [S4]. It stores either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream, where $(1 - w)s_i + w s_j$ is equivalent to $(j, i, 1 - w)$ [S4].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S5]. It fails to compress positions without introducing significant rendering error and is identified as a future research direction for end-to-end positional compression [S5].

### Quantization and Entropy Encoding/Decoding Pipeline
The compression procedure involving quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression [S6]. It converts compressed data to non-negative values to enable compression into int8 format, then reverts to the original range during decompression [S6]. The formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $E$ is entropy encoder, $D$ is entropy decoder, $Q$ is quantization, and $q$ is the quantization parameter [S6]. Adjusting $q$ trades off reconstruction quality and model storage size [S6].

### Post-finetune Refinement
A refinement step that uses a small number of bits to enhance the reconstruction quality of important voxels, compensating for losses introduced by coarse quantization [S7]. Disabling post-finetune causes a -1.72 dB $\Delta$PSNR, -0.07 $\Delta$SSIM, and -0.079 MB $\Delta$Size on the Synthetic-NeRF dataset [S7]. It is necessary for maintaining RD performance [S7].

## Background
Quantization is a fundamental step in lossy compression, originating from signal processing theory. In the context of volume rendering and neural fields, it has been adapted to handle the specific constraints of 3D data, such as spatial coherence, sparse representations, and the need for real-time rendering. The use of quantization matrices in 3D DCT coefficients for streaming free-viewpoint videos was detailed by [S1]. The integration of quantization into neural field training via differentiable estimators was introduced by [S10195]. The application of quantization to Gaussian splatting positions and weights has been explored in recent works [S5, S10414].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S3] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S4] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S5] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S6] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S7] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
