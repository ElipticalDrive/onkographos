# SinNeRF

Also called: Sinusoidal NeRF

## Overview
SinNeRF is a method for reconstructing Neural Radiance Fields (NeRF) from a single input image, addressing the severe under-constrained nature of single-view scene reconstruction [S1]. It distinguishes itself from standard NeRF and other single-view methods (like DS-NeRF or PixelNeRF) by integrating image warping for depth supervision, adversarial training for texture, and global feature comparison for structural consistency [S1]. While standard NeRF optimizes for photometric consistency across multiple views, SinNeRF regularizes the optimization loop using priors derived from the single reference view to constrain unseen views [S2, unofficial].

## How it works
SinNeRF builds on the canonical NeRF architecture [S1, S3]. It maps 6D coordinates (3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$) to density $\sigma$ and color $\mathbf{c}$ using two coupled MLPs with positional encoding $\gamma(\cdot)$ [S1][S3]. The rendering follows the standard volumetric integration:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
where $\delta_i$ is the distance between samples [S1][S3].

**Inputs:**
- Single reference image $I_{ref}$ with known camera intrinsics and pose.
- Randomly sampled unseen view poses (used for regularization).

**Outputs:**
- A trained NeRF model capable of synthesizing novel views.

**Data Structures:**
- **MLP:** Same architecture as the original NeRF paper [S3].
    - Density Network $F_\sigma$: Input $\gamma(\mathbf{x})$, $L$ hidden layers with ReLU, skip connections from input to every 4th layer. Output $\sigma$.
    - Color Network $F_c$: Input $\gamma(\mathbf{x})$, raw $\mathbf{d}$, and intermediate feature vector from $F_\sigma$. Output $\mathbf{c}$ [S1][S3].
- **Discriminator:** CNN discriminator for patch-wise texture supervision [S1].
- **Feature Extractor:** Pretrained Vision Transformer (ViT, specifically DINO-ViT) for global structure prior [S1].

**Training Steps:**
1.  **Patch Sampling:** Randomly sample two patches of rays: one from the reference view and one from a randomly sampled unseen view [S3].
2.  **Rendering:** Render these patches using the current NeRF model to obtain predicted colors [S3].
3.  **Resize:** Resize rendered patches to $224 \times 224$ resolution to fit the DINO-ViT input [S3].
4.  **Depth Supervision via Warping:** Use image warping with known camera intrinsics and poses to create a reference depth map. This depth map constrains the unseen views [S1][S2, unofficial].
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ as a weighted sum of:
    -   **Photometric Loss:** L2 loss between rendered and ground truth (or warped) colors.
    -   **Adversarial Loss:** Patch-wise texture supervision using the CNN discriminator [S1].
    -   **Perceptual Loss:** Feature-level comparison to constrain unobserved views [S2, unofficial].
    -   **Global Structure Prior:** L2 loss between global image features extracted by the pretrained ViT from the reference patch and the unseen patch [S1].
6.  **Optimization:** Update MLP and discriminator weights using RAdam optimizer [S3].

**Parameters:**
-   **Optimizer:** RAdam [S3].
-   **Learning Rate:** Initial $1e-3$ for MLP; discriminator LR is $20\%$ of MLP's LR [S3].
-   **LR Decay:** Halved every 10k iterations [S3].
-   **Patch Sizes:**
    -   NeRF Synthetic (Blender): $64 \times 64$ [S3].
    -   LLFF: $84 \times 63$ [S3].
    -   DTU: $70 \times 56$ [S3].
-   **Stride Schedule:** Starts at 6, reduces by 2 every 10k iterations [S3].
-   **Loss Weights:** $\lambda_1, \lambda_2, \lambda_3$ initialized to $8, 0.1, 0$. $\lambda_2$ decreases linearly to 0, $\lambda_3$ increases linearly to 0.1 during training [S3].

**Edge Cases and Pitfalls:**
-   **Single View Ambiguity:** Without strong priors, the model may hallucinate geometry. The combination of adversarial, perceptual, and global feature losses is critical to mitigate this [S1].
-   **Depth Estimation Error:** The accuracy of the depth supervision relies on the correctness of the image warping and known camera parameters [S1].

## Subtopics

### SinNeRF Implementation Configuration
This entry details the specific hyperparameters and hardware setup used in the original SinNeRF implementation [S3]. The architecture matches the original NeRF paper [S3]. Training is conducted on an NVIDIA RTX A6000 GPU, taking several hours per scene [S3]. The patch sampling stride schedule and loss weight annealing are key implementation details for stable convergence [S3].

## Operating characteristics
-   **Time Complexity:** Training takes several hours per scene [S3].
-   **Memory Complexity:** Not given in the sources.
-   **Performance:** SinNeRF outperforms DS-NeRF, PixelNeRF, and DietNeRF on the NeRF Synthetic, DTU, and LLFF Forward Facing datasets [S1].
-   **Strengths:** Effective single-view reconstruction; leverages multiple complementary priors (texture, structure, depth) [S1].
-   **Limitations:** Requires known camera intrinsics and poses for depth warping [S1]. Performance is limited by the quality of the single input image and the accuracy of the warping-based depth estimation.

## Background
SinNeRF was introduced in April 2022 in the paper "SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image" [S3]. It was proposed to address the limitations of existing NeRF methods that require multiple views. The method integrates techniques from image warping, adversarial training, and global feature comparison to regularize the single-view reconstruction problem [S1]. It is cited as a significant advancement in single-view NeRF reconstruction, outperforming contemporaries like DS-NeRF and PixelNeRF [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
