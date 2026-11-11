# Statistical Depth Refinement

Also called: Corrupt Depth Patching, Underfitting Depth Smoothing

## Overview
Statistical Depth Refinement is a training-time regularization technique that mitigates overfitting to noisy or corrupt depth priors (such as those from stereo matching in specular or fuzzy regions) by replacing outlier depth values with smoother estimates from an underfitting model. It differs from standard depth supervision [8687-depth-supervised-nerf.md] by actively identifying and patching corrupt pixels based on residual statistics rather than applying a uniform loss, and differs from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by using temporal model state (early training outputs) rather than static geometric priors.

## How it works
**Inputs:**
- $D_i$: The noisy stereo depth map for frame $i$.
- $\theta$: Current NeRF parameters.
- $K$: The iteration count at which the "underfitting" model state is captured.
- $\alpha$: The quantile threshold for identifying corrupt pixels ($\alpha \in [0, 1]$).

**Outputs:**
- $\hat{D}_i^{\text{patched}}$: The refined depth map for frame $i$, used for subsequent gradient updates.

**Data Structures:**
- Depth maps $D \in \mathbb{R}^{H \times W}$, stored as dense arrays of floating-point values.
- Residual maps $R \in \mathbb{R}^{H \times W}$, storing absolute differences.

**Steps:**
1. **Capture Underfitting State:** Train the NeRF for $K$ iterations. At iteration $K$, render the expected depth map $D_b^K_i$ for frame $i$ using the current parameters $\theta_K$. This map represents the model's smoothed, underfitting estimate.
   $$ D_b^K_i(\mathbf{r}) = \sum_{j=1}^{N} w_j t_j $$
   where $w_j$ and $t_j$ are the rendering weights and sample distances for ray $\mathbf{r}$ [S1, 8687-depth-supervised-nerf.md].

2. **Compute Residuals:** Calculate the absolute difference between the underfitting depth and the noisy stereo depth prior for each pixel $(u, v)$:
   $$ R_i[u, v] = | D_b^K_i[u, v] - D_i[u, v] | $$

3. **Determine Threshold:** Compute the $\alpha$-quantile of the residual distribution across all pixels in $R_i$. Let $Q_\alpha$ be this threshold value. Pixels with $R_i[u, v] > Q_\alpha$ are identified as having large residuals.

4. **Patch Corrupt Depths:** For each pixel $(u, v)$ where $R_i[u, v] > Q_\alpha$, replace the noisy depth $D_i[u, v]$ with the smoother underfitting depth $D_b^K_i[u, v]$. Pixels with residuals below the threshold remain unchanged.
   $$ \hat{D}_i^{\text{patched}}[u, v] = \begin{cases} D_b^K_i[u, v] & \text{if } R_i[u, v] > Q_\alpha \\ D_i[u, v] & \text{otherwise} \end{cases} $$

5. **Optimize:** Use $\hat{D}_i^{\text{patched}}$ as the target for the geometric loss $\mathcal{L}_{\text{geom}}$ in subsequent training iterations.

**Parameters:**
- $K = 4000$: The number of iterations to train before capturing the underfitting depth map [S1].
- $\alpha = 0.1$: The quantile threshold; pixels in the top 10% of residuals are considered corrupt [S1].
- $\lambda = 1$: Weight for the depth loss term in the total loss function [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The residual calculation must use sufficient precision to distinguish between minor noise and significant corruption [S1].
- **Boundary Conditions:** If all residuals are below the threshold (unlikely with $\alpha=0.1$ and noisy stereo data), no patching occurs. If all are above, the entire depth map is replaced by the underfitting estimate [S1].
- **Temporal Consistency:** The method assumes the underfitting model at iteration $K$ provides a valid "smooth" prior. If $K$ is too large, the model may have already overfit; if too small, the depth may be too noisy [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def statistical_depth_refinement(noisy_depth_map, nerf_model, k_iterations, alpha):
    """
    Refines a noisy depth map by patching outliers with underfitting model output.
    
    Args:
        noisy_depth_map: Tensor of shape (H, W), the stereo depth prior D_i.
        nerf_model: The NeRF model parameters theta.
        k_iterations: Integer, the iteration count K to capture underfitting state.
        alpha: Float in [0, 1], the quantile threshold.
        
    Returns:
        patched_depth_map: Tensor of shape (H, W), the refined depth.
    """
    # Step 1: Capture underfitting depth at iteration K
    # Note: In practice, this requires pausing training or saving checkpoint at K.
    # We assume nerf_model is the state at iteration K.
    underfitting_depth = nerf_model.render_expected_depth() # Shape (H, W)
    
    # Step 2: Compute residual maps
    residuals = torch.abs(underfitting_depth - noisy_depth_map)
    
    # Step 3: Determine threshold based on alpha-quantile
    # Flatten residuals to compute quantile
    flat_residuals = residuals.view(-1)
    threshold = torch.quantile(flat_residuals, 1.0 - alpha)
    
    # Step 4: Patch corrupt depths
    # Create mask for pixels with residuals > threshold
    corrupt_mask = residuals > threshold
    
    # Replace noisy depth with underfitting depth where mask is true
    patched_depth = torch.where(corrupt_mask, underfitting_depth, noisy_depth_map)
    
    return patched_depth
```

## Subtopics

### Perceptual Distillation
A technique that transfers deblurred texture information from pre-deblurred images to the NeRF using perceptual loss on rendered patches, overcoming geometric inconsistency of pre-deblurred images [S2]. It uses an off-the-shelf deep learning-based image deblurring network $\Theta_D$ to generate pre-deblurred images $\bar{I}$ from blurred inputs $I$ ($\bar{I} = \Theta_D(I)$) [S2]. Perceptual loss is applied by extracting features from both the pre-deblurred color patch $\bar{C}_{ptc}$ and the rendered color patch $\hat{C}_{ptc}$ (rendered from patch-wise sampled rays $r_{ptc}^{pd}$) using a shared pre-trained image feature extractor $\phi$ [S2]. A key pitfall is that pre-deblurred images cannot be used as direct pixel-wise color supervision due to lack of 3D consistency caused by the ill-posed property of image deblurring and independent processing across multi-view images [S2].

## Operating characteristics
- **Time Complexity:** Adds negligible overhead during training, primarily the cost of rendering an additional depth map at iteration $K$ and computing residuals. The patching operation is $O(HW)$ [S1].
- **Memory Complexity:** Requires storing the underfitting depth map $D_b^K_i$ temporarily, $O(HW)$ [S1].
- **Measured Performance:** Evaluated on robotic surgery stereo videos (DaVinci prostatectomy data, 807 frames, 15fps) [S1]. Metrics include PSNR, SSIM, and LPIPS [S1].
- **Strengths:** Effectively handles corrupt stereo depth caused by fuzzy pixels and specular highlights, preventing abrupt artifacts in reconstruction [S1].
- **Limitations:** Relies on the assumption that early training outputs are smoother and more accurate for corrupt regions, which may not hold for all scenes or noise distributions [S1].
- **When to Use:** When stereo depth priors are noisy due to specularities, motion blur, or textureless regions, and direct supervision leads to overfitting artifacts [S1].
- **When Not to Use:** When high-quality, clean depth priors are available, or when the scene geometry is simple enough that standard depth supervision suffices [S1].

## Background
Originated in the context of neural rendering for stereo 3D reconstruction of deformable tissues in robotic surgery [S1]. The method addresses the challenge of using stereo depth in surgical scenes where specular highlights and fuzzy pixels corrupt the depth estimates [S1]. Preliminary findings indicated that underfitting models at early training stages produce smoother depth results, which can be leveraged to correct corrupt depth [S1]. The method was implemented with a depth refinement iteration count $K=4000$ and threshold $\alpha=0.1$ [S1].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
