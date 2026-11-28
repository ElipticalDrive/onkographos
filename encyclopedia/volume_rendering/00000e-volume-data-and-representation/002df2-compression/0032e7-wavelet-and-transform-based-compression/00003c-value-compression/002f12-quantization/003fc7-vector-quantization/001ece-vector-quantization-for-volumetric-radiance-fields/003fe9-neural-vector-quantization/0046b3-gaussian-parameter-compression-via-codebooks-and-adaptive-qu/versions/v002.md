# Gaussian Parameter Compression via Codebooks and Adaptive Quantization

Also called: Codebook-based Quantization, Gaussian Parameter Compression, Memory-Efficient 3D GS via Compression, Quantized Gaussian Properties, Vector Quantization (VQ) for Gaussian Attributes

## Overview
This method addresses the excessive memory footprint of 3D Gaussian Splatting by compressing Gaussian attributes (opacity, scales, rotations, and spherical harmonic coefficients) into compact codebooks using vector quantization (VQ) and adaptive quantization modules [S1, S2]. It distinguishes itself from sibling methods by combining sensitivity-aware clustering for codebook generation with entropy-constrained or adaptive quantization steps to minimize storage while preserving visual fidelity, achieving compression ratios of 10–20× [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** A set of 3D Gaussians, each defined by 59 attributes: mean position (3), scale (3), rotation (4, quaternion), opacity (1), and spherical harmonic (SH) coefficients (49, for degree 3) [S2].
- **Output:** A set of codebook indices and associated metadata (e.g., anchor references, latent vectors) that reconstruct the Gaussian properties with minimal error [S1, S2].

**Data Structures:**
1. **Codebook:** A lookup table storing representative vectors for specific attribute groups (e.g., color, covariance) [S1, S2].
2. **Anchor Structure:** A reference to a cluster center or "local anchor" obtained by clustering neighboring Gaussians [S1, S2].
3. **Latent Vector:** A compressed representation per attribute, decoded by an MLP [S2].
4. **Quantization Steps:** Dynamic parameters selected by an Adaptive Quantization Module (AQM) [S2].

**Numbered Steps:**
1. **Attribute Categorization:** Divide Gaussian attributes into "key" and "unimportant" groups based on reconstruction impact [S2]. Key attributes include opacity, scales, Euler angles, and 0-degree SH coefficients [S2]. Unimportant attributes include SH coefficients with degree > 0 [S2].
2. **Entropy Reduction:** For key attributes, apply Recursive Adaptive Hierarchical Transform (RAHT) to reduce entropy before quantization [S2].
3. **Sensitivity-Aware Codebook Generation:** Compute sensitivity measures for each attribute type to determine importance, then cluster Gaussian properties into compact codebooks [S1].
4. **Vector Quantization (VQ):** Compress unimportant attributes via standard VQ using the generated codebooks [S2]. For covariance and color, use entropy-constrained VQ to select codebook entries that minimize the rate-distortion trade-off [S2].
5. **Adaptive Quantization (Anchor-Based):** Associate each Gaussian with an anchor structure [S1, S2]. Use an Adaptive Quantization Module (AQM) to dynamically select quantization steps for the anchor attributes based on their probability distribution [S1, S2].
6. **Latent Vector Encoding:** Encode attributes (rotation, view-dependent color, opacity) into a latent vector [S2]. Decode the latent vector using a Multilayer Perceptron (MLP) [S2]. During training, preserve a continuous latent approximation and use a Straight-Through Estimator (STE) to round the values for forward propagation, ensuring differentiability [S2].

**Formulas:**
Not given in the sources.

**Parameters:**
- **Codebook Size:** Not given in the sources.
- **SH Degree:** Typically degree 3 (49 coefficients) [S2].
- **Compression Ratio:** 10–20× [S1].

**Edge Cases and Pitfalls:**
- **Differentiability:** Standard VQ is non-differentiable, requiring STE or latent vector approaches for gradient-based optimization [S2].
- **Sensitivity Measurement:** Incorrect sensitivity measures may lead to over-compression of important attributes (e.g., 0-degree SH) [S1, S2].

## Subtopics

### Canonical Gaussian Compression (CCA)
A module that compresses canonical Gaussian attributes (appearance and geometry) using autoencoding, quantization, and residual codebooks to reduce storage footprint [S3]. It applies autoencoding to the canonical attributes, then applies quantization using residual codebooks [S3]. Choose it when compressing dynamic 4D scenes to enable a storage-efficient deformation pathway [S3].

### Compression of the Motion State
Compresses the 8-dimensional temporal vector $[v_i, a_i, t'_i, \rho_i]$ using sub-vector codebooks fitted only on dynamic Gaussians [S4, unofficial]. Static Gaussians share an index that decodes to an exact zero-motion state, consuming no temporal codebook capacity [S4, unofficial]. It stores compact codebook indices, entropy codes the index streams, and stores the committed binary partition as a packed bitmask [S4, unofficial]. Temporal quantization is performed after static quantization, followed by a separate refinement stage for temporal codebooks [S4, unofficial]. A pitfall is backward compatibility: a checkpoint without explicit gate bits is treated as fully dynamic [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Training time increases due to codebook generation, RAHT application, and MLP decoding [S1, S2].
- **Memory Complexity:** Storage is reduced by 10–20× compared to uncompressed 3D GS [S1].
- **Measured Performance:** Achieves 10–20× storage reduction ratios across various compression techniques [S1].
- **Scaling:** Scene-agnostic, reusable codebooks and quantization-aware training protocols are needed to scale memory efficiency during training [S1].
- **Strengths:** Significant memory savings while preserving visual fidelity through sensitivity-aware quantization [S1, S2].
- **Limitations:** The trade-off between compression efficiency and visual fidelity remains an open problem [S1]. Deployment on edge devices is challenging due to tight memory budgets [S1].
- **When to Use:** When memory is constrained (e.g., edge devices) and high compression ratios are needed [S1].
- **When Not to Use:** When real-time training speed is critical and compression overhead is prohibitive [S1].

## Background
- **Origin:** Developed as part of the broader effort to compress 3D Gaussian Splatting, with key contributions from Niedermayr et al. [183], HAC [31, 9], RDOGaussian [49], and EAGLES [16] [S1, S2].
- **History:** Emerged as 3D GS models grew too large for practical deployment, prompting research into codebook-based and adaptive quantization techniques [S1, S2].
- **Usage:** Used in memory-efficient 3D GS implementations for edge devices and large-scale scene representation [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S3] CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3721136, fetched 2026-10-27
[S4] Mobile-4DGS: Unified Static-Dynamic Real-time Mobile Gaussian Splatting, https://arxiv.org/abs/2610.05289, fetched 2026-10-29 (unofficial)
