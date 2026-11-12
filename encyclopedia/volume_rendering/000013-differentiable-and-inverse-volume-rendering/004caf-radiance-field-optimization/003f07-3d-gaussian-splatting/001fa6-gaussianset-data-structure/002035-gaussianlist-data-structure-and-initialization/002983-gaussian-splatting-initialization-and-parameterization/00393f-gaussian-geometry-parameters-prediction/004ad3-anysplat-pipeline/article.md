# AnySplat Pipeline

Also called: Feed-forward 3D Gaussian Splatting from Unconstrained Views

## Overview
AnySplat is a feed-forward neural network that predicts a set of 3D Gaussians directly from uncalibrated multi-view images, enabling instant novel-view synthesis without per-scene optimization. It differs from iterative inverse methods by using a transformer-based encoder-decoder to predict Gaussian parameters, depth, and camera poses in a single forward pass, followed by differentiable voxelization to handle density artifacts [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of $K$ uncalibrated images with shape $[1, K, 3, 448, 448]$ (batch, views, channels, height, width) [S1, unofficial].
*   **Outputs:** A set of 3D Gaussians (defined by mean $\mu$, covariance $\sigma$, rotation $r$, scale $s$, and color $c$) and predicted camera poses (extrinsic $E$ and intrinsic $K$) [S1, unofficial].

**Data Structures:**
The Gaussian parameters are predicted as dense feature maps or point clouds by the decoder heads. The "pre-voxel" Gaussians are constructed from these predictions. The specific internal layout of the Gaussian tensor is not explicitly detailed in the source, but standard 3D Gaussian Splatting conventions apply: $\mu \in \mathbb{R}^3$, $\sigma$ (or covariance matrix) $\in \mathbb{R}^{3 \times 3}$, $r \in \mathbb{R}^4$ (quaternion), $s \in \mathbb{R}^3$, $c \in \mathbb{R}^3$ [G].

**Steps:**
1.  **Encoding:** The input image tensor is processed by a transformer-based geometry encoder.
2.  **Decoding:** Three decoder heads ($F_G, F_D, F_C$) predict:
    *   $F_G$: Gaussian parameters ($\mu, \sigma, r, s, c$).
    *   $F_D$: Depth map $D$.
    *   $F_C$: Camera poses $p$ (extrinsic and intrinsic matrices) [S1, unofficial].
3.  **Construction:** Pixel-wise 3D Gaussians are constructed from the predicted parameters.
4.  **Voxelization:** A Differentiable Voxelization module converts the pixel-wise Gaussians into "pre-voxel" 3D Gaussians to resolve density issues and artifacts [S1, unofficial].
5.  **Rendering:** Multi-view images and depth maps are rendered from the voxelized Gaussians using standard differentiable rasterization [S1, unofficial].
6.  **Supervision (Training):**
    *   **RGB Loss:** Rendered images are compared against ground truth images.
    *   **Geometry Loss:** Rendered depth maps, decoded depth $D$, and predicted poses $p$ are used to compute geometry losses.
    *   **Pseudo-Geometry Priors:** Geometries are supervised by priors obtained from a pretrained VGGT model [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Input: images tensor of shape [1, K, 3, 448, 448], values in [-1, 1]
# Normalize to [0, 1] as per source convention
images_normalized = (images + 1.0) * 0.5

# Forward pass through the AnySplat model
# model includes the encoder and decoder heads (F_G, F_D, F_C)
gaussians, pred_context_pose = model.inference(images_normalized)

# Extract camera poses
pred_extrinsic = pred_context_pose['extrinsic']
pred_intrinsic = pred_context_pose['intrinsic']

# gaussians contains the predicted 3D Gaussian parameters
# These can be used for rendering or further processing
```

**Edge Cases and Pitfalls:**
*   **Uncalibrated Input:** The method handles uncalibrated views, but accuracy depends on the quality of the pseudo-geometry priors from VGGT [S1, unofficial].
*   **Density Artifacts:** Without the differentiable voxelization step, pixel-wise Gaussians may suffer from insufficient density and artifacts [S1, unofficial].

## Subtopics

### [SubSplat](13524-subsplat.md)
A high-resolution variant that uses a Sub-pixel Gaussian Reparameterizer to subdivide primary Gaussians into fine-grained primitives, improving fidelity from low-resolution backbone inputs.

## Operating characteristics
*   **Time Complexity:** Feed-forward inference is constant time with respect to scene complexity, depending only on the number of input views and image resolution. Training time is not explicitly given but involves supervising against VGGT priors and ground truth [S1, unofficial].
*   **Memory Complexity:** Depends on the number of predicted Gaussians and the resolution of the feature maps. The source does not specify exact memory usage [S1, unofficial].
*   **Performance:** Enables instant novel-view synthesis from unconstrained views. Specific metrics (PSNR, SSIM) are not provided in the source [S1, unofficial].
*   **Strengths:** No per-scene optimization required; handles uncalibrated inputs.
*   **Limitations:** Relies on the quality of the pretrained VGGT model for geometry priors; potential artifacts if voxelization is not properly tuned [S1, unofficial].

## Background
AnySplat was introduced by InternRobotics as a feed-forward approach to 3D Gaussian Splatting. It builds on [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by predicting parameters directly from images rather than optimizing them. The method uses a pretrained VGGT model for pseudo-geometry supervision [S1, unofficial].

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
