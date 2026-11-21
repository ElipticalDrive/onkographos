# Editable Gaussian Representation

Also called: Editable Gaussian, iVR-GS Gaussian

## Overview
Editable Gaussian Representation extends the iVR-GS framework [7018-ivr-gs-relighting-via-gaussian-primitives.md] by parameterizing each Gaussian primitive with explicit shading attributes, enabling direct manipulation of color, opacity, and lighting effects [S1]. Unlike standard 3D Gaussian Splatting which treats color as a static view-dependent attribute, this method decouples geometry from shading, allowing for scene recoloring, opacity adjustment, and directional light editing by modifying specific primitive parameters rather than retraining the entire model [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of a scene; initial Gaussian parameters (geometry and shading); ground-truth color images for regularization.
*   **Output:** A collection of $N$ editable Gaussian primitives, each defined by geometry and shading attributes, optionally compressed via Vector Quantization (VQ) and composed into a single model.

**Data Structures:**
Each editable Gaussian $i$ is defined by two sets of attributes:
1.  **Geometry Attributes:**
    *   Mean position $\mu_i \in \mathbb{R}^3$
    *   Rotation quaternion $q_i \in \mathbb{R}^4$
    *   Scale $s_i \in \mathbb{R}^3$
    *   Opacity $o_i \in \mathbb{R}$
    *   Normal $n_i \in \mathbb{R}^3$
2.  **Shading Attributes:**
    *   Offset color $\Delta c_i \in \mathbb{R}^3$
    *   Ambient coefficient $k_{ia} \in \mathbb{R}$
    *   Diffuse coefficient $k_{id} \in \mathbb{R}$
    *   Specular coefficient $k_{is} \in \mathbb{R}$
    *   Shininess $\beta_i \in \mathbb{R}$

Additionally, a global **Palette Color** $c_p \in \mathbb{R}^3$ is maintained per basic scene.

**Rendering Formula:**
The color $c_v$ of a voxel or pixel covered by Gaussian $i$ is computed as:
$$ c_v = c_p + \Delta c_i $$
The final rendered color is accumulated using standard alpha compositing, where the contribution of Gaussian $i$ depends on its opacity $\alpha_i$ (derived from $o_i$) and depth-based transmittance $T_i$.

**Training and Optimization Steps:**
1.  **Initialization:** Initialize Gaussians with geometry attributes. Set the palette color $c_p$ to the mean color value of the input multi-view images [S1].
2.  **Regularization Application:** During optimization, apply the following loss terms in addition to the standard 3DGS photometric loss:
    *   **Opacity L1 Regularization:** Encourages low opacity to prevent over-reconstruction of compact surfaces and facilitate the removal of redundant Gaussians. The specific formula is not provided, but it is an L1 term on opacity [S1].
    *   **Offset Color Sparsity:** Prevents significant shifts from the palette color. Defined as the L1 loss of the rendered offset color map $\Delta C$:
        $$ L_{\text{sparsity}} = \| \Delta C \|_1, \quad \text{where } \Delta C = \sum_{i \in N} T_i \alpha_i \Delta c_i $$
    *   **Bilateral Smoothness:** Applied to shading attributes ($k_a, k_d, k_s, \beta$) to ensure smoothness in regions with smooth ground-truth colors. For the ambient coefficient map $K_a = \sum_{i \in N} T_i \alpha_i k_{ia}$, the loss is:
        $$ L_{\text{smooth}} = \| \nabla K_a \| \exp(-\| \nabla c_{GT} \|) $$
        where $c_{GT}$ is the ground-truth color [S1]. This is similarly applied to $k_d, k_s, \beta$.
3.  **Pruning:** Periodically remove Gaussians with low opacity to reduce model size, similar to base 3DGS [S1].
4.  **Compression (Optional):** Apply Vector Quantization (VQ) using k-means clustering to compress the attributes of each basic iVR-GS model, achieving roughly 4× compression without sacrificing quality [S1].
5.  **Composition:** To form a composed model, save exact values of $\mu$ and $n$, and indices with shared codebooks for other attributes. Decompress quantized models and append attribute values from basic models into a single parameter list [S1].

**Parameters:**
*   **Palette Color ($c_p$):** Fixed during training as the mean color of input images; adjustable during inference for recoloring [S1].
*   **Offset Color ($\Delta c$):** Enhances expressiveness of $c_p$ [S1].

**Edge Cases and Pitfalls:**
*   **Compact Surfaces:** The model tends to use more Gaussians to reconstruct compact surfaces because color depends on the normal attribute. The L1 opacity regularization is critical to neutralize this effect [S1].
*   **Palette Shifts:** Without sparsity regularization on $\Delta c$, the offset color may cause significant deviations from the intended palette [S1].

## Subtopics

### [Text-Driven Gaussian Editing](11107-text-driven-gaussian-editing.md)
Text-Driven Gaussian Editing performs non-photorealistic scene editing (NPSE) on volume visualization (VolVis) scenes by applying text instructions via an image-to-image (IP2P) diffusion model. It distinguishes itself from sibling methods like DGE, IG2G, and IN2N by employing a one-time update strategy on a small subset of multi-view images and by keeping Gaussian geometry fixed while updating only texture attributes, thereby ensuring geometry consistency and significantly reducing editing time.

#### Scene Composition
Scene Composition allows the merging of multiple basic iVR-GS models into a single composed model. The process involves saving the exact values of geometry attributes $\mu$ and $n$, and indices with shared codebooks for other attributes for each editable Gaussian. The palette color $c_p$ is stored as a global parameter for each basic scene. The composed model is formed by decompressing each model from its quantized format and appending the attribute values of primitives from all basic iVR-GS models into a single parameter list. This enables real-time editing of color, opacity, and light direction by adjusting the attributes of each basic model within the composition [S1].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to base 3DGS with additional regularization overhead. The VQ compression time is negligible [S1].
*   **Memory Complexity:** The base model size is larger than standard 3DGS due to additional shading attributes per Gaussian. However, VQ provides roughly 4× compression [S1].
*   **Performance:** Enables real-time scene editing (recoloring, opacity adjustment, light direction) without retraining [S1].
*   **Strengths:** Explicit control over lighting and color; composability of scenes; reduced model size via pruning and VQ.
*   **Limitations:** Requires multi-view input for training; complexity in managing shared codebooks for composition.

## Background
The Editable Gaussian Representation was introduced in the iVR-GS paper, which proposes Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting. It builds upon the iVR-GS method which models lighting effects using the Blinn-Phong model. The method was developed to address the lack of explicit editability in standard 3DGS for volume visualization scenes [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
