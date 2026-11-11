# Quantization

Also called: Quantization with Matrix

## Overview
Quantization is a lossy compression technique that maps continuous or high-precision discrete data values to a limited set of discrete levels, reducing the memory footprint and bandwidth of volumetric data [S1, S2]. It solves the problem of storing high-precision volume data by exploiting redundancy in spatial or attribute domains, often achieving significant compression ratios (e.g., 4:1 for 32-bit floats to 8-bit integers) [S1, S2]. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which relies on fixed mathematical bases for coefficient generation, quantization methods like [Vector Quantization](907-vector-quantization.md) use data-specific dictionaries, while others like [Uniform Quantization](5855-uniform-quantization.md) use fixed spacing, setting them apart by their approach to mapping source values to discrete indices [S5, S1, S2].

## Shared principles
All methods in this family share the mechanism of decomposing the input volume $V$ into a set of coefficients $C$ using a linear transform, followed by encoding and optional decoding for rendering. This builds on [Value compression](6-value-compression.md).

1.  **Transform:** The volume is decomposed using a separable 1D filter bank applied along each axis. For a 3D volume, this results in subbands representing different frequency components. In wavelet transforms, these are typically approximation (low-pass) and detail (high-pass) coefficients. In Laplacian pyramids, they are detail residuals between levels of a Gaussian pyramid.
2.  **Hierarchy:** The decomposition is recursive. The approximation subband from level $j$ is further decomposed at level $j+1$, creating a tree-like structure of coefficients. This allows for multi-resolution access, where coarse data is available at the root and fine details are deeper in the tree.
3.  **Quantization and Entropy Encoding:** Coefficients are quantized (lossy) or kept precise (lossless). Entropy coding (e.g., Huffman, Arithmetic, Run-Length) is applied to the quantized or raw coefficients to produce the compressed bitstream.
4.  **Reconstruction:** To render, the necessary coefficients are decoded and the inverse transform is applied. This can be done globally (full reconstruction) or locally (block-wise or point-wise) to support adaptive resolution rendering.

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
Vector Quantization (VQ) is a lossy compression technique that maps $n$-dimensional input vectors to a single fixed-length index referencing an entry in a pre-defined or learned codebook (dictionary). It solves the problem of high storage requirements for volumetric data by exploiting spatial redundancy, where many local data blocks exhibit similar patterns. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which relies on fixed mathematical bases, VQ uses data-specific dictionaries that typically achieve better compression ratios for data with high spatial coherence. Its primary advantage is extremely fast decompression, often implemented directly on the GPU via texture lookups, making it suitable for real-time rendering of large or time-varying volumes.

### [Attribute Quantization](1912-attribute-quantization.md)
Attribute Quantization is a lossy compression technique that reduces the precision of voxel attributes, such as colors and normals, to lower memory footprint while maintaining perceptual quality. It solves the problem of excessive memory usage in attribute arrays by leveraging human perception limits to reduce the number of distinct values. It differs from [Scalar Quantization](5726-scalar-quantization.md) and [Uniform Quantization](5855-uniform-quantization.md) by specializing the quantization method based on the data type (e.g., using CIELAB for colors and Octahedron Normal Vectors for normals) to achieve higher precision and better compression ratios than generic uniform approaches.

### [Scalar Quantization](5726-scalar-quantization.md)
Scalar quantization (per-block) is a random-access decompression method used in compressed volume rendering to reconstruct voxel values on-the-fly from compressed data. It solves the problem of storing high-precision volume data by mapping a block of voxels to a local range defined by block-specific minimum and maximum values, allowing for parallel GPU decompression without loading full-resolution data into memory. It differs from general scalar quantization by computing range parameters locally per block rather than globally, and from hardware-accelerated block-coding (like S3TC) by relying on software-defined linear interpolation formulas rather than fixed hardware lookup tables.

### [Uniform Quantization](5855-uniform-quantization.md)
Uniform quantization is a lossy compression method that maps a continuous or high-precision discrete range of source data values to a limited set of discrete levels using constant spacing between levels. It solves the problem of reducing the memory footprint and bandwidth of volume data, typically achieving a 4:1 compression ratio by representing 32-bit floating-point data with 8-bit integers. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by allocating quantization levels with equal density across the entire data range, rather than adapting level density to the source data distribution or user-defined importance.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Chroma subsampling for volumetric data is a compression technique that reduces the resolution of chrominance components (U, V) while maintaining the luminance component (Y) at full resolution, thereby improving color quality at low bitrates. Unlike direct subsampling which causes significant information loss in sparse volumes, this method defines one color value for every eight adjacent corners, preserving color information more effectively. It sets itself apart from sibling quantization methods by specifically addressing the sparsity and geometric nature of volumetric data, where intensity and color are defined only for existing voxels, rather than treating the volume as a dense grid.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Differentiable quantization reduces the memory footprint of neural radiance fields by compressing grid values to single bytes during the training process, thereby preventing the quality degradation associated with post-training quantization. It distinguishes itself from standard scalar quantization by integrating the quantization operation directly into the optimization loop via a straight-through estimator, ensuring the model adapts to the discrete representation constraints.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Neural Implicit Network (INR) Compression replaces traditional lossy compression bitstreams with a compact coordinate-based neural network that maps spatial coordinates directly to field values. This approach serves as simultaneously compressed storage and a queryable data source, eliminating the need to fully decode the volume before visualization. Unlike sibling methods like [Scalar Quantization](5726-scalar-quantization.md) which operate on discrete grid values, INR compression learns a continuous function, allowing for infinite resolution queries and superior performance at high compression ratios, though it incurs significant training costs and lacks interactive rendering capabilities.

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

## Background
Quantization in volume rendering and compression has evolved alongside transform-based methods. Early work on 3D DCT coefficients utilized scalar quantization with a quantization matrix $Q(u,v,w)$ and a scaling factor $S_q$, where the quantized coefficient is given by $\hat{R}(u,v,w) = \text{round}\left(\frac{R(u,v,w)}{S_q \times Q(u,v,w)}\right)$ [S1]. The scaling factor $S_q$ allows for dynamic adaptive streaming by adjusting bitrates [S1]. The quantization matrix values are constructed based on psycho-visual experiments [S1].

In the context of neural representations, differentiable quantization was introduced to allow training with discrete representations, mapping grid values to single bytes using a straight-through estimator [S1]. For Gaussian splatting, attempts to compress positions via lattice quantization were found to introduce significant error, leading to a focus on other parameters [S5]. In voxel-based radiance fields, spatial predictive coding combined with post-finetune refinement has been shown to significantly improve compression efficiency and rendering quality [S7].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S3] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S4] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S5] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S6] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S7] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
