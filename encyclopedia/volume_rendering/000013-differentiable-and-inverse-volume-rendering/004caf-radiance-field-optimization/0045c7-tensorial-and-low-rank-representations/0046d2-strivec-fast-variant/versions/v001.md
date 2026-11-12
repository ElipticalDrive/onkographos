# Strivec Fast Variant

Also called: Multi(0.6, 0.3) with TopK=2

## Overview
The Strivec Fast Variant is a computationally optimized configuration of the Strivec method, designed to accelerate training and inference while maintaining competitive rendering quality [S1]. It achieves this by reducing the number of multi-scale levels and the number of top-k candidates considered during ray sampling [S1]. It serves as a faster alternative to the standard Strivec configuration, offering a speed profile closer to Vector-Matrix (VM) decomposition while outperforming Canonical Polyadic (CP) decomposition in efficiency [S1].

## How it works
The method builds on the shared principles of Tensorial and Low-Rank Representations [11617-tensorial-and-low-rank-representations.md], specifically using sparse tri-vector radiance fields. The "Fast Variant" is defined strictly by its hyperparameters rather than a change in the underlying mathematical formulation of the radiance field [S1].

**Inputs and Outputs:**
*   **Input:** Camera poses, image observations, and the specified hyperparameters (scales, TopK).
*   **Output:** Rendered images and optimized tensor factors.

**Data Structures:**
The radiance field is represented using a multi-scale tri-vector structure. The specific configuration for the Fast Variant is:
*   **Number of Scales:** 2 [S1].
*   **Scale Lengths:** The tensor lengths for the two scales are 0.6 and 0.3 [S1].
*   **TopK:** The number of top-k candidates selected per sample point along a ray is 2 [S1].

**Parameters:**
*   `num_scales`: 2 [S1].
*   `scale_lengths`: [0.6, 0.3] [S1].
*   `top_k`: 2 [S1].
*   *Note:* The standard Strivec model uses 3 scales (0.15, 0.3, 0.6) and TopK=4 [S1]. The Fast Variant reduces these to accelerate computation [S1].

**Steps:**
1.  **Initialization:** Initialize the tri-vector tensor factors for the two scales with lengths 0.6 and 0.3 [S1].
2.  **Ray Sampling:** For each ray, sample points along the path [S1].
3.  **Sparse Selection:** For each sample point, identify the relevant tensor components. Instead of evaluating all components or using a higher TopK, select only the top 2 candidates (TopK=2) based on their relevance or magnitude [S1].
4.  **Interpolation and Rendering:** Interpolate the values from the selected sparse tri-vector factors to compute density and color, then perform volume rendering [S1].
5.  **Optimization:** Update the tensor factors using gradient descent, applying regularization as per the parent method's principles [11617-tensorial-and-low-rank-representations.md].

**Edge Cases and Pitfalls:**
*   Reducing TopK from 4 to 2 reduces the computational load significantly but may slightly impact the ability to capture high-frequency details if the scene is complex [S1].
*   Reducing scales from 3 to 2 simplifies the multi-resolution hierarchy, which may affect the smoothness of the rendered output compared to the 3-scale variant [S1].

## Subtopics

### Multi-scale Tri-Vector Radiance Fields
This is the standard multi-scale configuration of Strivec, serving as the baseline from which the Fast Variant is derived. It uses 3 scales with tensor lengths of 0.15, 0.3, and 0.6 [S1]. It typically employs 24 appearance components by default [S1]. This configuration provides more smoothness and better rendering quality than single-scale counterparts, consistent with findings in iNGP [S1]. A multi-scale model with 24 components outperforms a single-scale model with more parameters [S1]. The benefit of adding more components becomes marginal when the number reaches 48 [S1].

## Operating characteristics
**Time and Memory Complexity:**
The Fast Variant reduces computation by lowering the number of tensor ranks and the number of TopK candidates per sample point [S1]. This results in faster training and inference times compared to the standard Strivec configuration and TensoRF-CP [S1].

**Measured Performance:**
Evaluated on the NeRF Synthetic dataset [S1]:
*   **Training Time:** 959 seconds [S1].
*   **Inference Time:** 1.67 seconds per iteration [S1].
*   **Parameters:** 6.20 Million [S1].
*   **PSNR:** 33.09 [S1].

**Comparison:**
*   **vs. TensoRF-CP:** Faster training (959s vs 1914s), faster inference (1.67s/it vs 2.01s/it), more parameters (6.20M vs 0.98M), but higher PSNR (33.09 vs 31.56) [S1].
*   **vs. TensoRF-VM:** Similar training time (959s vs 915s), slightly slower inference (1.67s/it vs 1.60s/it), significantly fewer parameters (6.20M vs 17.95M), and comparable PSNR (33.09 vs 33.14) [S1].

**Strengths and Limitations:**
*   **Strengths:** Achieves a balance between speed and quality. It becomes faster than CP and close to VM in speed, while maintaining competitive quality [S1]. It improves quality, training time, and compactness together with proper hyper-parameters [S1].
*   **Limitations:** The reduction in scales and TopK may limit the model's capacity to learn very high-frequency details compared to the full 3-scale, TopK=4 configuration [S1].

## Background
The Strivec method (Sparse Tri-Vector Radiance Fields) was introduced in 2023 [S1]. The Fast Variant is an ablation study result from the same work, demonstrating that reducing the number of scales and TopK candidates accelerates the method without significant quality loss [S1]. The method is part of the broader family of tensorial and low-rank representations for neural radiance fields [11617-tensorial-and-low-rank-representations.md].

## Sources
[S1] Strivec: Sparse Tri-Vector Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01611, fetched 2026-10-26
