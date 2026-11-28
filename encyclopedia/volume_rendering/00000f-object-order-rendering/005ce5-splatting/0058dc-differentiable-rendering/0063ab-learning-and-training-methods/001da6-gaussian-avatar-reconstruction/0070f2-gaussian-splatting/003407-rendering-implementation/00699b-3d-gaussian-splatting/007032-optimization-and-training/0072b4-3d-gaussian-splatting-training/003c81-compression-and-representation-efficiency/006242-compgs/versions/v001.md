# CompGS

Also called: Efficient 3D Scene Representation via Compressed Gaussian Splatting

## Overview
CompGS is a compression method for 3D Gaussian Splatting that reduces representation size by introducing a hybrid primitive structure consisting of anchor and coupled primitives [S1]. It differs from sibling methods like [Mini-Splatting](7850-mini-splatting.md) and [C3DGS](9184-c3dgs.md) by explicitly modeling the dependency between primitives to predict coupled attributes from anchors, rather than relying solely on global quantization or structural pruning [S1].

## How it works
**Inputs and Outputs:**
*   Input: A set of trained 3D Gaussians with attributes $\{\mathbf{p}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_{i}\}$ [S1].
*   Output: A compressed bitstream representing the scene [S1].

**Data Structures:**
The method partitions the Gaussian set into two groups:
1.  **Anchor Primitives:** Stored with full precision or high-fidelity quantization.
2.  **Coupled Primitives:** Attributes are predicted from nearby anchors, storing only residuals or indices [S1].

**Steps:**
1.  **Hybrid Structure Construction:** Partition Gaussians into anchors and coupled primitives. The specific selection criteria (e.g., random, based on importance) are not detailed in the provided sources [S1].
2.  **Attribute Prediction:** For each coupled primitive, predict its attributes (position, rotation, scale, opacity, SH coefficients) based on the attributes of associated anchor primitives [S1].
3.  **Residual Encoding:** Compute the difference (residual) between the actual attributes of the coupled primitive and the predicted attributes. Encode these residuals compactly [S1].
4.  **Rate-Constrained Optimization:** Optimize the representation by jointly minimizing rendering distortion and bit rate [S1].
5.  **Entropy Estimation:** Model the bit rate for both anchor and coupled primitives using entropy estimation to guide the quantization and coding process [S1].

**Formulas:**
The optimization objective is a rate-distortion cost function:
$$ \mathcal{L} = \mathcal{D} + \lambda \mathcal{R} $$
Where:
*   $\mathcal{D}$ is the rendering distortion [S1].
*   $\mathcal{R}$ is the total bit rate, modeled via entropy estimation for anchors and coupled primitives [S1].
*   $\lambda$ is the Lagrange multiplier controlling the trade-off [S1].

**Parameters:**
*   $\lambda$: Not given in the sources.
*   Number of anchors/coupled split ratio: Not given in the sources.

**Edge Cases and Pitfalls:**
*   Prediction accuracy for coupled primitives depends on the spatial coherence of the scene. Degenerate cases where anchors are too far from coupled primitives may lead to large residuals [S1].
*   Entropy estimation accuracy is critical for rate modeling; poor estimation may lead to suboptimal quantization [S1].

## Subtopics

### [PCGS (Progressive Compression of 3D Gaussian Splatting)](20013-pcgs-progressive-compression-of-3d-gaussian-splatting.md)
A progressive compression framework that jointly and adaptively controls the quantity and quality of Gaussians to enable incremental bitstreams.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Reduced compared to standard 3DGS due to compact residual representations and entropy coding [S1].
*   **Performance:** Achieves high compression ratios while maintaining visual fidelity [S1].
*   **Strengths:** Effective hybrid structure reduces redundancy; rate-distortion optimization ensures efficient bit usage [S1].
*   **Limitations:** Complexity of predicting coupled attributes and entropy modeling [S1].
*   **When to use:** When storage or transmission bandwidth is constrained [S1].
*   **When not to use:** When real-time decoding complexity is a primary concern and compression is not critical [S1].

## Background
CompGS was proposed in the paper "CompGS: Efficient 3D Scene Representation via Compressed Gaussian Splatting" [S1]. It addresses the high memory footprint of 3D Gaussian Splatting by introducing a hybrid primitive structure and rate-constrained optimization [S1].

## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
