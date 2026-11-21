# Data Compression and Storage Optimization

Also called: 3DGS Compression, Gaussian Quantization, Codebook-based Compression

## Overview
Data Compression and Storage Optimization addresses the excessive memory footprint and storage requirements of 3D Gaussian Splatting (3DGS) scenes, which often exceed 1GB for real-world datasets due to per-Gaussian attributes [S1, S2, unofficial]. It reduces this footprint by replacing continuous per-Gaussian parameters with discrete codebooks, hash-based fields, or quantized representations, achieving compression ratios from 10× to over 25× while maintaining reconstruction quality [S1, S2, unofficial]. Unlike standard 3DGS, which treats each Gaussian as an independent primitive with full precision, these methods exploit spatial redundancy, sensitivity-aware clustering, or neural field approximations to share or compress attributes [S1, unofficial].

## Shared principles
The family shares the goal of reducing the storage size of $N$ Gaussian primitives $\{G_i\}$, where each $G_i$ is defined by position $\mu_i$, scale $\hat{s}_i$, rotation $r_i$ (quaternion), opacity $o_i$, and spherical harmonic (SH) coefficients $c_i$ [S1, S2, unofficial].

1.  **Codebook-Based Quantization:** Most methods replace continuous attributes (typically SH coefficients for color, and scale/rotation for geometry) with indices into a finite codebook $C$ [S1, S2, unofficial]. The storage cost shifts from $N \times D$ floats to $N \times \text{bits}(\text{index}) + |C| \times D$ floats, where $|C| \ll N$ [S1, unofficial].
2.  **Sensitivity-Awareness:** High-impact parameters are preserved in higher precision or excluded from aggressive clustering to prevent visual artifacts [S1, unofficial]. Sensitivity $S(p)$ is often defined as the gradient of the image energy with respect to the parameter, normalized by pixel count [S1, unofficial].
3.  **Spatial Redundancy:** Neighboring Gaussians often share similar geometric or color properties. Methods exploit this via shared neural fields (hash grids) or clustering based on spatial coherence (e.g., Z-order/Morton order) [S1, S2, unofficial].
4.  **Quantization-Aware Training (QAT):** To recover quality lost during compression, fine-tuning is performed where quantization is simulated during the forward pass but gradients are computed on continuous parameters during the backward pass [S1, S2, unofficial].

## Subtopics

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
A method for optimizing data transmission speed during training by encoding numerical attributes into ASCII strings. It maps integers in $[0, 95]$ to ASCII characters 32–126 and floats to a base-94 integer character plus a delimiter `~` and decimal string. It yields modest speed improvements (~6%) but suffers from slow matrix traversal.

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Reduces storage by clustering SH coefficients and shape parameters into codebooks using sensitivity-aware k-means. It calculates sensitivity as the normalized gradient of image energy, excluding high-sensitivity vectors from clustering. It uses weighted distance $D(x, c_k) = S(x) \|x - c_k\|_2^2$ for centroid updates and applies entropy coding (DEFLATE) after Z-ordering.

### [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md)
Reduces memory by replacing per-Gaussian color with a shared hash-based neural field (Instant NGP) and geometry with a discrete codebook. It uses a learnable mask to prune redundant Gaussians and achieves over 25× compression. It exploits spatial redundancy by mapping positions to color features and quantizing scale/rotation.

### [Monte Carlo Specular Energy Aggregator](15439-monte-carlo-specular-energy-aggregator.md)
Compresses third-order SH to first-order SH by aggregating specular residuals via Monte Carlo sampling. It computes scalar energy magnitude $E_{mag}$ and directional tensor $E_{dir}$ from the residual between third- and second-order SH, then uses an MLP to predict first-order coefficients. It avoids pre-training distillation, unlike Mobile-GS.

### Sparse Adaptive Scalable Compact Gaussian Splatting
A compression method that utilizes vector clustering to encode color and geometric attributes into two codebooks, employing a sensitivity-aware k-means method to prioritize important parameters [S1, unofficial].

### 3DGS Quantization Compression
Reduces storage footprint by quantizing per-Gaussian attributes and using codebooks, achieving ~10× smaller sizes without significant quality loss [S2, unofficial]. The layout stores position in FP16, scales in FP16, opacity in INT8, and SH coefficients in INT8 or via codebooks [S2, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [G]. The high storage cost of storing full SH coefficients and continuous parameters for millions of Gaussians led to immediate research into compression. Early methods like LightGaussian combined pruning with INT8 quantization [S2, unofficial]. Subsequent works like C3DGS, SASCGS, and EAGLES explored residual vector quantization, sensitivity-aware clustering, and full attribute quantization to reduce scene sizes by 10× to 25× [S1, unofficial]. Recent approaches like MC-SEA focus on reducing SH order for mobile rendering without distillation [S1, unofficial].

## Sources
[S1] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S2] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
