# Depth Regularization

Also called: DPT Depth Regularization, Monocular Depth Regularization

## Overview
Depth Regularization incorporates geometric priors from a monocular depth estimator (typically DPT) into the training of radiance fields or 3D Gaussian Splatting to enforce geometric consistency and suppress artifacts like floaters or geometry collapse [S1, S3]. It distinguishes itself from sibling methods by explicitly aligning the scale and offset of the monocular depth map using sparse Structure-from-Motion (SfM) points, rather than relying on scale-invariant losses or generalizable priors [S1]. Unlike [Edge-Aware Disparity Regularization](9140-edge-aware-disparity-regularization.md) which preserves edges via image gradients, or [DNGaussian](7399-dngaussian.md) which uses hard/soft regularization stages, this method focuses on per-frame SfM-based scaling and exponential decay weighting [S1].

## How it works
**Inputs:**
- Rendered depth map $\hat{D}$ from the scene representation (Gaussians or spheres).
- Monocular depth estimate $D$ from a pre-trained network (e.g., DPT) [S1].
- Sparse SfM points (inverse depth $D^{SFM}$) from COLMAP for the current view [S1].
- Camera intrinsics and extrinsics.

**Outputs:**
- Depth regularization loss $L_D$ and its gradients propagated to the scene representation parameters (Gaussian means, opacities, etc.) [S1].

**Data Structures:**
- **Monocular Depth Map $D$**: 2D array of inverse depth values predicted by DPT [S1].
- **SfM Inverse Depth $D^{SFM}$**: Sparse set of inverse depth values at known 3D point locations projected to the image plane [S1].
- **Rendered Depth $\hat{D}$**: Computed via alpha-compositing of Gaussian depths [S1].

**Steps:**

1. **Compute Scale and Offset for Monocular Depth:**
   Calculate the temporal median $t(D)$ and scale $s(D)$ for the monocular inverse depth map $D$ using the SfM points [S1]:
   $$
   t(D) = \text{median}(D)
   $$
   $$
   s(D) = \frac{1}{M} \sum_{i \in \text{SFM}} |D_i - t(D)|
   $$
   where $M$ is the number of SfM points in the image, and $i \in \text{SFM}$ are the indices of the SfM points [S1]. Note: The source formula for $s(D)$ is described as $s(D) = \frac{1}{M} |D_i - t(D)|$ with summation implied by the context of "scale" calculation based on SfM points [S1].

2. **Align Monocular Depth to Dataset Scale:**
   Transform the monocular inverse depth $D$ to the dataset scale $D^*$ using the SfM inverse depth statistics $t(D^{SFM})$ and $s(D^{SFM})$ [S1]:
   $$
   D^* = \frac{D + t(D^{SFM}) - t(D)}{s(D)} \cdot s(D^{SFM})
   $$
   This aligns the relative scale and offset of the monocular estimate with the sparse but metrically accurate SfM points [S1].

3. **Render Scene Depth:**
   Compute the rendered depth $\hat{D}$ for each pixel by alpha-compositing the depths of the primitives (e.g., Gaussians) along the ray [S1]. For a ray with $N$ primitives, where $T_i$ is the transmittance and $\alpha_i$ is the opacity of the $i$-th primitive, and $d_i$ is its depth:
   $$
   \hat{D} = \sum_{i=1}^{N} T_i \alpha_i d_i
   $$
   where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S1].

4. **Compute Depth Loss:**
   Calculate the difference between the rendered depth $\hat{D}$ and the aligned monocular depth $D^*$ [S1]:
   $$
   L_{raw} = \| \hat{D} - D^* \|
   $$
   The specific norm is not explicitly defined in the source text beyond the subtraction, but typically implies an $L_1$ or $L_2$ loss [S1].

5. **Apply Exponential Decay Weighting:**
   Weight the depth loss with an exponential decay factor $w(t)$ throughout the optimization, starting at 1 and ending at 0.01 [S1]:
   $$
   L_D = w(t) \cdot L_{raw}
   $$
   where $t$ is the current training step [S1].

6. **Backpropagation:**
   Propagate gradients from $L_D$ to the scene representation parameters. Specifically, gradients flow to each Gaussian's depth (mean position) and to the rendering $\alpha$ (opacity), which affects the Gaussian's contribution to the final depth and its screen-space position [S1].

**Parameters:**
- **Monocular Depth Estimator:** DPT [Ranftl et al. 2021] [S1].
- **Decay Start/End:** 1.0 to 0.01 [S1].
- **SfM Points:** Provided by COLMAP [S1].

**Edge Cases and Pitfalls:**
- **Sparse SfM Points:** If SfM points are insufficient, the scale/offset alignment may be inaccurate [S1].
- **Transient Objects:** The method assumes static geometry; transient objects may violate the depth prior [S1, Parent].
- **Precision:** Inverse depth is used to handle infinite depth and improve precision for distant objects [S1].

## Subtopics

### Opacity-Depth Regularization
A regularization term applied during novel-view synthesis optimization to encourage spheres to move to the correct scene depth [S2]. It sets itself apart by using an energy function based on opacity and depth rather than direct depth matching [S2].
- **Definition:** Encourages spheres to move to correct depth [S2].
- **Step:** Thresholding: Apply a threshold to spheres with low opacity to remove them [S2].
- **Formula:** Energy = $-z_i \cdot O_i$, where $z_i$ is the depth of sphere $i$ and $O_i$ is its opacity [S2].

### Sparse Input Reconstruction via Regularization
Methods that introduce additional constraints such as depth information to enhance detail and global consistency when observational data is sparse [S3]. It distinguishes itself by focusing on the trade-off between strong geometric constraints and computational cost in sparse-view scenarios [S3].
- **Definition:** Enhances detail and global consistency with sparse data [S3].
- **Pitfall:** Efficacy diminishes with very limited views (e.g., one view) [S3].
- **Comparison:** Relies on strong constraints and per-scene optimization, favoring geometric quality but incurring higher computation compared to generalizability-based methods [S3].
- **Usage:** DNGaussian uses a depth-regularized scheme [S3]. FSGS uses Gaussian Unpooling and depth regularization [S3]. MVSplat uses cost volume for geometry cues [S3].

### 2DGS + $\tilde{L}_d$ variant
A variant of 2D Gaussian Splatting (2DGS) that incorporates a depth regularization strategy by enabling supervision and gradient flow to depths within the CUDA backend rasterizer, supervised with sensor or monocular depth estimates [S4]. It sets itself apart by integrating depth supervision directly into the rasterizer's gradient flow [S4].
- **Definition:** 2DGS with depth regularization in CUDA rasterizer [S4].
- **Formula:** The overall optimization loss is defined as $L = L_{rgb} + \lambda_d \tilde{L}_d$, where $L_{rgb}$ is the original loss from 2DGS [16] and $\tilde{L}_d$ is the depth regularization term [S4].
- **Parameter:** $\lambda_d = 0.2$ [S4].

### Frustum Constraint Loss
A regularization term preventing Gaussians from drifting to unsupported regions of space by penalizing continuous violations of image bounds and valid depth ranges across input views [S5]. It distinguishes itself by using a soft penalty based on the minimum frustum violation across all input views [S5].
- **Definition:** Prevents Gaussians from drifting to unsupported regions [S5].
- **Step:** For each Gaussian $n$ with center $\mu_n$, transform to camera coordinates, project to image coordinates, and measure continuous violations of image bounds and valid depth range for each input view. If a Gaussian lies inside at least one input-view frustum, its penalty is zero. Otherwise, it receives a smooth positive penalty based on the minimum violation $v_n$ [S5].
- **Formula:** Compute frustum loss: $L_{fru} = \frac{1}{N} \sum_{n=1}^{N} \log(1 + v_n/\tau)$, where $v_n$ is the minimum frustum violation of Gaussian $n$ across input views, and $\tau$ controls softness [S5].
- **Parameter:** $\tau$: Parameter controlling the softness of the penalty [S5].

## Operating characteristics
- **Time Complexity:** Depends on the monocular depth estimation inference time (DPT) and the additional gradient computation for depth alignment [S1].
- **Memory Complexity:** Requires storing the monocular depth map and SfM points per view [S1].
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales with the number of training views and the density of SfM points [S1].
- **Strengths:** Improves geometric consistency and reduces floaters by leveraging strong monocular priors aligned with sparse SfM data [S1, S3].
- **Limitations:** Efficacy diminishes with very sparse input views (e.g., single view) [S3]. Relies on the accuracy of the monocular depth estimator and SfM reconstruction [S1].
- **When to Use:** When geometric consistency is critical and sparse SfM points are available to align the monocular depth scale [S1].
- **When Not to Use:** When input views are extremely sparse (e.g., single view) or when SfM reconstruction is unreliable [S3].

## Background
Depth Regularization using monocular depth estimators like DPT was introduced in the context of 3D Gaussian Splatting optimization to address geometry degradation in large datasets [S1]. The method builds on the observation that monocular depth estimates, while scale-ambiguous, provide strong geometric cues that can be aligned with sparse SfM points to provide metric supervision [S1]. It is used in training loops for radiance fields and Gaussian Splatting to improve novel view synthesis quality [S1].

## Sources
[S1] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://www.youtube.com/watch?v=p-mb2TzVJZk, fetched 2026-10-31
[S2] Pulsar: Efficient Sphere-based Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00149, fetched 2026-10-26
[S3] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S4] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S5] GlobalSplat: Efficient Feed-Forward 3D Gaussian Splatting via Global Scene Tokens, https://doi.org/10.1007/978-3-032-37595-7_10, fetched 2026-10-29
