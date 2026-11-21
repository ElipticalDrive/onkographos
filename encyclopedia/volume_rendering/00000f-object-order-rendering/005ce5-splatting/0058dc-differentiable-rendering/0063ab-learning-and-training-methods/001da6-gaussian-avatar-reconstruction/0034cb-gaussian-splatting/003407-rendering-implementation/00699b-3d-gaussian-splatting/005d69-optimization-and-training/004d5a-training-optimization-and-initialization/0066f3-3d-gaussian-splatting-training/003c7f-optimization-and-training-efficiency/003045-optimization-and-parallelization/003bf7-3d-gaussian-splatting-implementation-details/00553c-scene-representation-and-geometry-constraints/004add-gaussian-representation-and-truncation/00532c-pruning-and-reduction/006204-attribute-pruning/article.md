# Attribute Pruning

Also called: SH Degree Adjustment, Spherical Harmonics Pruning

## Overview
Attribute pruning selectively reduces the storage and computational burden of per-Gaussian attributes, primarily spherical harmonics (SH) coefficients, to optimize memory usage and rendering efficiency in 3D Gaussian Splatting [S1]. Unlike sibling methods that focus on geometric redundancy or quantization, this method dynamically tailors the precision of view-dependent color representation to local scene complexity, allowing high-fidelity detail where needed while using minimal attributes in simpler regions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each Gaussian has associated SH coefficients $\mathbf{c}_i$ representing view-dependent color.
*   **Output:** A pruned set of Gaussians with reduced attribute storage, maintaining visual fidelity while lowering memory footprint.

**Data Structures:**
Each Gaussian $G_i$ stores SH coefficients. In standard 3DGS, SH up to degree 3 is used for RGB data, resulting in 48 coefficients (16 per channel). This consumes 81% (48 out of 59 attributes) of the storage size for a single Gaussian [S1].

**Steps:**
1.  **Evaluate Scene Complexity:** Analyze the local complexity of the scene or the importance of view-dependent effects for each Gaussian.
2.  **Determine SH Degree:** Assign a SH degree to each Gaussian based on the evaluation:
    *   For less complex regions, assign a single degree (basic RGB color, typically degree 0) [S1].
    *   For complex regions requiring detailed representation, assign higher degrees (e.g., degree 1, 2, or 3) [S1].
3.  **Adjust Coefficients:** Reduce the number of stored SH coefficients for Gaussians assigned lower degrees. This involves truncating higher-order coefficients that are not needed for the assigned precision [S1].
4.  **Render:** Use the adjusted SH coefficients during the splatting process to compute view-dependent colors [S1].

**Formulas:**
The number of SH coefficients $C$ for degree $L$ and color channels $K$ is given by:
$$ C = (L+1)^2 \times K $$
For RGB ($K=3$) and degree 3 ($L=3$):
$$ C = (3+1)^2 \times 3 = 16 \times 3 = 48 $$
This results in 48 coefficients per Gaussian [S1].

**Parameters:**
*   **SH Degree:** Typically ranges from 0 (basic RGB) to 3 (high detail) [S1].
*   **Storage Reduction:** Degree 3 SH consumes 81% of Gaussian storage attributes [S1].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Aggressively reducing SH degree in complex regions may lead to visible artifacts or loss of fine detail [S1].
*   **Uniform vs. Adaptive:** Applying uniform SH degree across the entire scene is inefficient; adaptive pruning is required for optimal balance [S1].
*   **Alternative Representations:** Completely forgoing SH in favor of NeRF-like MLPs or classical shading can reduce attributes further but requires custom, potentially slower rendering pipelines [S1].

## Subtopics

### Reduced3DGS
Reduced3DGS is a specific implementation of attribute pruning that addresses large storage sizes by reducing primitive count, adjusting SH bands, and quantizing attributes [S1]. It introduces a scale- and resolution-aware redundant primitive removal method that extends opacity-based pruning by incorporating a redundancy score to identify regions with many low-impact primitives [S1]. For SH adjustment, it evaluates color consistency across views and reduces higher-order SH bands when view-dependent effects are minimal [S1]. Additionally, it uses K-means clustering to develop a codebook and applies 16-bit half-float quantization to remaining uncompressed floating-point values [S1].

## Operating characteristics
*   **Memory Efficiency:** Significantly reduces storage size by lowering the number of SH coefficients per Gaussian. Degree 3 SH accounts for 81% of Gaussian attributes; reducing this degree yields substantial savings [S1].
*   **Computational Efficiency:** Fewer coefficients to evaluate during rendering can improve computational efficiency, especially on memory-constrained devices [S1].
*   **Visual Fidelity:** Maintains high visual fidelity in complex regions while accepting lower precision in simpler areas [S1].
*   **Limitations:** Requires careful tuning of SH degree assignment to avoid artifacts. Replacing SH entirely with other methods (e.g., MLPs) may introduce rendering overhead [S1].
*   **Use Cases:** Ideal for applications where memory and computational resources are limited, such as VR headsets or mobile devices, and for scenes with varying levels of geometric and photometric complexity [S1].

## Background
Attribute pruning emerged as a response to the high memory footprint of 3D Gaussian Splatting, particularly due to the storage requirements of high-degree spherical harmonics [S1]. The method builds on the observation that not all parts of a scene require high-fidelity view-dependent color representation [S1]. It is part of a broader effort to compress and optimize 3DGS for real-time applications and constrained hardware [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
