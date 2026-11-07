# L1+SSIM Loss

Also called: 3D Gaussian Splatting Training Loss, Photometric Loss, SSIM+L1 Loss, Warped Photometric Loss

## Overview
L1+SSIM Loss is a composite photometric objective used to optimize radiance fields and 3D Gaussian splats by minimizing the difference between rendered images and ground-truth reference views. It combines a pixel-wise L1 error term with a Structural Similarity (SSIM) term to preserve local structures and edges while maintaining color accuracy. It is distinguished from pure MSE-based methods by its robustness to outliers and its ability to produce sharper details, and from visibility-based losses by its direct comparison of rendered and target pixel values without explicit occlusion reasoning.

## How it works
**Inputs and Outputs:**
- **Inputs:** Rendered image $\hat{\mathbf{I}}$, reference image $\mathbf{I}$, SSIM weight $\lambda$ (or $\lambda_s, \alpha$).
- **Outputs:** Scalar loss value $\mathcal{L}$ and gradients with respect to scene parameters (Gaussian attributes, neural network weights, etc.).

**Data Structures:**
- Images $\hat{\mathbf{I}}, \mathbf{I}$ are typically 3D tensors of shape $(H, W, 3)$ representing height, width, and RGB channels. Values are usually normalized to $[0, 1]$ or $[-1, 1]$.

**Formulas:**
The total loss is a weighted sum of the L1 loss and the SSIM loss [S1, unofficial][S2, unofficial]:

$$
\mathcal{L} = (1 - \lambda) \mathcal{L}_1(\hat{\mathbf{I}}, \mathbf{I}) + \lambda \mathcal{L}_{\text{SSIM}}(\hat{\mathbf{I}}, \mathbf{I})
$$

Where:
- $\mathcal{L}_1(\hat{\mathbf{I}}, \mathbf{I}) = \frac{1}{HW} \sum_{h,w} \| \hat{\mathbf{I}}_{h,w} - \mathbf{I}_{h,w} \|_1$ is the mean absolute RGB error [S1, unofficial].
- $\mathcal{L}_{\text{SSIM}}(\hat{\mathbf{I}}, \mathbf{I}) = 1 - \text{SSIM}(\hat{\mathbf{I}}, \mathbf{I})$ is the structural similarity loss, where SSIM is computed over local windows [S1, unofficial][S2, unofficial].

In some contexts, such as warped photometric consistency, the loss is applied between a warped image $\mathbf{I}_i'$ and a target image $\mathbf{I}_i$ [S3]:

$$
\mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{N}_i|} \sum_{j \in \mathcal{N}_i} \left( \alpha \frac{1 - \text{SSIM}(\mathbf{I}_i', \mathbf{I}_i)}{2} + (1 - \alpha) \| \mathbf{I}_i' - \mathbf{I}_i \| \right)
$$

**Parameters:**
- $\lambda$ (or $\lambda_s$): Weight for the SSIM term. The standard value is **0.2** [S1, unofficial][S2, unofficial].
- $\alpha$: Weight for the SSIM component in warped photometric loss. Typical value not explicitly stated in sources [S3].

**Steps:**
1. **Render:** Compute the rendered image $\hat{\mathbf{I}} = \mathcal{R}(\mathcal{G}; \pi, T)$ using the current scene parameters $\mathcal{G}$, camera pose $\pi$, and preset/transfer function $T$ [S1, unofficial].
2. **Compute L1:** Calculate the mean absolute difference between $\hat{\mathbf{I}}$ and $\mathbf{I}$ [S1, unofficial].
3. **Compute SSIM:** Calculate the Structural Similarity index between $\hat{\mathbf{I}}$ and $\mathbf{I}$ using a sliding window approach (typically $3\times3$ or $11\times11$ windows) [S1, unofficial]. The SSIM index is defined as:
   $$
   \text{SSIM}(\mathbf{x}, \mathbf{y}) = \frac{(2\mu_x\mu_y + C_1)(2\sigma_{xy} + C_2)}{(\mu_x^2 + \mu_y^2 + C_1)(\sigma_x^2 + \sigma_y^2 + C_2)}
   $$
   where $\mu_x, \mu_y$ are local means, $\sigma_x^2, \sigma_y^2$ are local variances, $\sigma_{xy}$ is the covariance, and $C_1, C_2$ are stabilization constants [S1, unofficial].
4. **Combine:** Compute the weighted sum $\mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda (1 - \text{SSIM})$ [S1, unofficial][S2, unofficial].
5. **Backpropagate:** Compute gradients of $\mathcal{L}$ with respect to the scene parameters. Gradients flow through the differentiable renderer $\mathcal{R}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l1_ssim_loss(rendered_image, target_image, ssim_weight=0.2, window_size=11):
    """
    Compute L1+SSIM loss between rendered and target images.
    
    Args:
        rendered_image: Tensor of shape (H, W, 3), values in [0, 1].
        target_image: Tensor of shape (H, W, 3), values in [0, 1].
        ssim_weight: Weight for SSIM term (lambda).
        window_size: Size of the SSIM window.
        
    Returns:
        loss: Scalar loss value.
    """
    # 1. Compute L1 Loss
    # Mean Absolute Error over all pixels and channels
    l1_loss = torch.mean(torch.abs(rendered_image - target_image))
    
    # 2. Compute SSIM Loss
    # SSIM is typically computed per channel or on luminance. 
    # Here we assume a standard implementation that returns a scalar SSIM value.
    # Note: SSIM ranges from -1 to 1, where 1 is perfect similarity.
    # We use 1 - SSIM to make it a loss.
    ssim_value = compute_ssim(rendered_image, target_image, window_size=window_size)
    ssim_loss = 1.0 - ssim_value
    
    # 3. Combine Losses
    # lambda_s = 0.2 is the standard weight for SSIM
    loss = (1.0 - ssim_weight) * l1_loss + ssim_weight * ssim_loss
    
    return loss

def compute_ssim(img1, img2, window_size=11):
    """
    Compute Structural Similarity Index.
    This is a simplified placeholder for the actual SSIM calculation.
    In practice, use a library like `torchmetrics` or implement the sliding window
    calculation with Gaussian weights.
    """
    # Constants for stability
    C1 = 0.01 ** 2
    C2 = 0.03 ** 2
    
    # Apply Gaussian filter to compute local means and variances
    # This step is omitted for brevity but is required for a correct implementation.
    # mu_x, mu_y = gaussian_filter(img1), gaussian_filter(img2)
    # sigma_x2, sigma_y2, sigma_xy = ...
    
    # For the purpose of this article, we assume a library function is available.
    # In a real implementation, this would involve convolutions.
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** SSIM computation involves division by small numbers; stabilization constants $C_1, C_2$ are critical [S1, unofficial].
- **Boundaries:** SSIM computation near image boundaries requires padding or window adjustment [G].
- **Degenerate Input:** If the rendered image is constant or the target image has no texture, SSIM may be undefined or unstable [G].

## Subtopics

### Warped Photometric Loss
Uses SSIM+L1 on images warped from neighboring views to enforce multi-view consistency. It differs from standard L1+SSIM by comparing warped projections rather than direct renders. Choose when multi-view consistency is more important than single-view accuracy. [DBARF](3012-dbarf.md)

### Adaptive Photometric Loss
Dynamically adjusts the loss based on pixel confidence or error magnitude. It differs by thresholding or weighting pixels differently. Choose when dealing with noisy or occluded regions. [Adaptive Photometric Loss](9354-adaptive-photometric-loss-with-thresholding.md)

## Operating characteristics
- **Time Complexity:** $O(HW)$ for L1. SSIM is $O(HW \cdot K^2)$ where $K$ is the window size. Overall complexity is dominated by SSIM computation.
- **Memory Complexity:** $O(HW)$ for storing intermediate gradients and images.
- **Performance:** SSIM+L1 is widely used in 3D Gaussian Splatting and NeRF optimization. It provides a good balance between sharpness (SSIM) and color fidelity (L1) [S2, unofficial].
- **Strengths:** Robust to outliers (L1), preserves edges and textures (SSIM).
- **Limitations:** SSIM is non-convex and can be sensitive to local structures. Computationally more expensive than MSE or L1 alone.
- **When to use:** Standard photometric optimization for radiance fields and Gaussian splats.
- **When not to use:** When real-time performance is critical and SSIM is too slow, or when structural similarity is not a desired property (e.g., textureless surfaces).

## Background
The L1+SSIM loss was introduced in the context of 3D Gaussian Splatting and NeRF optimization to improve visual quality over MSE-based losses. It builds on the SSIM metric proposed by Wang et al. (2004) [S1, unofficial]. It is a standard component in modern differentiable rendering pipelines [S2, unofficial].

## Sources
[S1] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
[S2] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S3] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
