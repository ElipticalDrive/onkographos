# LPIPS Evaluation Protocol

Also called: Learned Perceptual Image Patch Similarity

## Overview
The LPIPS Evaluation Protocol computes the Learned Perceptual Image Patch Similarity distance between reference and rendered images to assess perceptual quality [S1]. It is used in radiance field evaluation protocols, including those for 3D Gaussian Splatting variants [S1].

## How it works
**Inputs:**
- Reference image $I_{ref}$
- Rendered image $I_{pred}$
- Intensity scale: $[-1, 1]$ [S1]

**Data Structures:**
- Images are represented as tensors of shape $(H, W, C)$ or $(C, H, W)$, where $C=3$ for RGB. Not given in the sources.
- Values are normalized to the range $[-1, 1]$ [S1].

**Steps:**
1. **Normalization:** Scale both $I_{ref}$ and $I_{pred}$ to the intensity range $[-1, 1]$ [S1]. Not given in the sources.
2. **Feature Extraction:** Pass both images through a pre-trained convolutional neural network to extract multi-scale feature maps. Not given in the sources.
3. **Per-Patch Comparison:** Compute the normalized Euclidean distance between the feature vectors of $I_{ref}$ and $I_{pred}$ at each spatial location and channel. Not given in the sources.
4. **Aggregation:** Average the distances across all spatial locations and channels to produce a single scalar LPIPS score. Not given in the sources.

**Formula:**
The LPIPS distance $D$ is defined as:
$$ D(I_{ref}, I_{pred}) = \sum_{l} w_l \sum_{c} \frac{1}{H_l W_l} \sum_{i,j} \frac{(f_{l,c}(i,j)^{ref} - f_{l,c}(i,j)^{pred})^2}{2\sigma_{l,c}^2} $$
Where:
- $l$ indexes the network layers. Not given in the sources.
- $w_l$ is the weight for layer $l$. Not given in the sources.
- $c$ indexes the channels. Not given in the sources.
- $f_{l,c}(i,j)$ is the feature value at layer $l$, channel $c$, position $(i,j)$. Not given in the sources.
- $\sigma_{l,c}$ is the standard deviation of the feature values in the training dataset for that layer and channel, used for normalization [G].

**Parameters:**
- Intensity scale: $[-1, 1]$ [S1].

**Edge Cases:**
- Ensure consistent normalization between reference and prediction images to avoid biased distances. Not given in the sources.
- Degenerate inputs (e.g., constant color images) may yield non-zero LPIPS due to feature space variance. Not given in the sources.

## Subtopics

### [LPIPS Computation](5498-lpips-computation.md)
LPIPS measures perceptual similarity between original and reconstructed images by comparing deep features extracted from neural networks, outputting a distance metric where lower values indicate higher similarity. Not given in the sources. It is preferred over pixel-space metrics like Mean Squared Error (MSE) because it accounts for neighboring information and structural context, providing a more robust optimization objective for differentiable rendering. Not given in the sources.

## Operating characteristics
LPIPS is computationally more expensive than PSNR or SSIM due to the need for feature extraction via a deep neural network. Not given in the sources. It provides a better correlation with human perception than pixel-based metrics. Not given in the sources. It is suitable for evaluating perceptual quality in radiance field optimization where structural fidelity is critical. Not given in the sources.

## Background
LPIPS was introduced as a learned perceptual image patch similarity metric. Not given in the sources. It has been adopted in various radiance field evaluation protocols, including those for 3D Gaussian Splatting variants, to address limitations of MSE-based metrics [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
