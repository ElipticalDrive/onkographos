# GPS-Gaussian

Also called: Generalizable Point-based Gaussian Splatting

## Overview
GPS-Gaussian is a generalizable 3D Gaussian Splatting technique that estimates instant novel views from sparse source views without requiring per-scene fine-tuning or optimization [S2]. It solves the problem of slow, iterative optimization in standard 3D Gaussian Splatting by directly regressing Gaussian parameters from 2D image features [S1]. It is distinguished from its sibling [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by its feed-forward nature, and from [WildGaussians](6064-wildgaussians.md) by its specific focus on human subjects trained on large-scale human data [S2].

## How it works
**Inputs:** Sparse source views (images) and their corresponding camera parameters (intrinsics $K$, extrinsics).
**Outputs:** A set of 3D Gaussians defined by position $\mu$, color $c$, opacity $\alpha$, and covariance $\Sigma$, which can be rendered to novel views.

**Data Structures:**
The method operates on 2D Gaussian parameter maps defined on the sparse source view image plane.
- Feature maps: $f_s \in \mathbb{R}^{H/2 \times W/2 \times D_s}$, where $H, W$ are image dimensions and $D_s$ is the feature dimension [S2].
- Gaussian Maps: Pixel-wise maps for position $M_p$, color $M_c$, rotation $M_r$, scaling $M_s$, and opacity $M_\alpha$ [S2].

**Steps:**
1. **Feature Extraction:** Pass each source view $I$ through an image encoder $\epsilon_{img}$ to extract dense feature maps $f_s \in \mathbb{R}^{H/2 \times W/2 \times D_s}$ [S2].
2. **Correlation Volume Generation:** For a target view $I_{tar}$, select two neighboring source views $I_r$ and $I_l$. Generate a 3D correlation volume $C$ using the feature maps from these source views ($f_{r}, f_{l}$) [S2].
3. **Depth Estimation:** Iteratively estimate depth maps $\langle D_l, D_r \rangle$ using the correlation volume $C$ and camera parameters ($K_r, K_l$) via a depth estimation module $\phi_{depth}$ [S2]:
   $$ \langle D_l, D_r \rangle = \phi_{depth}(f_{l}, f_{r}, K_l, K_r) $$
   [S2]
4. **3D Position Unprojection:** Use the estimated depth and camera projection matrix to unproject pixel coordinates $x$ from the image plane to 3D coordinates $M_p(x)$ [S2].
5. **Color Assignment:** Set the color map $M_c(x)$ directly from the source image color $I(x)$ [S2].
6. **Gaussian Feature Generation:** Generate depth features using a depth encoder. Combine these with the image feature $f_s$ using concatenation ($\oplus$) and pass through a U-Net-like decoder to generate pixel-wise Gaussian features $\Gamma$ [S2]:
   $$ \Gamma = D_{parm}(\epsilon_{img}(I) \oplus \epsilon_{depth}(I)) $$
   [S2]
7. **Parameter Prediction:** Use three prediction heads to generate the remaining Gaussian feature maps: $M_r$ (rotation), $M_s$ (scaling), and $M_\alpha$ (opacity) [S2].
8. **Rendering:** Render the scene using the standard 3D Gaussian Splatting rasterization pipeline.

**Formulas:**
- **3D Gaussian Representation:**
  $$ G(x) = e^{-\frac{1}{2} (x-\mu)^T \Sigma^{-1} (x-\mu)} $$
  where $\mu$ is the 3D position and $\Sigma$ is the 3D covariance matrix [S2].
- **2D Covariance Projection:**
  The 3D covariance $\Sigma$ is decomposed into a rotation matrix $R$ and a scaling matrix $S$. The 2D covariance $\Sigma'$ is projected using the view transformation matrix $W$ and the Jacobian $J$ of the affine approximation of the projective transform [S2]:
  $$ \Sigma' = J W \Sigma W^T J^T $$
  [S2]
- **Alpha Blending:**
  The final pixel color $C$ is computed via alpha blending [S2]:
  $$ C = \sum_{i \in N} c_i \alpha'_i \prod_{j=1}^{i-1} (1 - \alpha'_j) $$
  where $c_i$ is the learned color and $\alpha'_i$ is the re-scaled opacity [S2].

**Parameters:**
- Feature map dimensions: $H/2 \times W/2 \times D_s$ [S2].
- Loss functions: Photometric loss and depth loss are used to optimize the method end-to-end [S2].

**Edge Cases and Pitfalls:**
- The method relies on the accuracy of the depth estimation module $\phi_{depth}$; errors in depth directly propagate to 3D position errors [S2].
- The method is designed for human subjects; performance on non-human scenes is not guaranteed as it is trained on large amounts of human data [S2].
- No fine-tuning or optimization is performed at inference time, so the quality is limited by the generalization capability of the trained network [S1].

## Subtopics

### [FaCT-GS](10199-fact-gs.md)
FaCT-GS is a framework for fast and flexible Computed Tomography (CT) reconstruction that optimizes the voxelization and rasterization pipelines of 3D Gaussian Splatting. It solves the problem of slow convergence and high computational cost in Gaussian-based tomographic reconstruction by removing view-dependent occlusion logic, coalescing data structures, and enabling rapid fitting to pre-existing volumes for warm-starting. It is distinguished from standard 3D Gaussian Splatting by its removal of depth-sorting overhead and from [GPS-Gaussian](9197-gps-gaussian.md) by its specific focus on CT geometry and voxelization-based initialization.

### [SegAnySplatting Feature Optimization](12209-seganysplatting-feature-optimization.md)
SegAnySplatting Feature Optimization is a method for optimizing semantic features for 3D Gaussians, incorporating scale-awareness via a learned gate and feature smoothing. It builds on [GPS-Gaussian](9197-gps-gaussian.md) by adding a mechanism to distill semantic information (e.g., from CLIP or LSeg) into the Gaussian attributes while handling the multi-scale nature of 3D Gaussians. It differs from [FaCT-GS](10199-fact-gs.md) by focusing on semantic feature distillation with scale-adaptive gating rather than compact factorized color representations.

### [Joint Fine-tuning](14680-joint-fine-tuning.md)
Joint Fine-tuning is a training strategy within the GPS-Gaussian framework that progressively co-optimizes a 3D Gaussian Splatting model and an Unsigned Distance Function (UDF) model to refine scene geometry. It addresses the ambiguity in pulling Gaussians to a surface by using dense queries around the UDF zero-level set as geometric cues, preventing error accumulation. Unlike methods that regress Gaussian parameters directly from sparse views without optimization, this approach iteratively improves both representations.

### Conditional Slicing
Conditional Slicing is a mechanism that projects high-dimensional (7D) Gaussians into 3D counterparts conditioned on specific time and view direction, preserving computational efficiency of 3DGS [S3]. It conditions the 6D Gaussian on the viewing direction $d$ (where $X_d = d$) to yield a conditional 3D Gaussian for the spatial component, using the conditional distribution formulas from 6DGS [S3]. The conditional mean is calculated as $\mu_{cond} = \mu_p + \Sigma_{pd} \Sigma_d^{-1} (d - \mu_d)$, where $\mu_p$ is the spatial mean, $\Sigma_{pd}$ is the cross-covariance between position and direction, $\Sigma_d$ is the directional covariance, $d$ is the viewing direction, and $\mu_d$ is the directional mean [S3]. The conditional covariance is calculated as $\Sigma_{cond} = \Sigma_p - \Sigma_{pd} \Sigma_d^{-1} \Sigma_{pd}^\top$, where $\Sigma_p$ is the spatial covariance [S3]. The opacity of each Gaussian is modulated to reflect the alignment between the current view direction and the Gaussian’s preferred direction: $f_{cond} = \exp(-\lambda (d - \mu_d)^\top \Sigma_d^{-1} (d - \mu_d))$, $\alpha_{cond} = \alpha \cdot f_{cond}$, where $\lambda$ is a positive scaling parameter controlling the sensitivity of the opacity modulation to view direction alignment [S3].

## Operating characteristics
- **Time Complexity:** The method is feed-forward, allowing for instant novel view estimation without iterative optimization [S1, S2].
- **Memory Complexity:** Requires storing feature maps and the correlation volume during inference, but avoids storing large per-scene optimization states [S2].
- **Performance:** Enables real-time rendering speeds [S2].
- **Strengths:** Eliminates the need for per-scene optimization, making it suitable for applications requiring immediate novel view synthesis from sparse inputs [S2].
- **Limitations:** Limited to human subjects due to training data bias [S2]. Accuracy depends on the depth estimation module [S2].

## Background
GPS-Gaussian was introduced as a generalizable approach for 3D human representation that takes advantage of 2D Gaussian parameter maps to estimate human features and 3D pose in real-time [S2]. It is cited as reference [73] in the source document [S2]. It builds on the foundation of 3D Gaussian Splatting to overcome the performance constraints of using the same set of Gaussians for various human features which may have different frequencies [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] HFGaussian: Learning Generalizable Gaussian Human with Integrated Human Features, https://doi.org/10.48550/arXiv.2411.03086, fetched 2026-10-26
[S3] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
