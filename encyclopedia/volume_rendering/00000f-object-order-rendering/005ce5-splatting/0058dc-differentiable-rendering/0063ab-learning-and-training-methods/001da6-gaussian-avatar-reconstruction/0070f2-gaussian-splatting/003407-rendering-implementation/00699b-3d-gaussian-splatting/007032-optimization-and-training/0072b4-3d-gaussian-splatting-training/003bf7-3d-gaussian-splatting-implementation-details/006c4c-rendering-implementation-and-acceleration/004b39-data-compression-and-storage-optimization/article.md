# Data Compression and Storage Optimization

Also called: 3DGS Compression, Gaussian Quantization, Codebook-Based Compression

## Overview
This family reduces the storage footprint and transmission bandwidth of 3D Gaussian Splatting (3DGS) models by compressing geometric attributes (position, scale, rotation), opacity, and view-dependent color (Spherical Harmonics coefficients) [S1, S2, unofficial]. It solves the bottleneck of large model sizes (often hundreds of MBs) which hinders deployment on mobile devices or web platforms [S1, S2, unofficial]. It is distinguished from sibling methods like [Hardware and Computational Acceleration](16214-hardware-and-computational-acceleration.md) by focusing on data representation size rather than rendering speed, and from [Occupancy Region Derivation & ϵ-Intersection Criterion](19020-occupancy-region-derivation-intersection-criterion.md) by explicitly quantizing or clustering attributes rather than just culling [S1, S2, unofficial].

## Shared principles
All methods in this family operate on the set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$ defined in [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md). The core rendering operation involves projecting these 3D primitives to 2D screen space and performing alpha blending. Acceleration relies on the following shared mechanisms:

1.  **Tiling:** Dividing the image plane into non-overlapping tiles (commonly $16 \times 16$ pixels) to localize Gaussian processing and enable coalesced memory accesses [S2, unofficial].
2.  **Sorting:** Sorting Gaussians within each tile by view-space depth (front-to-back) to enable correct alpha blending [S2, unofficial].
3.  **Culling:** Discarding Gaussians that do not overlap with the camera frustum or specific tiles to reduce enumeration overhead [S2, unofficial].
4.  **Memory Hierarchy Management:** Exploiting the distinction between global memory (VRAM), shared memory (on-chip), and host memory (CPU RAM) to minimize data movement.

## Subtopics

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
This method accelerates data transmission during training by compressing numerical attributes into ASCII strings. It solves the bottleneck of transmitting large volumes of floating-point and integer data between processing units or across networks by reducing the byte count per value. It is distinguished from sibling methods like [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md) and [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md) by its focus on transmission speed via character-level encoding rather than geometric quantization or structural simplification.

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Sensitivity-Aware Vector Clustering is a compression scheme for 3D Gaussian Splatting that reduces storage requirements by clustering Spherical Harmonic (SH) coefficients and Gaussian shape parameters (scale and rotation) into compact codebooks using sensitivity-aware vector quantization. It distinguishes itself from standard post-training quantization by deriving a sensitivity measure for each scene parameter to guide the clustering process, prioritizing the preservation of parameters that significantly affect image quality.

### [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md)
Compact 3D Gaussian Representation (C3DGS) is a compression framework that reduces the storage and computational cost of 3D Gaussian Splatting by pruning redundant Gaussians and compressing their attributes via spatial redundancy exploitation. It achieves over 25× storage reduction and faster rendering compared to standard 3DGS by replacing per-Gaussian color storage with a grid-based neural field and quantizing geometric attributes into a codebook. Unlike sibling methods that may rely on global significance scoring or 2D grid sorting, C3DGS specifically targets the spatial correlation of color and the geometric similarity of scales/rotations to minimize memory footprint.

### [Monte Carlo Specular Energy Aggregator](15439-monte-carlo-specular-energy-aggregator.md)
The Monte Carlo Specular Energy Aggregator (MC-SEA) is a compression technique that maps high-order (e.g., third-order) Spherical Harmonics (SH) radiance energy into a compact lower-order (e.g., first-order) subspace without pre-training or distillation. It preserves specular features by extracting directional moments of the specular residual via Monte Carlo projection, preserving magnitude and spatial direction without pretraining or distillation. Unlike Mobile-GS which uses distillation from a high-order teacher model, Flux-GS compresses energy directly without costly distillation or pre-training.

### Sparse Adaptive Scalable Compact Gaussian Splatting
- [S1, unofficial] A compression method that utilizes vector clustering to encode color and geometric attributes.

### 3DGS Quantization Compression
- [S2, unofficial] Reduces storage footprint of 3DGS scenes by quantizing per-Gaussian attributes and using codebooks, achieving ~10x smaller sizes without significant quality loss.
- [S2, unofficial] Per-Gaussian attribute storage: position (FP16), scales (FP16), opacity (INT8), spherical harmonic coefficients (INT8 or codebook-based).

## Background
The need for compression in 3DGS arises from the large number of Gaussians required for high-fidelity novel view synthesis, leading to model sizes that are impractical for many applications [S1, S2, unofficial]. Early work focused on pruning and quantization [S1, unofficial]. C3DGS introduced grid-based color representation and geometric codebooks [S1, unofficial]. Sensitivity-aware methods were developed to prevent quality degradation during quantization by identifying critical parameters [S1, unofficial]. MC-SEA addressed the inefficiency of high-order SH for specular highlights by aggregating energy into lower-order representations [S1, unofficial]. Recent surveys highlight various techniques including VQ-NeRF, LightGaussian, and CompGS as part of the broader effort to make 3DGS production-ready [S2, unofficial].

## Sources
[S1] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S2] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
