# Flash3D

Also called: Flash3D

## Overview
Flash3D is a feed-forward network (FFN) method for monocular 3D Gaussian Splatting (3DGS) reconstruction, generating 3DGS scenes by estimating depth from a single view [S1][S2, unofficial]. It differs from sibling methods like GS-LRM [1483-gs-lrm.md] and Large Reconstruction Model (LRM) [5959-large-reconstruction-model-lrm.md] by relying on monocular depth estimation rather than multi-view feature aggregation or tokenized multi-image inputs [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A single RGB image $I$.
- Output: A set of 3D Gaussian parameters $\{\mu_i, \Sigma_i, c_i, \alpha_i\}$ representing the scene [S1][S2, unofficial].

**Data Structures:**
The network outputs a list of Gaussians. Each Gaussian $i$ is defined by:
- $\mu_i \in \mathbb{R}^3$: Mean position.
- $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized as rotation $R \in SO(3)$ and scale $s \in \mathbb{R}^3$).
- $c_i \in \mathbb{R}^3$: Spherical harmonic coefficients for color.
- $\alpha_i \in [0, 1]$: Opacity.

**Steps:**
1. **Depth Estimation:** Estimate a dense depth map $D$ from the input image $I$ using a monocular depth estimation network [S1][S2, unofficial].
2. **Point Cloud Generation:** Convert the depth map $D$ and camera intrinsics into a 3D point cloud $P$. Each pixel $(u, v)$ with depth $d$ is back-projected to 3D space [S1][S2, unofficial].
3. **Gaussian Parameter Prediction:** The FFN predicts the Gaussian parameters ($\mu, \Sigma, c, \alpha$) for each point in $P$ [S1][S2, unofficial].
4. **Rendering:** Render the predicted Gaussians using standard differentiable rasterization to produce the output image [S1][S2, unofficial].
5. **Training:** Optimize the network parameters using photometric losses (e.g., LPIPS, SSIM) between the rendered output and the ground truth image [S1][S2, unofficial].

**Formulas:**
The monocular depth estimation step can be represented as:
$$ D = \varphi_d(\varphi_e(I)) $$
where $\varphi_e$ is the encoder and $\varphi_d$ is the decoder [S3].

**Parameters:**
- Loss functions: LPIPS, SSIM [S1][S2, unofficial].
- Backbone: Vision Transformer (ViT) or similar encoder-decoder architecture [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
- Monocular depth estimation may suffer from scale ambiguity. Supervised depth estimation methods (like BTS) provide correct scale, while self-supervised or zero-shot models require additional scale and drift estimation during training [S3].
- Degenerate input (e.g., textureless regions) may lead to inaccurate depth and thus poor Gaussian placement [S3].

## Subtopics

### Monocular Depth Estimation (BTS)
An encoder-decoder network that infers dense depth from a single image, supervised to have correct scale [S3]. It is used as the depth prior in Flash3D [S3]. Pitfall: Self-supervised or zero-shot pre-trained models require additional scale and drift estimation during training [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and Gaussians for rendering. The depth estimation and FFN inference are constant time per image.
- **Memory Complexity:** Proportional to the number of Gaussians and the depth map size.
- **Performance:** Fast inference as it is a feed-forward method [S1][S2, unofficial].
- **Strengths:** Single-view input, fast reconstruction [S1][S2, unofficial].
- **Limitations:** Depth estimation errors can propagate to Gaussian parameters [S3].

## Background
Flash3D was proposed to address the challenge of training feed-forward networks to directly predict 3D Gaussian parameters [S1][S2, unofficial]. It uses an FFN for monocular reconstruction, generating 3DGS scenes by estimating depth from a single view [S1][S2, unofficial]. It is part of a family of methods that includes PixelSplat [55] and MVSplat [58] [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
