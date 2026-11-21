# Epipolar 3D Gaussian Splatting

Also called: Epipolar Transformer-based 3DGS, PixelSplat-style 3DGS

## Overview
Epipolar 3D Gaussian Splatting is a feed-forward method for reconstructing 3D Gaussian Splatting (3DGS) scenes from one or more input images by leveraging epipolar geometry to generalize features across views [S1]. Unlike standard 3DGS which requires iterative optimization per scene, this approach predicts Gaussian parameters directly from image features, typically using an epipolar transformer to align and fuse multi-view information [S1]. It sets itself apart from iterative inverse rendering methods by offering real-time reconstruction capabilities, though it trades off the high-fidelity detail achievable through per-scene optimization [S1].

## How it works
**Inputs:** One or more input images with known camera parameters (intrinsics and extrinsics).
**Outputs:** A set of 3D Gaussian primitives defined by mean position $\mu$, covariance $\Sigma$, opacity $\alpha$, and spherical harmonics coefficients for color $c$.

**Data Structures:**
*   **Feature Maps:** 2D feature maps extracted from input images via a backbone network (e.g., ResNet or Transformer encoder).
*   **Epipolar Transformer:** A module that processes features along epipolar lines to enforce geometric consistency between views.
*   **Gaussian Predictor:** A decoder (e.g., FFN or U-Net) that maps fused features to Gaussian parameters.

**Steps:**
1.  **Feature Extraction:** Extract 2D feature maps $F_i$ for each input image $i$.
2.  **Epipolar Feature Aggregation:** Use an epipolar transformer to aggregate features from multiple views. For a target pixel, features from other views are sampled along the corresponding epipolar line to capture contextual information consistent with the camera geometry [S1].
3.  **Probabilistic Gaussian Prediction:** Predict the parameters of Gaussians aligned along camera-to-pixel rays. This involves estimating the depth distribution and projecting it into 3D Gaussian parameters [S1].
4.  **Parameter Decoding:** Decode the aggregated features into the full set of Gaussian attributes: position $\mu$, covariance $\Sigma$, opacity $\alpha$, and color $c$ [S1].
5.  **Rendering:** Render the predicted Gaussians using the standard 3DGS rasterization pipeline to produce the output image [S1].

**Formulas:**
The core innovation lies in the feature aggregation. Let $F_i(u, v)$ be the feature at pixel $(u, v)$ in view $i$. The epipolar transformer aggregates features from view $j$ along the epipolar line corresponding to $(u, v)$ in view $i$. The exact aggregation function is not detailed in the provided notes, but it relies on the epipolar constraint [S1].

**Parameters:**
*   Number of input views: Typically 2 or more for multi-view variants like MVSplat [S1].
*   Transformer depth and width: Not specified in the notes [S1].

**Edge Cases and Pitfalls:**
*   **Unknown Poses:** Standard epipolar methods require known camera poses. Variants like NoPoSplat address this by eliminating the need for known poses, but this introduces complexity [S1].
*   **Monocular Ambiguity:** Single-view methods (e.g., Flash3D) rely on depth estimation priors, which can be less accurate than multi-view epipolar constraints [S1].

## Subtopics

### Pose-Aware Projection (PAP)
A technique used in VS-Splat to extract multi-view contextual information for selected voxels by projecting skeletal Gaussian positions onto 2D feature maps and aggregating features based on camera distance [S2, unofficial]. It resolves ambiguous feature assignment from global backprojection by aligning selected 3D skeletal primitives with 2D feature maps from different camera views [S2, unofficial].
**Step:** Project positions of skeletal Gaussians $\{\mu^{\text{crs}}_{h,w,d} : (h, w, d) \in I\}$ onto each of $N$ pose-aware feature maps $\{f_n : n = 1, \dots, N\}$ obtained in the coarse stage, and sample features at the corresponding projected locations to create PAP-derived feature volumes $\{\bar{v}_{(h,w,d),n} : (h, w, d) \in I\}$ for each view $n$ [S2, unofficial].
**Formula:** The aggregated feature volume $\hat{v}_{h,w,d}$ is computed using inverse distance weighting:
$$ \hat{v}_{h,w,d} = \frac{\sum_{n=1}^N z_{(h,w,d),n}^{-1} \cdot \bar{v}_{(h,w,d),n}}{\sum_{n'=1}^N z_{(h,w,d),n'}^{-1}} $$
where $z_{(h,w,d),n}$ is the distance between the skeletal Gaussian position $\mu^{\text{crs}}_{h,w,d}$ and the $n$-th view camera position [S2, unofficial].
**Parameter:** $N$: Number of input camera views used for projection and aggregation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Feed-forward inference is significantly faster than iterative 3DGS optimization, enabling real-time reconstruction [S1].
*   **Memory Complexity:** Depends on the feature map size and transformer capacity. Multi-view methods require storing features for all input views [S1].
*   **Performance:** High-quality reconstruction for multi-view inputs. Monocular variants may suffer from depth ambiguity [S1].
*   **Strengths:** Real-time reconstruction, no per-scene optimization required [S1].
*   **Limitations:** Requires known camera poses (unless using variants like NoPoSplat) [S1]. Quality may be lower than optimized 3DGS for complex scenes [S1].

## Background
Epipolar 3D Gaussian Splatting builds on the foundation of 3D Gaussian Splatting [1795-3d-gaussian-splatting.md] and feed-forward reconstruction methods like PixelSplat [S1]. PixelSplat incorporates an epipolar transformer to generalize features between two views, pairing this with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays [S1]. MVSplat extends this by combining multi-view feature extraction through transformers and cost volumes [S1]. GS-LRM tokenizes images and camera parameters for transformer processing [S1]. NoPoSplat addresses the limitation of requiring known camera poses [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
