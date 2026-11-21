# Editable Gaussian Representation

Also called: Editable Gaussian, iVR-GS Gaussian

## Overview
Editable Gaussian Representation extends the iVR-GS framework [7018-ivr-gs-relighting-via-gaussian-primitives.md] by parameterizing each Gaussian primitive with explicit shading attributes, enabling direct manipulation of color, opacity, and lighting effects [S1]. Unlike standard 3D Gaussian Splatting which treats color as a static view-dependent attribute, this method decouples geometry from shading, allowing for scene recoloring and light editing by modifying specific primitive parameters rather than retraining the entire model [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of a scene; initial Gaussian parameters (geometry and shading); ground-truth color images for regularization [S1].
*   **Output:** A collection of $N$ editable Gaussian primitives, each defined by geometry and shading attributes, optionally compressed via Vector Quantization (VQ) and composed into a single model [S1].

**Data Structures:**
Each editable Gaussian $i$ is parameterized with two sets of attributes [S1]:
1.  **Geometry Attributes:** mean position $\mu_i$, rotation quaternion $q_i$, scale $s_i$, opacity $o_i$, and normal $n_i$ [S1].
2.  **Shading Attributes:** offset color $\Delta c_i$, ambient coefficient $k_{ia}$, diffuse coefficient $k_{id}$, specular coefficient $k_{is}$, and shininess $\beta_i$ [S1].

A global **Palette Color** $c_p \in \mathbb{R}^3$ is maintained per basic scene [S1].

**Rendering Formula:**
The voxel color $c_v$ is expressed as $c_v = c_p + \Delta c_i$ [S1]. The final rendered color is accumulated using standard alpha compositing, where the contribution of Gaussian $i$ depends on its opacity $\alpha_i$ (derived from $o_i$) and depth-based transmittance $T_i$ [S1].

**Training and Optimization Steps:**
1.  **Initialization:** Initialize Gaussians with geometry attributes. Set the palette color $c_p$ to the mean color value of the input multi-view images [S1].
2.  **Regularization:** During optimization, apply the following loss terms alongside the standard 3DGS photometric loss [S1]:
    *   **Opacity L1 Regularization:** Encourages opacity to be low to neutralize the tendency to use more Gaussians for compact surfaces, facilitating removal [S1]. (Specific formula not provided in the sources.)
    *   **Offset Color Sparsity:** Prevents significant palette color shifts. Defined as the L1 loss of the rendered offset color map $\Delta C = \sum_{i \in N} T_i \alpha_i \Delta c_i$ [S1].
    *   **Bilateral Smoothness:** Ensures shading attributes do not change significantly in smooth-color regions. For the ambient coefficient map $K_a = \sum_{i \in N} T_i \alpha_i k_{ia}$ and ground-truth color $c_{GT}$, the loss is $L_{\text{smooth}} = \| \nabla K_a \| \exp(-\| \nabla c_{GT} \|)$ [S1]. This regularization is similarly applied to $k_d, k_s,$ and $\beta$ [S1].
3.  **Pruning:** Periodically remove low-opacity Gaussians to reduce model size, similar to base 3DGS [S1].
4.  **Compression & Composition:** To recover all visible parts, multiple basic iVR-GS models are composed into a single model [S1]. Before composition, VQ using k-means clustering is applied to extract a more compact form of editable Gaussians for each basic model [S1]. The exact compression ratio and compression time are not given in the sources.

**Parameters:**
*   **Palette Color ($c_p$):** Fixed during training as the mean color of input images; adjustable during inference for scene recoloring [S1].
*   **Offset Color ($\Delta c$):** Added to enhance the expressiveness of $c_p$ [S1].

**Edge Cases and Pitfalls:**
*   **Compact Surfaces:** The model tends to utilize more Gaussians to reconstruct compact surfaces because color depends on the normal attribute, making opacity L1 regularization critical to neutralize this effect [S1].
*   **Palette Shifts:** Without sparsity regularization on $\Delta c$, the offset color may cause significant deviations from the intended palette [S1].

## Subtopics

### [Text-Driven Gaussian Editing](11107-text-driven-gaussian-editing.md)
Text-Driven Gaussian Editing modifies the appearance of VolVis scenes represented by textured Gaussian primitives using natural language text prompts while preserving underlying geometry and lighting. It distinguishes itself from iterative baseline methods by employing a one-time update strategy that applies image-to-image (IP2P) editing only once on a small subset of multi-view images, significantly reducing editing time and avoiding blurriness in semi-transparent scenes. Choose this variant when text-based appearance editing is required without altering scene geometry or requiring iterative optimization.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to base 3DGS with additional regularization overhead [S1].
*   **Memory Complexity:** The base model size is larger than standard 3DGS due to additional shading attributes per Gaussian, but VQ reduces the final model size [S1].
*   **Performance:** Enables real-time scene editing (recoloring, opacity adjustment, light direction) without retraining [S1].
*   **Strengths:** Explicit control over lighting and color; composability of scenes; reduced model size via pruning and VQ [S1].
*   **Limitations:** Requires multi-view input for training; complexity in managing shared codebooks for composition [S1].

## Background
The Editable Gaussian Representation was introduced in the iVR-GS paper, which proposes Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting [S1]. It builds upon the iVR-GS method which models lighting effects using the Blinn-Phong model [7018-ivr-gs-relighting-via-gaussian-primitives.md]. The method was developed to address the lack of explicit editability in standard 3DGS for volume visualization scenes [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
