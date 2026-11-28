# LPIPS Evaluation Protocol

Also called: Learned Perceptual Image Patch Similarity

## Overview
The LPIPS Evaluation Protocol assesses the perceptual quality of rendered images by comparing deep feature representations rather than raw pixel values, providing a metric where lower distances indicate higher perceptual similarity. It is preferred over pixel-space metrics like Mean Squared Error (MSE) because it accounts for neighboring information and structural context, offering a more robust evaluation for differentiable rendering [S1].

## How it works
The protocol computes the Learned Perceptual Image Patch Similarity (LPIPS) distance between a reference image and a rendered image.

**Inputs:**
- Reference image $I_{ref}$
- Rendered image $I_{pred}$
- Intensity scale: $[-1, 1]$ [S1]

**Data Structures:**
- Images are represented as tensors of shape $(H, W, C)$ or $(C, H, W)$, where $C=3$ for RGB.
- Values are normalized to the range $[-1, 1]$ [S1].

**Steps:**
1. **Normalization:** Ensure both $I_{ref}$ and $I_{pred}$ are scaled to $[-1, 1]$. If inputs are in $[0, 1]$, apply $x' = 2x - 1$ [S1].
2. **Feature Extraction:** Pass both images through a pre-trained convolutional neural network (e.g., VGG or AlexNet) to extract multi-scale feature maps. Let $F_{ref}$ and $F_{pred}$ be the feature maps at a specific layer.
3. **Per-Patch Comparison:** For each spatial location and channel, compute the normalized Euclidean distance between the feature vectors of $I_{ref}$ and $I_{pred}$.
4. **Aggregation:** Average the distances across all spatial locations and channels to produce a single scalar LPIPS score.

**Formula:**
The LPIPS distance $D$ is defined as:
$$ D(I_{ref}, I_{pred}) = \sum_{l} w_l \sum_{c} \frac{1}{H_l W_l} \sum_{i,j} \frac{(f_{l,c}(i,j)^{ref} - f_{l,c}(i,j)^{pred})^2}{2\sigma_{l,c}^2} $$
Where:
- $l$ indexes the network layers.
- $w_l$ is the weight for layer $l$.
- $c$ indexes the channels.
- $f_{l,c}(i,j)$ is the feature value at layer $l$, channel $c$, position $(i,j)$.
- $\sigma_{l,c}$ is the standard deviation of the feature values in the training dataset for that layer and channel, used for normalization [G].

**Parameters:**
- Intensity scale: $[-1, 1]$ [S1].

**Edge Cases:**
- Ensure consistent normalization between reference and prediction images to avoid biased distances.
- Degenerate inputs (e.g., constant color images) may yield non-zero LPIPS due to feature space variance.

## Subtopics

### [LPIPS Computation](5498-lpips-computation.md)
LPIPS (Learned Perceptual Image Patch Similarity) measures perceptual similarity between two images by comparing deep feature representations rather than raw pixel values, outputting a distance metric where lower values indicate higher similarity. It is preferred over pixel-space metrics like Mean Squared Error (MSE) because it accounts for neighboring information and structural context, providing a more robust optimization objective for differentiable rendering.

## Operating characteristics
LPIPS is computationally more expensive than PSNR or SSIM due to the need for feature extraction via a deep neural network. It provides a better correlation with human perception than pixel-based metrics. It is suitable for evaluating perceptual quality in radiance field optimization where structural fidelity is critical.

## Background
LPIPS was introduced as a learned perceptual image patch similarity metric. It has been adopted in various radiance field evaluation protocols, including those for 3D Gaussian Splatting variants, to address limitations of MSE-based metrics [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
