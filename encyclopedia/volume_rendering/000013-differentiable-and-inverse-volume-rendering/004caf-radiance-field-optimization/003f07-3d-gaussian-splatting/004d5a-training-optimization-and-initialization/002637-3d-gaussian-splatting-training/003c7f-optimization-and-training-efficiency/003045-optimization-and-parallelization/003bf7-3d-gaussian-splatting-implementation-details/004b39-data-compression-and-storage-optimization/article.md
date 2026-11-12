# Data Compression and Storage Optimization

Also called: 3DGS Compression, Gaussian Splatting Storage Optimization

## Overview
This family of methods addresses the excessive memory footprint and storage requirements of 3D Gaussian Splatting (3DGS) models, which typically consist of millions of Gaussian primitives with high-dimensional attributes [S1, S2, S3]. While standard 3DGS stores explicit per-Gaussian parameters, these optimization techniques reduce redundancy through various mechanisms: ASCII-based encoding for training I/O efficiency [S1], sensitivity-aware vector clustering for quantization [S2], and hybrid representations using hash grids and codebooks for inference [S3]. They differ from sibling methods like [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) by focusing on data representation rather than rendering throughput, and from [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by employing structured compression rather than simple pruning.

## Shared principles
All methods in this family aim to reduce the size of the scene representation without significantly degrading rendering quality. They operate on the core 3DGS attributes: position, opacity, scale, rotation, and view-dependent color (Spherical Harmonics, SH). The shared goal is to replace high-precision, per-primitive storage with compact indices, shared structures, or lossy encodings. Common techniques include vector quantization, codebook-based storage, and spatial hashing. The methods assume the standard 3DGS differentiable rasterization pipeline remains unchanged, with the compression applied to the input data structures before rendering or during the training loop.

## Subtopics

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
Compresses floating-point and integer values into ASCII strings to accelerate training I/O. It targets the bottleneck of data transmission rather than final model size, achieving ~6% training speed improvement.

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Reduces storage by clustering SH coefficients and geometric features into codebooks using a sensitivity-aware k-Means algorithm. It prioritizes parameters that significantly impact image quality, differing from standard post-training quantization by incorporating gradient-based sensitivity measures.

### [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md)
Achieves >25x compression by replacing per-Gaussian attributes with a shared hash-based grid for colors and a discrete codebook for geometry. It eliminates redundant primitives via volume-based masking, setting it apart by integrating spatial redundancy exploitation directly into the representation.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [G]. The method's high fidelity comes at the cost of large model sizes, often exceeding several gigabytes for complex scenes. Subsequent research focused on optimizing this storage. ASCII-based compression was proposed to speed up the training process by reducing I/O overhead [S1]. Sensitivity-aware clustering was developed to enable efficient quantization by identifying critical parameters [S2]. C3DGS introduced a hybrid approach combining hash grids and codebooks to drastically reduce memory usage for real-time rendering [S3]. These methods are used in applications requiring efficient storage and transmission of 3D scenes, such as web-based viewers and mobile AR.

## Sources
[S1] ASCII-Based Data Compression for 3D Gaussian Splatting Training, https://github.com/3DGS-Compression/ASCII-3DGS, 2024
[S2] Sensitivity-Aware Vector Clustering for 3D Gaussian Splatting, https://github.com/3DGS-Compression/Sensitivity-Clustering, 2024
[S3] Compact 3D Gaussian Representation, https://github.com/3DGS-Compression/C3DGS, 2024
