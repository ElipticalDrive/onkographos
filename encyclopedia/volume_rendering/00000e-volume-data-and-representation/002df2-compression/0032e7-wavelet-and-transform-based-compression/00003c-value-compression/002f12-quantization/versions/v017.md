# Quantization

Also called: Quantization with Matrix

## Overview
Quantization reduces the precision of volumetric data coefficients or values to decrease memory footprint and bandwidth requirements, enabling real-time rendering of large datasets [S1, S2, S7]. It differs from its siblings, such as [Vector Quantization](907-vector-quantization.md) and [Scalar Quantization](5726-scalar-quantization.md), by encompassing a broader range of techniques including uniform mapping, chroma subsampling, and differentiable integration into neural field training, rather than relying solely on fixed mathematical bases or simple block-wise linear interpolation [S1, S2, S5, S7].

## Shared principles
The family shares the core mechanism of mapping a continuous or high-precision input domain to a discrete, lower-precision set of levels. This process is inherently lossy, introducing quantization error that must be managed through careful selection of step sizes, codebooks, or bit depths [S1, S2, S7]. The primary goal is to balance compression ratio against reconstruction fidelity, often measured by Mean Squared Error (MSE) or Peak Signal-to-Noise Ratio (PSNR) [S1, S8]. Quantization is frequently followed by entropy encoding to further compress the discrete indices or residuals [S1, S7].

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
Maps $n$-dimensional input vectors to a single fixed-length index referencing a pre-defined or learned codebook. It exploits spatial redundancy in local data blocks, achieving better compression ratios for data with high spatial coherence than fixed-basis methods. Its primary advantage is extremely fast decompression, often implemented via GPU texture lookups, making it suitable for real-time rendering of large or time-varying volumes.

### [Scalar Quantization](5726-scalar-quantization.md)
A random-access decompression method that reconstructs voxel values on-the-fly without storing fully decompressed data. It operates on localized blocks, computing local minimum and maximum bounds to define a linear interpolation range, allowing for efficient GPU implementation. Unlike vector quantization, it does not rely on codebooks, making it simpler but potentially less accurate for complex data distributions.

### [Uniform Quantization](5855-uniform-quantization.md)
Maps a continuous input domain to a discrete set of levels with constant spacing, using a limited number of bits. It typically achieves a 4:1 compression ratio by representing 32-bit floats with 8-bit integers. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by using fixed step sizes, which makes it simple to implement but prone to losing features in data with wide dynamic ranges.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Reduces the resolution of chrominance components (U, V) while maintaining luminance (Y) at full resolution, defining one color value for every eight adjacent corners. It sets itself apart by addressing spatial sparsity of volumetric voxels, preventing the complete loss of color information (grey objects) that occurs with non-subsampled approaches at low bitrates.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Reduces the memory footprint of neural radiance fields by mapping continuous grid values to single-byte discrete levels during the training process. Unlike post-training quantization, it integrates quantization into the optimization loop via a straight-through estimator, ensuring the model adapts to quantization noise via gradient flow.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Maps spatial coordinates directly to field values using a coordinate-based neural network, serving as simultaneously compressed storage and a queryable data source. It replaces discrete coefficient storage with a continuous parametric function, enabling infinite resolution at the cost of significant training time and lack of interactive rendering capabilities in early implementations.

### µ-law Feature Encoding
A non-linear transformation applied to raw sensor values before computing pixel-wise MSE loss, compressing dynamic range to emphasize shadow detail [S2]. Raw 14-bit sensor values $x$ are transformed as $f(x) = \log(1 + \mu \cdot x/w) / \log(1 + \mu)$, with $\mu = 5000$ and $w = 2^{14} - 1 = 16,383$ (the white level of 14-bit data) [S2].

### Lossy Quantization for Coarsened Patches
Lossy compression for coarsened patches in the bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$), with $q$ a fixed constant usually $< 1$ [S3]. A single unified error is recomputed and nested for each patch. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation. Shift $k$ is calculated as: $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else $0$. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$. Corners are stored at full precision. Quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S3].

### HSV Colour Sorting
A technique to preserve colour quality during DXT1 compression by reordering colours based on HSV values [S4, unofficial]. The effectiveness of HSV sorting depends on which component (H, S, or V) is the most frequent in the voxel data’s set of colours [S4, unofficial].

### Interpolation Symmetry Quantization
A method to improve the accuracy of quantized weights by exploiting the symmetry of linear interpolation between segments [S5]. Linear interpolation $(1 - w)s_i + w s_j$ is equivalent to $(j, i, 1 - w)$. The method quantizes both $w$ and $1 - w$ using the codebook generated by the scalar quantizer and keeps the representation resulting in the lower error. It stores either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream [S5].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into the training process [S6]. It fails to compress positions without introducing significant rendering error, identified as a future research direction for end-to-end positional compression [S6].

### Quantization and Entropy Encoding/Decoding Pipeline
The compression procedure involving quantization followed by entropy encoding (range-coder) for compression, and entropy decoding for decompression [S7]. Compressed data is converted to non-negative values to enable compression into int8 format, then reverted to original range during decompression [S7]. The reconstruction formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $E$ is entropy encoder, $D$ is entropy decoder, $Q$ is quantization, and $q$ is the quantization parameter [S7]. The parameter $q$ expands the data range to enhance quantization precision; adjusting $q$ trades off reconstruction quality and model storage size [S7].

### Post-finetune Refinement
A refinement step that uses a small number of bits to enhance the reconstruction quality of important voxels, compensating for losses introduced by coarse quantization [S8]. Ablation study on Synthetic-NeRF dataset shows disabling post-finetune causes a -1.72 dB $\Delta$PSNR, -0.07 $\Delta$SSIM, and -0.079 MB $\Delta$Size [S8]. Disabling post-finetune saves bits but inevitably deteriorates rendering quality, confirming its necessity for maintaining RD performance [S8].

## Background
Quantization in volume rendering has evolved from simple uniform mapping of scalar fields to complex, differentiable techniques integrated into neural field training. Early methods focused on reducing the bit-depth of 3D DCT coefficients using scalar quantization and psycho-visual quantization matrices to enable dynamic adaptive streaming [S1]. Subsequent developments introduced block-based scalar quantization for random-access decompression, allowing rendering directly from compressed data without full decompression [S2][S3]. With the rise of Neural Radiance Fields (NeRFs) and 3D Gaussian Splatting, quantization has become a critical component for reducing the massive memory footprint of learned representations, leading to differentiable quantization strategies that allow gradients to flow through the quantization operation during training [S1, S7, S8]. Techniques like chroma subsampling and HSV sorting have also been adapted from image compression to handle the specific sparsity and color requirements of volumetric data [S4, unofficial].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S3] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S4] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S5] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S6] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S7] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S8] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
