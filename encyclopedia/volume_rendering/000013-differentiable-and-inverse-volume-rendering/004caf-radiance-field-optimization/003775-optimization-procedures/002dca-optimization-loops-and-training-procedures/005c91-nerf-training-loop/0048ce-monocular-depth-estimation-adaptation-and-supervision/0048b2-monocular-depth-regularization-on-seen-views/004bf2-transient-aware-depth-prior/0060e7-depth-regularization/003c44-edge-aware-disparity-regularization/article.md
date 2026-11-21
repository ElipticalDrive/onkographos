# Edge-Aware Disparity Regularization

Also called: Edge-Aware Depth Regularization, Edge-Aware Smoothness Regularization, Edge-Preserving Depth Regularization

## Overview
Edge-Aware Disparity Regularization is a loss term that encourages local depth smoothness while preserving depth discontinuities that are aligned with image edges [S1, unofficial][S2]. It solves the problem of noisy or inconsistent depth maps in neural radiance fields by leveraging the correlation between geometric boundaries and color gradients [S1, unofficial][S2]. It sets itself apart from its parent, [Depth Regularization](18191-depth-regularization.md), by explicitly weighting the smoothness constraint with image gradients to prevent blurring across object boundaries [S1, unofficial][S2]. It differs from siblings like [SparseGS](6135-sparsegs.md) and [PixelSplat Training and Depth Regularization](6420-pixelsplat-training-and-depth-regularization.md) by focusing on patch-based gradient penalties on rendered views rather than point-cloud or splat-based constraints [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered disparity map (or inverse depth) $D$ (or $d$) and corresponding RGB image $I$ from a sampled view or patch.
*   **Outputs:** Scalar regularization loss value $L_{reg}$ or $L_{smooth}$.

**Data Structures:**
*   **Disparity/Depth Map:** A 2D tensor of shape $(H, W)$ representing per-pixel depth or disparity values.
*   **RGB Image:** A 2D or 3D tensor of shape $(H, W, 3)$ representing pixel colors.
*   **Gradients:** First-order partial derivatives computed via finite differences or Sobel filters.

**Steps:**
1.  **Sample Views:** Render additional patches from training and test viewpoints [S2].
2.  **Normalize (Optional but Recommended):** Compute mean-normalized inverse depth to discourage depth shrinking [S1, unofficial].
    $$ D^*_i = \frac{D_i}{\bar{D}_i} $$
    where $D_i$ is the inverse depth and $\bar{D}_i$ is the mean inverse depth of the patch or view [S1, unofficial]. Note: Some implementations use raw disparity $d^*_t$ directly without normalization [S2].
3.  **Compute Gradients:** Calculate the first-order gradients of the depth/disparity map and the RGB image in the x and y directions [S2].
    *   Depth gradients: $\partial_x D^*$, $\partial_y D^*$ (or $\partial_x d^*$, $\partial_y d^*$) [S1, unofficial][S2].
    *   Image gradients: $\partial_x I$, $\partial_y I$ (typically computed on luminance or per-channel and aggregated) [S1, unofficial][S2].
4.  **Compute Edge Weights:** Calculate the exponential of the negative image gradients to create weights that suppress regularization at edges [S1, unofficial][S2].
    $$ w_x = e^{-|\partial_x I|}, \quad w_y = e^{-|\partial_y I|} $$
5.  **Calculate Loss:** Compute the weighted L1 norm of the depth gradients [S1, unofficial][S2].

**Formula:**
The regularization loss is defined as [S1, unofficial][S2]:
$$ L_{reg} = |\partial_x D^*_i| e^{-|\partial_x I_i|} + |\partial_y D^*_i| e^{-|\partial_y I_i|} $$
Or equivalently using disparity $d^*_t$ and RGB $I_t$ [S2]:
$$ L_{smooth} = |\partial_x d^*_t| e^{-|\partial_x I_t|} + |\partial_y d^*_t| e^{-|\partial_y I_t|} $$

**Parameters:**
*   **Weighting Factors:** $\lambda_1$ for depth consistency and $\lambda_2$ for smoothness [S2].
*   **Typical Values:**
    *   Early training: $\lambda_1 = 0.001, \lambda_2 = 0$ [S2].
    *   Later training: $\lambda_1 = 0, \lambda_2 = 1.0$ [S2].
*   **Normalization:** Mean normalization factor $\bar{D}_i$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Gradients are sensitive to noise; rendering artifacts in NeRF can create spurious depth gradients [S2].
*   **Boundaries:** The method assumes local smoothness; global consistency is not enforced by this term alone [S1, unofficial].
*   **Depth Shrinking:** Without mean normalization ($D^*$), the optimizer may shrink depth estimates to minimize the L1 gradient penalty [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_aware_disparity_regularization(depth_map, rgb_image, use_normalization=True):
    """
    Computes edge-aware disparity regularization loss.
    
    Args:
        depth_map: Tensor of shape (H, W) or (B, H, W). Disparity or inverse depth.
        rgb_image: Tensor of shape (H, W, 3) or (B, H, W, 3). RGB values.
        use_normalization: Boolean. If True, normalizes depth by its mean.
        
    Returns:
        loss: Scalar tensor.
    """
    # 1. Normalize depth if requested
    if use_normalization:
        mean_depth = torch.mean(depth_map)
        # Guard against division by zero
        if mean_depth == 0:
            mean_depth = 1e-8
        normalized_depth = depth_map / mean_depth
    else:
        normalized_depth = depth_map

    # 2. Compute gradients
    # Using simple finite differences for x and y
    # Pad with zeros to handle boundaries
    pad = 1
    depth_padded = torch.nn.functional.pad(normalized_depth, (pad, pad, pad, pad), mode='constant', value=0)
    rgb_padded = torch.nn.functional.pad(rgb_image, (pad, pad, pad, pad, 0, 0), mode='constant', value=0)
    
    # Gradient in x: D[i, j+1] - D[i, j]
    dx_depth = depth_padded[:, 1:-1, 2:] - depth_padded[:, 1:-1, :-2]
    dy_depth = depth_padded[:, 2:, 1:-1] - depth_padded[:, :-2, 1:-1]
    
    # Gradient in x and y for RGB (using luminance or mean channel for edge detection)
    # Converting RGB to grayscale for edge detection is common, but sources imply direct gradient usage.
    # We compute magnitude of RGB gradient.
    dx_rgb = rgb_padded[:, 1:-1, 2:] - rgb_padded[:, 1:-1, :-2]
    dy_rgb = rgb_padded[:, 2:, 1:-1] - rgb_padded[:, :-2, 1:-1]
    
    # Compute magnitude of RGB gradient
    rgb_grad_mag = torch.sqrt(torch.sum(dx_rgb**2, dim=-1) + torch.sum(dy_rgb**2, dim=-1))
    
    # 3. Compute edge weights
    # e^(-|grad|)
    weights_x = torch.exp(-torch.abs(dx_rgb))
    weights_y = torch.exp(-torch.abs(dy_rgb))
    
    # 4. Compute loss
    # L = |dDx| * wx + |dDy| * wy
    loss_x = torch.abs(dx_depth) * weights_x
    loss_y = torch.abs(dy_depth) * weights_y
    
    loss = torch.mean(loss_x + loss_y)
    
    return loss
```

## Subtopics

### [Geometric Regularization](12086-geometric-regularization.md)
Geometric Regularization is a loss term imposed on rendered depth patches from sampled views to regularize geometry. It smooths the depth of unobserved viewpoints to alleviate density holes caused by sparsely observed viewpoints, inspired by the piece-wise smoothing characteristics of real-world geometry. It sets itself apart from its parent, Edge-Aware Disparity Regularization, by applying a uniform smoothness constraint on depth patches without explicitly weighting by image edges.

### Disparity Smoothing
Disparity Smoothing is a technique to reduce misalignments around the edges of a reference mask by smoothing the aligned reference disparity to improve visual continuity between the masked and unmasked regions [S3]. It computes the edge-smoothed disparity $D_r^{smooth}$ as the sum of the aligned disparity $D_r$ and a correction term $D_{correction}$: $D_r^{smooth} = D_r + D_{correction}$ [S3]. Without smoothing, unsmoothed geometry has a much more jarring transition between the masked and unmasked areas, particularly close to the mask boundary [S3].

### Disparity Field-Guided Deep Re-regularization
Disparity Field-Guided Deep Re-regularization is a module proposed to raise the target view image $I_{t0}$ into a locally immersive Light Field $\hat{L}_{t0}$ by back-warping and deep spatial-angular regularization [S4]. It inputs the target view image $I_{t0}$ and refined disparity $\hat{D}_{t0}$, back-warping pixels of the target view image with the estimated accurate disparity prior, and further refines the output by spatial-angular convolutional blocks to produce the synthesized immersive LF $\hat{L}_{t0}$ [S4].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the sampled patches, as it involves computing gradients and element-wise operations [S1, unofficial][S2].
*   **Memory Complexity:** Linear with respect to the number of pixels, storing depth and RGB tensors and their gradients [S1, unofficial][S2].
*   **Measured Performance:** Effective in mitigating high-frequency noise and artifacts in flat regions while preserving details [S2]. In LEVIR-NVS dataset experiments, it was activated in later training stages with $\lambda_2 = 1.0$ [S2].
*   **Scaling:** Scales well with resolution as it is a local operation [S1, unofficial][S2].
*   **Strengths:** Preserves sharp edges in depth maps corresponding to object boundaries; reduces noise in textureless or flat regions [S1, unofficial][S2].
*   **Limitations:** Relies on the quality of rendered RGB images; if RGB edges are noisy, depth edges may be incorrectly preserved or smoothed [S1, unofficial][S2]. Not suitable for early training stages where depth estimates are unstable [S2].
*   **When to use:** When depth maps exhibit noise in flat regions but need to preserve object boundaries; in later stages of NeRF training [S2].
*   **When not to use:** When RGB images are highly noisy or lack clear edges; in early training stages where depth consistency is more critical than smoothness [S2].

## Background
Edge-Aware Disparity Regularization originates from the work on GeCoNeRF, which introduced the term to improve geometric consistency in few-shot neural radiance fields [S1, unofficial]. It was inspired by Godard et al. (2017), who used similar edge-aware weights for monocular depth estimation [S1, unofficial]. The technique was later adapted in TriDF for remote sensing novel view synthesis, where it was applied in the latter stages of training to enhance geometric structure [S2]. It is used in scenarios where sparse views lead to inconsistent depth maps, such as few-shot learning and remote sensing [S1, unofficial][S2].

## Sources
[S1] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
[S2] TriDF: Triplane-Accelerated Density Fields for Few-Shot Remote Sensing Novel View Synthesis, https://doi.org/10.1109/tgrs.2025.3643432, fetched 2026-10-30
[S3] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
[S4] Scale-Consistent Fusion: From Heterogeneous Local Sampling to Global Immersive Rendering, https://doi.org/10.1109/tip.2022.3205745, fetched 2026-10-26
