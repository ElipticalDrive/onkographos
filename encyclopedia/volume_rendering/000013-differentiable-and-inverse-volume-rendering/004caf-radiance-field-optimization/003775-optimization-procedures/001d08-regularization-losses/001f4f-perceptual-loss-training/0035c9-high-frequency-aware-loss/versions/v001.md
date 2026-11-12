# High-Frequency Aware Loss

Also called: LHF, Shallow VGG Loss

## Overview
High-Frequency Aware Loss (LHF) is a regularization term used in radiance-field optimization to preserve high-frequency details and prevent texture distortion. It addresses the limitation of standard perceptual losses, which often distort actual textures, by restricting feature extraction to the first block of a pre-trained VGG-19 model before any max-pooling operations. This approach improves high-frequency detail recovery while maintaining real textures, serving as a replacement or supplement to per-pixel MSE loss in alignment-aware training pipelines [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $G_i$ (ground truth patch after alignment), $R$ (rendered patch).
*   **Output:** Scalar loss value $L_{HF}$.

**Data Structures:**
*   $F$: The feature extractor, defined as the first block of a pre-trained VGG-19 model. This block consists of convolutional layers without subsequent max-pooling.
*   $C, W, H$: The channel, width, and height dimensions of the feature maps extracted by $F$.

**Steps:**
1.  Extract feature maps from the ground truth patch $G_i$ using the shallow VGG network: $F(G_i)$.
2.  Extract feature maps from the rendered patch $R$ using the same network: $F(R)$.
3.  Compute the squared L2 difference between the feature maps.
4.  Normalize by the total number of elements in the feature map ($C \times W \times H$) and a factor of $1/2$.

**Formula:**
$$L_{HF}(G_i, R) = \frac{1}{2CW H} \|F(G_i) - F(R)\|_2^2$$

Where:
*   $\| \cdot \|_2^2$ denotes the squared L2 norm (sum of squared differences).
*   $C, W, H$ are the dimensions of the extracted feature maps.
*   $G_i$ is the ground truth patch.
*   $R$ is the rendered patch.

**Parameters:**
*   $F$: First block of pre-trained VGG-19.
*   $w$: Weight multiplier for $L_{HF}$ in the total loss function (typically combined with patch-based MSE as $L_{PM} + w \cdot L_{HF}$) [S1].

**Edge Cases and Pitfalls:**
*   **Misalignment:** If the rendered image is misaligned with the ground truth, high-frequency details are lost, and LHF alone may not recover them without prior alignment or warping [S1].
*   **Texture Distortion:** Deeper VGG layers (standard perceptual loss) can distort textures; restricting to the first block mitigates this [S1].

## Subtopics

### VGG Loss for Image Patches
A variant of perceptual loss computed specifically on image patches using a VGG network. It encourages generated patches to have similar high-level features to real patches. Implementation involves extracting real and fake patches from outputs, batching them, slicing to include only the first `num_patches`, and computing the VGG loss between them. It is controlled by a multiplier `vgg_loss_mult` in the total loss computation [S2, unofficial].

## Operating characteristics
*   **Complexity:** Adds computational overhead proportional to the forward pass of the first VGG-19 block. Since it operates on patches, complexity scales with patch size and number of patches.
*   **Performance:** Improves high-frequency detail recovery in neural radiance fields, particularly when combined with alignment-aware training [S1].
*   **Strengths:** Preserves real textures better than deep perceptual losses; effective for high-fidelity rendering.
*   **Limitations:** Requires pre-alignment or warping to be effective against misalignment artifacts; relies on the specific architecture of VGG-19.
*   **When to use:** When high-frequency detail preservation is critical and standard MSE or deep perceptual losses result in blurry or distorted textures.

## Background
Originated in the AligNeRF paper (2023) by the authors of "AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training" [S1]. It modifies the original perceptual loss proposed by Johnson et al. [12] by using only the output of the first block before max-pooling. The method was introduced to address the trade-off between high-frequency detail preservation and texture distortion in image restoration and radiance field optimization tasks.

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
