# Quantization

Also called: Quantization with Matrix

## Overview
Quantization reduces the precision of volumetric data coefficients or values to lower storage and bandwidth requirements, serving as the primary lossy step in transform-based compression pipelines. It sets itself apart from sibling methods like [Vector Quantization](907-vector-quantization.md) or [Chroma Subsampling](6512-chroma-subsampling.md) by operating on individual scalar coefficients or values rather than mapping blocks to codebooks or rearranging color channels, though it often precedes entropy coding in the compression chain.

## Shared principles
This family shares the fundamental mechanism of mapping a continuous or high-precision input domain to a discrete, lower-precision output domain. While specific implementations vary (scalar, vector, neural), the core mathematical operation involves a scaling factor and a rounding or indexing function. In transform-based contexts, this is often applied to coefficients $C$ derived from the volume $V$, where the quantized value $\hat{C}$ is derived from the original coefficient $C$ via a quantization matrix or step size [S1]. The process is inherently lossy, introducing quantization error that is managed through bitrate control parameters or post-processing refinement [S6, S7].

## Subtopics

### [Vector Quantization](907-vector-quantization.md)
Maps $n$-dimensional blocks to codebook indices. Chosen for real-time GPU rendering where decompression speed is critical.

### [Scalar Quantization](5726-scalar-quantization.md)
Maps individual voxel values or coefficients to local indices based on block bounds. Chosen for random-access decompression without codebooks.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Reduces chrominance resolution while keeping luminance full-res. Chosen for color volume compression to exploit human visual sensitivity.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Integrates byte-level quantization into the training loop of neural fields. Chosen to prevent quality degradation from post-training quantization.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Quantizes the weights of a neural network representing the volume. Chosen to replace explicit grid storage with a compact, queryable model.

### [Unquantization Functions](13872-unquantization-functions.md)
Defines the inverse mapping from integer indices to floating-point values. Chosen as the reconstruction step in block-based formats like BC6H.

### Lossy Quantization for Coarsened Patches
A method for compressing coarsened patches in a bintree hierarchy where quantization error $q$ is proportional to coarsening error $c$ ($q = qc$, with $q < 1$) [S2]. Quantization levels are restricted to power-of-two multiples of vertical resolution $\Delta z$ for efficient bit-shift implementation [S2]. The shift $k$ is calculated as $k = \lceil \log_2(\epsilon_q / \Delta z) \rceil + 1$ if $\epsilon_q \ge \Delta z$, else $0$ [S2]. All finest level vertices have zero shift. Vertices are classified as boundary (corners $C$, legs $L$, hypotenuse $H$) and interior $I$; corners are stored at full precision, and quantization of original height values (not residuals) ensures adjacent patches agree on shared boundaries [S2].

### HSV Colour Sorting
A technique to preserve color quality during DXT1 compression by reordering colors based on HSV values before compression [S3, unofficial]. The effectiveness depends on which HSV component (H, S, or V) is the most frequent in the voxel data's color set [S3, unofficial].

### Interpolation Symmetry Quantization
A method to improve quantization accuracy by exploiting the symmetry of linear interpolation between segments [S4]. It quantizes both $w$ and $1 - w$ using the scalar quantizer's codebook, keeping the representation with lower error, and stores either $(i, j, w)$ or $(j, i, 1 - w)$ in the output stream [S4].

### Position Lattice Quantization
An attempted method to compress Gaussian positions by quantizing them to a lattice structure and embedding positional constraints into training [S5]. It fails to compress positions without introducing significant rendering error and is identified as a future research direction [S5].

### Quantization and Entropy Encoding/Decoding Pipeline
A pipeline where quantization is followed by entropy encoding (e.g., range-coder) [S6]. To enable int8 compression, data is converted to non-negative values and reverted during decompression [S6]. The formula is $\hat{x} = \frac{D(E(Q(q \cdot x) - Q(q \cdot \min(x)))) + Q(q \cdot \min(x))}{q}$, where $q$ is the quantization parameter that expands the data range to enhance precision [S6].

### Post-finetune Refinement
A step using a small number of bits to enhance reconstruction quality of important voxels, compensating for coarse quantization losses [S7]. Disabling it causes -1.72 dB $\Delta$PSNR, -0.07 $\Delta$SSIM, and -0.079 MB $\Delta$Size on the Synthetic-NeRF dataset [S7].

## Background
Quantization in volume rendering has evolved from simple scalar quantization of DCT coefficients [S1] to differentiable methods integrated into neural field training [S1, S5, S6]. Early methods focused on psycho-visual quantization matrices for 3D DCT coefficients [S1], while recent approaches address the specific needs of neural representations, such as 3D Gaussian Splatting [S5] and NeRFs [S6, S7]. The field also includes specialized techniques for color data [S3, unofficial] and terrain-like structures [S2].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
[S3] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
[S4] The Mixture Graph-A Data Structure for Compressing, Rendering, and Querying Segmentation Histograms, https://doi.org/10.1109/tvcg.2020.3030451, fetched 2026-10-26
[S5] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S6] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S7] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
