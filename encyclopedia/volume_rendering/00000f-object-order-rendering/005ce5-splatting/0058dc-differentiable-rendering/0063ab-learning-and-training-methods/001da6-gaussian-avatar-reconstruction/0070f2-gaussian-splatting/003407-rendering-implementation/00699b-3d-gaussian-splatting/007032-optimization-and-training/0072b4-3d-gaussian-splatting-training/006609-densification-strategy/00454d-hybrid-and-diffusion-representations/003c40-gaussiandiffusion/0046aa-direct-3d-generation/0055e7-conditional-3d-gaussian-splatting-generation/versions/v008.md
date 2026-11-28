# Conditional 3D Gaussian Splatting Generation

Also called: Conditional Generation (Text/Image/Partial), DiffGS Conditional

## Overview
Conditional 3D Gaussian Splatting Generation produces 3D Gaussian Splatting (3DGS) representations directly from text prompts, input images, or partial 3DGS inputs containing large occlusions [S1]. It solves the problem of controllable 3D asset creation and scene completion without relying on iterative per-scene optimization or explicit intermediate image sets, amortizing reconstruction cost by learning a feedforward or diffusion-based mapping [S1, Parent]. It sets itself apart from sibling methods like [Align Your Gaussians (AYG)](14896-align-your-gaussians-ayg.md) by focusing on static or diffusion-based generation conditioned on semantic or geometric priors rather than dynamic 4D scene synthesis [S1, S7].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Text prompts, single/multi-view images, or partial 3DGS point clouds/splats [S1].
*   **Outputs:** A set of 3D Gaussians defined by position $\mathbf{x}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and spherical harmonics coefficients for color [S1, Parent].

**Data Structures:**
The method utilizes custom encoders to map conditions into a latent space compatible with the generation model (e.g., a diffusion model or MLP).
*   **Text Encoder ($\gamma_{\text{text}}$):** Frozen CLIP text encoder [S1].
*   **Image Encoder ($\gamma_{\text{image}}$):** Frozen CLIP image encoder [S1].
*   **Partial Input Encoder ($\gamma_{\text{partial}}$):** Modified PointNet architecture for encoding partial 3DGS inputs [S1].

**Algorithm Steps:**
1.  **Condition Encoding:**
    *   If the condition is text, compute embedding $\mathbf{c}_{\text{text}} = \gamma_{\text{text}}(\text{text})$ [S1].
    *   If the condition is an image, compute embedding $\mathbf{c}_{\text{image}} = \gamma_{\text{image}}(\text{image})$ [S1].
    *   If the condition is partial 3DGS, encode the partial geometry/appearance using $\gamma_{\text{partial}}$ to produce $\mathbf{c}_{\text{partial}}$ [S1].

2.  **Generation (DiffGS Approach):**
    *   Train a diffusion model (DiffGS) with a conditional optimization objective [S1]. The specific formulation of Eq.(8) in [S1] is not provided in the source text, but it involves conditioning the diffusion process on $\mathbf{c}_{\text{text}}$, $\mathbf{c}_{\text{image}}$, or $\mathbf{c}_{\text{partial}}$ [S1].
    *   Sample from the diffusion model to generate Gaussian attributes [S1].

3.  **Alternative Implementation (Text-to-3D Reimplementation):**
    *   Use a learnable MLP background and specific gradient propagation rules [S3].
    *   Render depth maps by sampling view-space depth at Gaussian centers [S3].
    *   Implement a z-variance renderer to compute z-variance for a z-var loss term [S3].
    *   Clamp Gaussian opacity to $[0.004, 0.99]$ during rendering and optimization to stabilize gradients and prevent numerical overflow/underflow [S3].
    *   Propagate gradients in forward rendering order (unlike the official inverse order) to simplify per-pixel background integration [S3].

4.  **Geometry Refinement (Gaussian Pulling to Zero-Level Set):**
    *   To prevent floating ellipsoids and achieve accurate geometry, pull Gaussian centers onto the zero-level set of a neural SDF field [S4].
    *   Pull neighboring space/query points onto Gaussian disks (rather than just centers) to bridge the gap between continuous SDF fields and sparse Gaussian distributions [S4].
    *   Use the normalized SDF gradient for the pulling operation [S4].
    *   Do not add an Eikonal loss term, as it degenerates optimization by making the SDF field optimization too complex [S4].

5.  **Pruning (Multi-view Consistent Pruning):**
    *   Evaluate Gaussian contribution to multi-view reconstruction quality [S5, unofficial].
    *   Compute photometric loss $E^j_{\text{photo}}$ for each view $j$:
        $$ E^j_{\text{photo}} = (1 - \lambda)L^j_1 + \lambda(1 - L^j_{\text{SSIM}}) $$
        where $L^j_1$ is mean absolute error and $L^j_{\text{SSIM}}$ is structural similarity loss [S5, unofficial].
    *   Identify high-error pixels using a mask $M^j_{\text{mask}}$ derived from normalized reconstruction error [S5, unofficial].
    *   Calculate the pruning score $s^i_p$ for Gaussian $G_i$:
        $$ s^i_p = N \left( \frac{1}{K} \sum_{j=1}^K \sum_{p \in \Omega_i} I(M^j_{\text{mask}}(p) = 1) \cdot E^j_{\text{photo}} \right) $$
        where $K$ is the number of views, $\Omega_i$ is the 2D footprint of $G_i$, and $N(\cdot)$ is min-max normalization [S5, unofficial].
    *   Select $G_i$ for pruning if $s^i_p > \tau_p$ [S5, unofficial].

**Parameters:**
*   $\lambda$: Weight for the SSIM term in photometric loss. Default: $0.2$ [S5, unofficial].
*   $\tau_p$: Threshold for the pruning score $s^i_p$ [S5, unofficial].
*   Opacity Clamp: $[0.004, 0.99]$ [S3].
*   Target Gaussians: $10^5$ to $10^6$ after densification [S3].
*   Dataset: ShapeNet chair class with standard splits [S1].

**Edge Cases and Pitfalls:**
*   **Gradient Order:** The official 3DGS implementation propagates gradients in inverse order (last rendered Gaussians first). Reimplementations may use forward order to simplify background integration, which is a design choice rather than a bug [S3].
*   **Z-var Loss:** While implemented in some variants, z-var loss has limited impact on generated assets due to the natural sparsity of Gaussians enforcing a thin surface [S3].
*   **Eikonal Loss:** Adding an Eikonal loss term to the Gaussian pulling operation significantly degrades results; it should be omitted [S4].
*   **4D Motion Distortion:** In dynamic generation variants (e.g., Align Your Gaussians), interpolation in the temporal dimension may induce motion distortion because it does not consider physical dynamics constraints [S7].

## Subtopics

### [Promptable Explicit Scene Representation](15906-promptable-explicit-scene-representation.md)
A mechanism for 3D editing (extraction, deletion, appearance modification) in 3D Gaussian Splatting by computing activation scores of 3D Gaussians based on prompts (text, points, boxes) using cosine similarity and softmax. It differs from implicit NeRF feature fields by operating on explicit Gaussian primitives, allowing direct manipulation of color and opacity parameters for specific semantic subsets.

### [GSEdit](17282-gsedit.md)
GSEdit is a method for editing pre-trained 3D Gaussian Splatting (3DGS) representations or texture meshes using text or image instructions. It solves the problem of applying 2D diffusion-based edits to 3D scenes while maintaining geometric consistency. It sets itself apart from sibling methods like [GaussianObject](17321-gaussianobject.md) or [Promptable Explicit Scene Representation](15906-promptable-explicit-scene-representation.md) by directly utilizing Instruct-Pix2Pix and Score Distillation Sampling (SDS) loss to update the input representation, rather than relying on complex multi-view consistency constraints or explicit texture mapping networks like [Texture-GS](18279-appearance-aware-3d-gaussian-splatting-aags.md).

### [GaussianObject](17321-gaussianobject.md)
GaussianObject is a method for 3D Gaussian Splatting (3DGS) reconstruction from sparse or partial inputs that initializes Gaussians using a visual hull and employs a fine-tuned ControlNet to repair degraded rendered images. It distinguishes itself from prior NeRF-based sparse-view methods by leveraging diffusion priors for attribute refinement, outperforming previous approaches in reconstruction quality.

### [Appearance-Aware 3D Gaussian Splatting (AAGS)](18279-appearance-aware-3d-gaussian-splatting-aags.md)
Appearance-Aware 3D Gaussian Splatting (AAGS) extends 3D Gaussian Splatting to handle unconstrained photo collections by explicitly modeling per-image appearance variations and removing transient objects (e.g., pedestrians, vehicles) from the static scene reconstruction. It sets itself apart from standard 3DGS by integrating a transient object visibility mask and an appearance interpolation mechanism, and from NeRF-based methods like Ha-NeRF by maintaining the efficiency of splatting while achieving competitive or superior perceptual metrics (LPIPS) without requiring test-set images for appearance embedding training.

### [VTC (Visibility-Aware Texture Consensus)](21244-vtc-visibility-aware-texture-consensus.md)
VTC is a pre-training observation filtering and consensus mechanism designed to improve 3D Gaussian Splatting reconstruction in texture-degraded scenes by rejecting unreliable observations and decoupling geometry from appearance. It differs from existing approaches that treat all visible observations as equally reliable by performing observation-level rejection of colors corrupted by occlusion, background mixing, or exposure differences, and by using a fixed robust consensus target computed before training rather than an online-updated uncertainty estimate.

#### Interactive3D
A method operating in two stages for interactive 3D object generation and refinement, optimizing 3D objects through 3DGS and SDS in first stage, then transforming model into InstantNGP structure with interactive hash refinement module in second stage [S2].

#### 3D Gaussian Splatting (Text-to-3D Implementation)
A reimplementation of 3D Gaussian Splatting adapted for text-to-3D generation, modified to support a learnable MLP background and specific gradient propagation [S3]. The official implementation propagates gradients in inverse order (last rendered Gaussians receive gradients first), whereas this variant uses rendering order to simplify per-pixel background integration [S3].

#### Gaussian Pulling to Zero-Level Set
A technique that pulls Gaussian centers onto the zero-level set of a neural SDF field to prevent floating ellipsoids and achieve accurate geometry estimation. It also pulls query points to Gaussian disks rather than centers to bridge the gap between continuous SDF fields and sparse Gaussian distributions [S4].

#### Multi-view Consistent Pruning
A pruning strategy that removes redundant Gaussians by evaluating their contribution to multi-view reconstruction quality, using photometric loss combined with the densification score [S5, unofficial].

#### LGM
Exploits off-the-shelf models to generate multi-view images from text or a single image, then trains a U-Net-based network to generate 3DGS from multi-view images [S6, unofficial]. The U-Net is asymmetric, which allows for the input of high-resolution images while limiting the number of output Gaussians [S6, unofficial].

#### Align Your Gaussians
A method integrating text-to-image/video diffusion models for generating 4D dynamic scenes [S7]. 4D generation induces motion distortion because interpolation in the temporal dimension does not consider physical dynamics constraints [S7].

## Operating characteristics
*   **Time Complexity:** Training time varies by variant. Text-to-3D implementations using diffusion guidance (e.g., Stable Diffusion) take approximately 40 minutes per prompt [S3]. DiffGS training time is not explicitly given but involves diffusion steps [S1].
*   **Memory Complexity:** The number of Gaussians after densification is typically between $10^5$ and $10^6$ [S3].
*   **Measured Performance:** DiffGS produces significantly more visual-appealing and high-fidelity generations in terms of rendering and geometry qualities compared to prior methods like DiffTF, which suffer from blurry generations and poor textures [S1].
*   **Strengths:** Capable of recovering complex geometries and detailed appearances from highly occluded 3DGS inputs [S1]. Accurately recovers semantics and geometries described in text prompts [S1].
*   **Limitations:** 4D generation variants may suffer from motion distortion due to lack of physical dynamics constraints [S7]. Z-var loss has limited impact on asset quality [S3].
*   **When to Use:** When conditional generation from text, images, or partial data is required without iterative per-scene optimization.
*   **When Not to Use:** When strict physical dynamics consistency in 4D generation is required [S7].

## Background
Conditional 3D Gaussian Splatting Generation originates from the need to control 3D generation using semantic or geometric priors. DiffGS was introduced to enable conditional generation using texts, images, and partial 3DGS inputs, demonstrating the ability to recover complex geometries from occluded inputs [S1]. The field has expanded to include interactive refinement (Interactive3D) [S2], efficient text-to-3D reimplementations (LGM, Text-to-3D GS) [S3][S6, unofficial], and geometry-aware techniques like Gaussian Pulling to Zero-Level Set [S4] and Multi-view Consistent Pruning [S5, unofficial]. Dynamic generation methods like Align Your Gaussians have also emerged, though they face challenges with temporal consistency [S7].

## Sources
[S1] DiffGS: Functional Gaussian Splatting Diffusion, https://arxiv.org/abs/2410.19657v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S3] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S4] Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, https://doi.org/10.48550/arXiv.2410.14189, fetched 2026-10-29
[S5] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
[S6] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S7] Implicit vs. explicit a comparative survey on NeRF and 3DGS for large-scale aerial scene editing, https://link.springer.com/article/10.1007/s10791-026-10029-9, fetched 2026-10-26
