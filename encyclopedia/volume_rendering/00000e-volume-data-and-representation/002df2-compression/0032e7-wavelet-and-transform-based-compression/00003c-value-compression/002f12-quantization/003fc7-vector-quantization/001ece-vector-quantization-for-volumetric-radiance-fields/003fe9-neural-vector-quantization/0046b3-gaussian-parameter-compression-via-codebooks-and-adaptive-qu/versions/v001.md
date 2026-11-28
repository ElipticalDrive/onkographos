# Gaussian Parameter Compression via Codebooks and Adaptive Quantization

Also called: Codebook-based Quantization, Gaussian Parameter Compression, Memory-Efficient 3D GS via Compression, Quantized Gaussian Properties, Vector Quantization (VQ) for Gaussian Attributes

## Overview
This method addresses the excessive memory footprint of 3D Gaussian Splatting by compressing Gaussian attributes (opacity, scales, rotations, and spherical harmonic coefficients) into compact codebooks using vector quantization (VQ) and adaptive quantization modules [S1, S2]. It distinguishes itself from sibling methods by combining sensitivity-aware clustering for codebook generation with entropy-constrained or adaptive quantization steps to minimize storage while preserving visual fidelity, achieving compression ratios of 10–20× [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** A set of 3D Gaussians, each defined by 59 attributes: mean position (3), scale (3), rotation (4, quaternion), opacity (1), and spherical harmonic (SH) coefficients (49, for degree 3) [S2].
- **Output:** A set of codebook indices and associated metadata (e.g., anchor references, latent vectors) that reconstruct the Gaussian properties with minimal error [S1, S2].

**Data Structures:**
1.  **Codebook:** A lookup table storing representative vectors for specific attribute groups (e.g., color, covariance) [S1, S2].
2.  **Anchor Structure:** A reference to a cluster center or "local anchor" obtained by clustering neighboring Gaussians [S1, S2].
3.  **Latent Vector:** A compressed representation per attribute, decoded by an MLP [S2].
4.  **Quantization Steps:** Dynamic parameters selected by an Adaptive Quantization Module (AQM) [S2].

**Numbered Steps:**
1.  **Attribute Categorization:** Divide Gaussian attributes into "key" and "unimportant" groups. Key attributes include opacity, scales, Euler angles, and 0-degree SH coefficients. Unimportant attributes include SH coefficients with degree > 0 [S2].
2.  **Entropy Reduction (Key Attributes):** For key attributes, apply Recursive Adaptive Hierarchical Transform (RAHT) to reduce entropy before quantization [S2].
3.  **Sensitivity-Aware Codebook Generation:** Compute sensitivity measures for each attribute type to determine importance. Use these measures to cluster Gaussian properties into compact codebooks [S1, S2].
4.  **Vector Quantization (VQ):**
    -   For unimportant attributes (e.g., high-degree SH), compress via standard VQ using the generated codebooks [S2].
    -   For covariance and color, use entropy-constrained VQ to select codebook entries that minimize the rate-distortion trade-off [S2].
5.  **Adaptive Quantization (Anchor-Based):**
    -   Associate each Gaussian with an anchor (cluster center) [S1, S2].
    -   Use an Adaptive Quantization Module (AQM) to dynamically select quantization steps for the anchor attributes based on their probability distribution [S1, S2].
6.  **Latent Vector Encoding (Alternative/Complementary Approach):**
    -   Encode attributes (rotation, view-dependent color, opacity) into a latent vector [S2].
    -   Decode the latent vector using a Multilayer Perceptron (MLP) [S2].
    -   During training, preserve a continuous latent approximation and use a Straight-Through Estimator (STE) to round the values for forward propagation, ensuring differentiability [S2].

**Formulas:**
-   **Rate-Distortion Optimization (Entropy-Constrained VQ):** Minimize $L = D + \lambda R$, where $D$ is distortion, $R$ is rate (bits), and $\lambda$ is the Lagrange multiplier [S2].
-   **Straight-Through Estimator (STE):** The gradient flows through the rounding operation $Q(x)$ as if it were the identity function: $\frac{\partial L}{\partial x} = \frac{\partial L}{\partial Q(x)} \cdot 1$ [S2].

**Parameters:**
-   **Codebook Size:** Not given in the sources.
-   **SH Degree:** Typically degree 3 (49 coefficients) [S2].
-   **Compression Ratio:** 10–20× [S1].

**Edge Cases and Pitfalls:**
-   **Differentiability:** Standard VQ is non-differentiable. The STE or latent vector approach is required for gradient-based optimization [S2].
-   **Sensitivity Measurement:** Incorrect sensitivity measures may lead to over-compression of important attributes (e.g., 0-degree SH) [S1, S2].
-   **Anchor Drift:** If anchors are not updated, quantization error may accumulate [S1].

## Operating characteristics
-   **Time Complexity:** Training time increases due to codebook generation, RAHT, and MLP decoding [S1, S2]. Inference time depends on the decoder complexity (MLP vs. direct lookup) [S2].
-   **Memory Complexity:** Storage is reduced by 10–20× compared to uncompressed 3D GS [S1].
-   **Strengths:** Significant memory savings; preserves visual fidelity through sensitivity-aware quantization [S1, S2].
-   **Limitations:** Trade-off between compression efficiency and visual fidelity remains an open problem [S1]. Deployment on edge devices is challenging due to memory budgets [S1].
-   **When to Use:** When memory is constrained (e.g., edge devices) and high compression ratios are needed [S1].
-   **When Not to Use:** When real-time training speed is critical and compression overhead is prohibitive [S1].

## Background
-   **Origin:** Developed as part of the broader effort to compress 3D Gaussian Splatting, with key contributions from Niedermayr et al. [183], HAC [31, 9], RDOGaussian [49], and EAGLES [16] [S1, S2].
-   **History:** Emerged as 3D GS models grew too large for practical deployment, prompting research into codebook-based and adaptive quantization techniques [S1, S2].
-   **Usage:** Used in memory-efficient 3D GS implementations for edge devices and large-scale scene representation [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
