# Conditional 3D Gaussian Splatting Generation

Also called: Conditional Generation (Text/Image/Partial), DiffGS Conditional

## Overview
Conditional 3D Gaussian Splatting Generation learns a feedforward or diffusion model to output 3D Gaussian Splatting (3DGS) representations directly from text prompts, input images, or partial 3DGS inputs, bypassing the iterative optimization required by standard reconstruction methods [S1]. It differs from sibling methods like Align Your Gaussians (AYG) by focusing on generative completion and synthesis from sparse or semantic conditions rather than aligning existing multi-view images [S1, S14896-align-your-gaussians-ayg.md].

## How it works
The method generates 3D Gaussian fields conditioned on external inputs. The core implementation, DiffGS, uses a diffusion framework to denoise Gaussian attributes [S1].

**Inputs and Outputs:**
*   **Inputs:** Text prompts, single/multi-view images, or partial 3DGS point clouds/attributes.
*   **Outputs:** A set of 3D Gaussians defined by position $\mathbf{p}$, covariance $\mathbf{\Sigma}$ (or scale/rotation), opacity $\alpha$, and spherical harmonics coefficients for color $\mathbf{c}$.

**Data Structures:**
*   **Gaussian Field:** A list of $N$ Gaussians. Each Gaussian $i$ has:
    *   Position: $\mathbf{p}_i \in \mathbb{R}^3$
    *   Scale: $\mathbf{s}_i \in \mathbb{R}^3$ (log-space)
    *   Rotation: $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion)
    *   Opacity: $\alpha_i \in [0, 1]$
    *   Color: $\mathbf{c}_i \in \mathbb{R}^k$ (SH coefficients)
*   **Condition Encoder:**
    *   Text/Image: Frozen CLIP encoder [S1].
    *   Partial 3DGS: Modified PointNet [S1].

**Steps:**
1.  **Condition Encoding:**
    *   For text: Encode prompt $T$ using frozen CLIP text encoder $\gamma_{text}$ to get embedding $\mathbf{e}_{text} = \gamma_{text}(T)$ [S1].
    *   For image: Encode input image $I$ using frozen CLIP image encoder $\gamma_{image}$ to get embedding $\mathbf{e}_{image} = \gamma_{image}(I)$ [S1].
    *   For partial 3DGS: Encode partial Gaussian attributes $\mathcal{G}_{partial}$ using modified PointNet $\gamma_{partial}$ to get embedding $\mathbf{e}_{partial} = \gamma_{partial}(\mathcal{G}_{partial})$ [S1].
2.  **Diffusion Process:**
    *   Initialize noisy Gaussian attributes $\mathcal{G}_T$.
    *   Train a neural network $\epsilon_\theta$ (or similar diffusion backbone) to predict noise or clean attributes conditioned on $\mathbf{e}_{cond}$ (where $\mathbf{e}_{cond}$ is $\mathbf{e}_{text}$, $\mathbf{e}_{image}$, or $\mathbf{e}_{partial}$) [S1].
    *   Optimize the model using the conditional diffusion objective (Eq. 8 in [S1]), which typically involves minimizing the difference between predicted and actual noise in the latent space of Gaussian attributes, conditioned on the encoded prompt [S1].
3.  **Sampling:**
    *   Iteratively denoise $\mathcal{G}_t$ to $\mathcal{G}_{t-1}$ using the conditioned model until $\mathcal{G}_0$ is obtained [S1].

**Parameters:**
*   **Dataset:** ShapeNet chair class with standard splits [S1].
*   **CLIP Model:** Pretrained CLIP [S1].
*   **PointNet:** Modified architecture for partial 3DGS [S1].

**Edge Cases and Pitfalls:**
*   **Partial Input Occlusion:** The method must handle highly occluded inputs; the PointNet encoder must robustly extract features from sparse partial Gaussians [S1].
*   **Semantic Alignment:** Text/image conditions must align with the generated geometry; poor alignment results in blurry textures or incorrect shapes [S1].

## Subtopics

### [Promptable Explicit Scene Representation](15906-promptable-explicit-scene-representation.md)
A mechanism for 3D editing (extraction, deletion, appearance modification) in 3D Gaussian Splatting by computing activation scores of 3D Gaussians based on prompts (text, points, boxes) using cosine similarity and softmax. It differs from implicit NeRF feature fields by operating on explicit Gaussian primitives, allowing direct manipulation of color and opacity parameters for specific semantic subsets.

### [Appearance-Aware 3D Gaussian Splatting (AAGS)](18279-appearance-aware-3d-gaussian-splatting-aags.md)
Appearance-Aware 3D Gaussian Splatting (AAGS) extends 3D Gaussian Splatting to handle unconstrained photo collections by explicitly modeling per-image appearance variations and removing transient objects (e.g., pedestrians, vehicles) from the static scene reconstruction. It sets itself apart from standard 3DGS by integrating a transient object visibility mask and an appearance interpolation mechanism, and from NeRF-based methods like Ha-NeRF by maintaining the efficiency of splatting while achieving competitive or superior perceptual metrics (LPIPS) without requiring test-set images for appearance embedding training.

#### Interactive3D
A method operating in two stages for interactive 3D object generation and refinement, optimizing 3D objects through 3DGS and SDS in first stage, then transforming model into InstantNGP structure with interactive hash refinement module in second stage [S2].

#### 3D Gaussian Splatting (Text-to-3D Implementation)
A reimplementation of 3D Gaussian Splatting adapted for text-to-3D generation, modified to support a learnable MLP background and specific gradient propagation [S3]. Gradient back-propagation calculates gradients in the forward rendering order, rather than the inverse order used in the official code [S3]. Depth maps are rendered by sampling the view-space depth at the centers of the Gaussians [S3]. A z-variance renderer is implemented to compute z-variance for use in a z-var loss term during optimization [S3]. Gaussian opacity is clamped to a fixed range during rendering and optimization to stabilize gradients and prevent numerical overflow/underflow [S3]. The target number of Gaussians after densification during training is around $10^5$ to $10^6$ [S3]. The official implementation propagates gradients in inverse order (last rendered Gaussians receive gradients first), whereas this variant uses rendering order to simplify per-pixel background integration [S3].

#### Gaussian Pulling to Zero-Level Set
A technique that pulls Gaussian centers onto the zero-level set of a neural SDF field to prevent floating ellipsoids and achieve accurate geometry estimation. It also pulls query points to Gaussian disks rather than centers to bridge the gap between continuous SDF fields and sparse Gaussian distributions [S4]. Steps: 1. Pull Gaussian centers to the zero-level set of the SDF field. 2. Pull neighboring space/query points onto Gaussian disks. 3. Use normalized SDF gradient for the pulling operation [S4]. Differs from NeuralPull [45], which pulls query points to Gaussian centers. This method pulls query points to Gaussian disks, better bridging continuous SDF fields and sparse Gaussian distributions [S4].

#### Multi-view Consistent Pruning
A pruning strategy that removes redundant Gaussians by evaluating their contribution to multi-view reconstruction quality, using photometric loss combined with the densification score [S5, unofficial]. Select Gaussian $G_i$ for pruning if its pruning score $s^i_p$ exceeds a predefined threshold $\tau_p$ [S5, unofficial]. Compute photometric loss $E^j_{photo}$ for each view $j$: $E^j_{photo} = (1 - \lambda)L^j_1 + \lambda(1 - L^j_{SSIM})$, where $L^j_1$ is mean absolute error and $L^j_{SSIM}$ is structural similarity loss [S5, unofficial]. $\lambda$: Weight for the SSIM term in the photometric loss. Default value is 0.2 [S5, unofficial]. $\tau_p$: Threshold for the pruning score $s^i_p$ [S5, unofficial].

#### LGM
Exploits off-the-shelf models to generate multi-view images from text or a single image [S6, unofficial]. Trains a U-Net-based network to generate 3DGS from multi-view images [S6, unofficial]. The U-Net is asymmetric, which allows for the input of high-resolution images while limiting the number of output Gaussians [S6, unofficial].

#### GSEdit
A method that takes a texture mesh or pre-trained 3DGS as input [S6, unofficial]. Utilizes Instruct-Pix2Pix and SDS loss to update the input mesh or 3DGS [S6, unofficial].

#### GaussianObject
A method that initializes 3D Gaussians with a visual hull and fine-tunes a pre-trained ControlNet to repair degraded rendered images generated by adding noise to each Gaussian's attributes [S6, unofficial]. Outperforms previous NeRF-based sparse-view reconstruction methods [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time depends on the diffusion steps and the complexity of the condition encoder. DiffGS requires training on datasets like ShapeNet [S1]. Text-to-3D implementations may take ~40 minutes per prompt for optimization [S3].
*   **Memory Complexity:** Depends on the number of Gaussians ($10^5$ to $10^6$) and the size of the condition embeddings [S3].
*   **Performance:** DiffGS produces significantly more visual-appealing and high-fidelity generations in terms of rendering and geometry qualities compared to DiffTF [S1].
*   **Strengths:** Enables controllable generation from text, images, and partial data; handles occlusions well [S1].
*   **Limitations:** Requires careful tuning of diffusion parameters; may suffer from blurry textures if not properly conditioned [S1].
*   **When to use:** When generating 3D assets from text descriptions, single images, or incomplete scans is required.
*   **When not to use:** When high-fidelity reconstruction from dense multi-view images is available and no generative prior is needed.

## Background
Originated with DiffGS, which introduced functional Gaussian Splatting Diffusion for conditional generation [S1]. The method leverages frozen CLIP encoders for text and image conditioning and modified PointNet for partial 3DGS conditioning [S1]. It addresses the limitation of previous diffusion-based 3D generation methods (like DiffTF) which produced blurry results [S1]. The approach has been extended to various variants including text-to-3D reimplimentations [S3], interactive generation [S2], and geometry-aware pulling [S4].

## Sources
[S1] DiffGS: Functional Gaussian Splatting Diffusion, https://arxiv.org/abs/2410.19657v2, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S3] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S4] Neural Signed Distance Function Inference through Splatting 3D Gaussians Pulled on Zero-Level Set, https://doi.org/10.48550/arXiv.2410.14189, fetched 2026-10-29
[S5] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
[S6] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
