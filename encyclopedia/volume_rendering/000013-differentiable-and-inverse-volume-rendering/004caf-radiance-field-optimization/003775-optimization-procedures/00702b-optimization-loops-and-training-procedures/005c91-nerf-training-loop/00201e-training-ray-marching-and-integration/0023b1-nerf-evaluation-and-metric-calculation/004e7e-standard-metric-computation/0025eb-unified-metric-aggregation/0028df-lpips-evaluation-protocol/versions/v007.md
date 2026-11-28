# LPIPS Evaluation Protocol

Also called: Learned Perceptual Image Patch Similarity

## Overview
LPIPS is a full-reference perceptual quality metric that measures image similarity by comparing deep feature representations extracted from a pretrained convolutional neural network [S1, S3]. Unlike pixel-wise metrics such as PSNR or MSE, which often correlate poorly with human perception, LPIPS aligns closely with human judgments of image similarity by leveraging semantic features [S2][S5, unofficial]. It is primarily used in volume rendering and NeRF optimization to evaluate reconstruction fidelity or as a perceptual loss function to guide training, capturing nuances like texture and structure that standard metrics miss [S3][S5, unofficial].

## How it works
The protocol evaluates the perceptual distance between a rendered image and a ground-truth reference image. The core distinction from pixel-wise metrics is the use of deep features to model human visual perception [S3].

**Inputs and Outputs:**
- Input: Rendered image $I_{render}$ and ground-truth image $I_{gt}$.
- Output: Scalar LPIPS score $L \in [0, \infty)$. Lower scores indicate higher perceptual similarity.

**Data Structures:**
- Feature maps: Tensors extracted from multiple layers of a pretrained network (e.g., VGG16 or AlexNet).
- Normalization parameters: Per-channel mean and standard deviation computed from the ImageNet dataset [S3].

**Steps:**
1. **Intensity Scaling:** Scale pixel values of both $I_{render}$ and $I_{gt}$ to the range $[-1, 1]$ [S1]. This is critical for correct evaluation, as noted in comparisons with 3DGS variants [S1].
2. **Feature Extraction:** Pass both images through a pretrained convolutional neural network (typically VGG16 [S3]). Extract feature maps from multiple layers $l = 1, \dots, L$.
3. **Normalization:** Normalize the feature maps at each layer using the mean and standard deviation of the features from the training dataset (ImageNet) [S3]. Let $\mu_l$ and $\sigma_l$ be the mean and standard deviation for layer $l$. The normalized feature map $\hat{F}_l$ is:
   $$ \hat{F}_l = \frac{F_l - \mu_l}{\sigma_l} $$
4. **Distance Calculation:** For each layer $l$, compute the squared Euclidean distance between the normalized feature maps of the rendered and ground-truth images. Let $F_l^{render}$ and $F_l^{gt}$ be the feature maps. The distance $d_l$ is:
   $$ d_l = \sum_{i,j} \left( \hat{F}_{l, i,j}^{render} - \hat{F}_{l, i,j}^{gt} \right)^2 $$
   where $i,j$ index the spatial dimensions of the feature map.
5. **Weighting:** Apply learned or fixed weights $w_l$ to each layer's distance to account for the perceptual importance of different scales [S3].
6. **Aggregation:** Sum the weighted distances across all layers to obtain the final LPIPS score:
   $$ L = \sum_{l=1}^{L} w_l d_l $$

**Parameters:**
- Intensity scale: $[-1, 1]$ [S1].
- Network: VGG16 is commonly used [S3].
- Weights: Learned via a regression task against human judgments [S3].

**Edge Cases and Pitfalls:**
- **Intensity Range:** Failure to scale inputs to $[-1, 1]$ leads to incorrect LPIPS values, as highlighted in evaluations of 3DGS variants [S1].
- **Network Choice:** Different networks (e.g., AlexNet vs. VGG16) yield different scores. Consistency in network choice is required for fair comparison [S3].

## Subtopics

### [LPIPS Computation](5498-lpips-computation.md)
LPIPS is a full-reference perceptual quality metric that measures the similarity between two images by comparing deep feature representations extracted from a pretrained convolutional neural network. Unlike pixel-wise metrics such as PSNR or MSE, which often correlate poorly with human perception, LPIPS aligns closely with human judgments of image similarity by leveraging semantic features. It is primarily used in volume rendering and NeRF optimization to evaluate reconstruction fidelity or as a perceptual loss function to guide training, capturing nuances like texture and structure that standard metrics miss.

## Operating characteristics
- **Time Complexity:** Dominated by the forward pass of the pretrained CNN. Typically fast enough for offline evaluation but slower than PSNR/SSIM.
- **Memory Complexity:** Requires storing feature maps from multiple layers.
- **Strengths:** High correlation with human perception [S2][S5, unofficial]. Captures texture and structural details better than pixel-wise metrics [S3].
- **Limitations:** Computationally more expensive than PSNR/SSIM. Requires a pretrained network.
- **When to Use:** When perceptual quality is more important than pixel-level accuracy. Essential for evaluating NeRFs where visual fidelity is key [S3].
- **When Not to Use:** For real-time feedback during training if computational resources are limited. For tasks requiring strict pixel-level alignment.

## Background
LPIPS was introduced as a perceptual similarity metric that leverages deep features to better align with human judgment [S3]. It has become a standard metric in computer vision and graphics, particularly for evaluating generative models and novel view synthesis techniques like NeRFs and 3DGS [S1, S3]. Its adoption in volume rendering evaluation protocols ensures that reported results reflect perceptual quality rather than just pixel-wise error [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
