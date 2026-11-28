# Quantization

Also called: Quantization with Matrix

## Overview
Quantization is a lossy compression technique that maps continuous or high-precision input data to a discrete, lower-precision set of levels, thereby reducing the memory footprint of volumetric data [S1, S2][S5]. It solves the problem of high storage requirements and bandwidth constraints by exploiting redundancy in the data domain, whether through uniform spacing, data-specific codebooks, or neural field constraints [S1, S2][S5]. Unlike [Scalar Quantization](5726-scalar-quantization.md) which operates on localized blocks with local bounds, or [Vector Quantization](907-vector-quantization.md) which relies on learned dictionaries, this family encompasses a broad spectrum of mapping strategies including uniform, chroma subsampling, and differentiable quantization for neural fields [S1, S2][S5].

## Shared principles
The family of quantization methods shares the fundamental mechanism of mapping a high-dimensional or high-precision input space to a lower-dimensional or lower-precision discrete space. While specific implementations vary (e.g., uniform steps vs. codebook indices), the core mathematical operation involves a quantization function $Q(\cdot)$ that maps an input value $x$ to a quantized index or level $q$, and an inverse unquantization function $U(\cdot)$ that maps $q$ back to an approximate value $\hat{x}$ [S1, S2][S6].

1.  **Discretization:** The input domain is partitioned into regions (bins) or mapped to a finite set of representative values (codewords or levels) [S1, S2].
2.  **Error Minimization:** The goal is to minimize a distortion metric, typically Mean Squared Error (MSE), between the original value $x$ and the reconstructed value $\hat{x}$ [S1, S2].
3.  **Bitrate Control:** The precision of the quantization (number of bits per value) directly controls the compression ratio and reconstruction quality [S1, S2][S6].

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
Vector Quantization (VQ) maps $n$-dimensional input vectors to a single fixed-length index referencing an entry in a pre-defined or learned codebook. It solves the problem of high storage requirements by exploiting spatial redundancy, where many local data blocks exhibit similar patterns. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which relies on fixed mathematical bases, VQ uses data-specific dictionaries that typically achieve better compression ratios for data with high spatial coherence. Its primary advantage is extremely fast decompression, often implemented directly on the GPU via texture lookups, making it suitable for real-time rendering of large or time-varying volumes.

### [Scalar Quantization](5726-scalar-quantization.md)
Scalar quantization (per-block) is a random-access decompression method used in compressed volume rendering to reconstruct voxel values on-the-fly without storing fully decompressed data in memory. It sets itself apart from general scalar quantization by operating on localized blocks, computing local minimum and maximum bounds to define a linear interpolation range, which allows for efficient GPU implementation and direct rendering from compressed data. Unlike vector quantization, it does not rely on codebooks or weighted combinations of multiple basis vectors, making it simpler to implement but potentially less accurate for complex data distributions.

### [Uniform Quantization](5855-uniform-quantization.md)
Uniform quantization is a lossy compression method that maps a continuous or high-precision input domain to a discrete, lower-precision set of levels using constant step sizes. It solves the problem of reducing the memory footprint of volume data, typically achieving a 4:1 compression ratio by representing 32-bit floating-point data with 8-bit integers. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by allocating levels with equal spacing across the entire data range, rather than adapting density to data distribution or user-defined importance.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Chroma subsampling for volumetric data is a lossy compression technique that reduces the resolution of chrominance components (U, V) while maintaining the luminance component (Y) at full resolution, thereby preserving color fidelity at low bitrates. Unlike direct subsampling which causes significant information loss in sparse volumes, this method aggregates color information from groups of eight adjacent voxel corners, ensuring that color is not completely lost even when the bitrate is insufficient to encode every voxel's color. It sets itself apart from sibling quantization methods by specifically addressing the spatial sparsity and geometric nature of volumetric data, where color is defined only on existing surface voxels rather than in a dense grid.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
This method reduces the render-time memory footprint of neural radiance fields by quantizing grid-based feature values to single bytes during the optimization process. Unlike post-training quantization, which causes quality degradation due to mismatch between the optimized model and the inference model, this approach integrates quantization into the training loop using a straight-through estimator. It sets itself apart from sibling methods like [Vector Quantization](907-vector-quantization.md) by using uniform byte-level quantization on grid values rather than clustering parameters into codebooks, and from [Scalar Quantization](5726-scalar-quantization.md) by explicitly handling the non-differentiable nature of rounding via gradient approximation during training.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Neural Implicit Network Compression (INR Compression) replaces traditional grid-based storage of scientific volumes with a compact coordinate-based neural network that maps spatial coordinates directly to field values. This approach serves as simultaneously compressed storage and a queryable data source, offering an alternative to lossy compressors like SZ and ZFP that require full decoding before visualization. It sets itself apart from siblings like [Quantization](5935-quantization.md) by encoding the data structure itself into network weights rather than discretizing values, and from [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md) by focusing on weight quantization and architectural efficiency for scientific data rather than differentiable quantization layers for training stability.

### [Unquantization Functions](13872-unquantization-functions.md)
Unquantization functions reconstruct continuous or high-precision floating-point values from discrete, lower-precision integer representations. This process is the inverse of quantization, mapping quantized indices back to their approximate original domain to minimize reconstruction error. In the context of volume rendering and texture compression (specifically BC6H), these functions are critical for converting compressed half-precision or integer data back into formats suitable for rendering or further processing.

### Lossy Quantization for Coarsened Patches
Lossy compression for coarsened patches in the bintree hierarchy. Quantization error $q$ is proportional to coarsening error $c$: $q = qc$, where $q$ is a fixed constant usually $< 1$. A single unified error is recomputed and nested for each patch. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. Shift $k$ is calculated as: $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else $0$. All finest level vertices have zero shift. Vertices classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S2].

### HSV Colour Sorting
A technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S3, unofficial]. The effectiveness of HSV sorting depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S3, unofficial].

### Interpolation Symmetry Quantization
A method to improve the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S4]. Quantize both $w$ and $1 - w$ using the codebook generated by the scalar quantizer. Keep the representation resulting in the lower error. Store either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream. Linear interpolation: $(1 - w)s_i + w s_j$, where $(i, j, w)$ is equivalent to $(j, i, 1 - w)$ [S4].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S5]. Fails to compress positions without introducing significant rendering error [S5]. Identified as a future research direction for end-to-end positional compression [S5].

### Quantization and Entropy Encoding/Decoding Pipeline
The compression procedure involving quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression [S6]. Convert compressed data to non-negative values to enable compression into int8 format, then revert to original range during decompression. $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $E$ is entropy encoder, $D$ is entropy decoder, $Q$ is quantization, and $q$ is the quantization parameter [S6]. $q$: The quantization parameter. Data is multiplied by $q$ to expand the data range, enhancing quantization precision. Adjusting $q$ trades off reconstruction quality and model storage size [S6].

### Post-finetune Refinement
A refinement step that uses a small number of bits to enhance the reconstruction quality of important voxels, compensating for losses introduced by coarse quantization [S7]. Ablation study on Synthetic-NeRF dataset shows disabling post-finetune causes a -1.72 dB $\Delta$PSNR, -0.07 $\Delta$SSIM, and -0.079 MB $\Delta$Size [S7]. Disabling post-finetune saves bits but inevitably deteriorates rendering quality, confirming its necessity for maintaining RD performance [S7].

### Quantization Functions
Converts float3 to half-precision and scales to integer range for 7, 9, or 10-bit quantization [S8, unofficial]. Quantized = (Half(x) * Scale) / 32767, where Scale is 128, 512, or 1024 for 7, 9, or 10 bits respectively [S8, unofficial].

## Background
Quantization is a foundational technique in signal processing and data compression, widely applied in volume rendering to manage the large memory footprint of volumetric datasets. In the context of volume rendering, it is often combined with transform-based methods (such as DCT or Wavelets) where coefficients are quantized using a quantization matrix derived from psycho-visual experiments [S1]. The scaling factor $S_q$ in the quantization formula $\hat{R}(u,v,w) = \text{round}(\frac{R(u,v,w)}{S_q \times Q(u,v,w)})$ allows for dynamic adaptive streaming by adjusting bitrates [S1].

Historically, quantization has evolved from simple uniform scalar quantization to more sophisticated methods like vector quantization with learned codebooks [S5], and differentiable quantization for neural radiance fields [S1]. The integration of quantization into the training process of neural fields (e.g., using straight-through estimators) represents a recent advancement to mitigate the quality degradation associated with post-training quantization [S1]. Additionally, specialized quantization strategies like chroma subsampling address the unique sparsity and geometric properties of volumetric color data [S1].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S3] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S4] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S5] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S6] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S7] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
[S8] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
