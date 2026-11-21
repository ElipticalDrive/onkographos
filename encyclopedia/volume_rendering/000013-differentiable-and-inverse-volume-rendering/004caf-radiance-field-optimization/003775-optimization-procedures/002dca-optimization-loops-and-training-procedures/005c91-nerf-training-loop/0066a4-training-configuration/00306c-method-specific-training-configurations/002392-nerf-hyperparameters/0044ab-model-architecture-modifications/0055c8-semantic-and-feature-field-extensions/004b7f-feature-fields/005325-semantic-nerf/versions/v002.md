# Semantic-NeRF


## Overview
Semantic-NeRF is a method for synthesizing semantic labels for novel views by extending the standard NeRF formulation with an additional direction-independent MLP branch [S1]. It solves the problem of 3D semantic understanding by integrating RGB images and corresponding 2D semantic labels to train the network, allowing for the recovery of semantic information from sparse or noisy annotations [S2]. It is distinguished from its siblings by being the first work to integrate semantic information directly into the NeRF framework using a dedicated semantic output branch [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB images and corresponding 2D semantic label maps for known views.
*   **Outputs:** Rendered RGB images and rendered semantic label maps for novel views.

**Data Structures:**
The model extends the standard NeRF architecture. The position MLP $\Phi_v$ and direction MLP $\Phi_d$ remain unchanged. An additional semantic MLP, denoted as $\Phi_s$, is introduced.
*   $\Phi_s$: A multi-layer perceptron that takes the output features from the position MLP (and potentially density) as input and outputs a semantic vector $s$. The source specifies this branch is "direction-independent" [S1].

**Rendering Formula:**
Semantic labels are generated via volume rendering, following the same integration principle as color [S1, S2]. For a ray $r$, the rendered semantic label $\hat{S}(r)$ is computed as:

$$
\hat{S}(r) = \sum_{i=1}^{N} T_i \alpha_i s_i
$$

where:
*   $N$ is the number of sample points along the ray.
*   $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance.
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity weight.
*   $s_i$ is the semantic label predicted by the semantic MLP $\Phi_s$ at point $x_i$.

**Training Steps:**
1.  **Input Preparation:** Collect RGB images and their corresponding semantic label maps for training views [S2].
2.  **Network Forward Pass:**
    *   Sample points along rays.
    *   Compute position features and density using the standard NeRF MLPs.
    *   Pass position features (and density, if applicable) through the semantic MLP $\Phi_s$ to obtain point-wise semantic labels $s_i$ [S1].
    *   Render the final semantic map $\hat{S}(r)$ using the volume rendering equation above [S1].
3.  **Loss Calculation:** Supervise the rendered semantic labels using categorical cross-entropy loss [S1]. The RGB loss is typically handled by the standard NeRF color branch (implied by the use of RGB images) [S2].
4.  **Optimization:** Update network parameters to minimize the combined loss [S2].

**Parameters:**
*   **Label Sparsity:** The method is capable of training with sparse semantic label data, specifically citing 10% labeled data [S1].
*   **Noise Robustness:** The method handles pixel-wise noise and region/instance-wise noise in labels [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Data:** The method is designed to handle cases where only a small fraction of views have semantic labels (e.g., 10%) [S1].
*   **Label Noise:** It is robust to noise in the input semantic labels, including pixel-wise and region-wise noise [S1].

## Subtopics

### Multi-Space Neural Radiance Field (MS-NeRF)
Extends NeRF to multiple spaces (e.g., 2D, 3D, 4D) for dynamic scenes. [Link](11707-multi-space-neural-radiance-field-ms-nerf.md)

### NRFF
Uses neural radiance feature fields for specific tasks. [Link](12072-nrff.md)

### Semantic and Feature Branches
Generalizes the concept of separate branches for different outputs. [Link](12095-semantic-and-feature-branches.md)

### NeRF for Object Segmentation
Focuses on segmenting specific objects within the scene. [Link](12404-nerf-for-object-segmentation.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method achieves good label super-resolution results and label propagation from sparse point-wise annotations [S1]. It outperforms non-deep learning methods in multi-view semantic fusion [S1].
*   **Strengths:** Ability to train with sparse labels (10%), robustness to label noise, and capability for label super-resolution [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When 3D semantic understanding is required and only sparse or noisy 2D semantic labels are available [S1, S2].
*   **When not to use:** When dense, high-quality semantic labels are available and computational efficiency is critical (though not explicitly stated, this is a general NeRF limitation).

## Background
Semantic-NeRF was introduced in March 2021 [S1]. It is the first work to integrate semantic information into the NeRF framework [S2]. It builds upon the earlier Fig-NeRF [S1]. The method was developed to address the need for 3D semantic understanding in computer vision, enabling tasks such as novel view synthesis of semantic labels and multi-view semantic fusion [S1]. It has influenced subsequent Gaussian-Splatting-based semantic view synthesis models [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
