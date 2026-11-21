# Self-Supervised Depth Losses
## Overview
Self-supervised depth losses constrain the estimated depth and confidence intervals in a self-supervised manner using monocular depth estimates and image gradients, addressing the difficulty of obtaining ground truth depth [S1]. They are distinguished from sibling methods by their reliance on explicit smoothness priors derived from monocular disparity maps rather than reprojection consistency or pseudo-depth supervision [S1, S2].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered depth map $\hat{D}$, monocular depth estimate $\tilde{D}_{mono}$, aligned RGB image $I$, and image dimensions $H, W$.
*   **Outputs:** Scalar loss value $\mathcal{L}_{smooth}$.

**Data Structures:**
*   $\hat{D}$: Tensor of shape $(H, W)$ containing per-pixel depth values.
*   $\tilde{D}_{mono}$: Tensor of shape $(H, W)$ containing per-pixel monocular depth/disparity values.
*   $I$: Tensor of shape $(H, W, C)$ containing RGB pixel values.

**Steps:**
1.  Compute spatial gradients of the rendered depth map $\hat{D}$ in horizontal ($x$) and vertical ($y$) directions: $\partial_x \hat{D}$ and $\partial_y \hat{D}$.
2.  Compute spatial gradients of the aligned RGB image $I$ in horizontal and vertical directions: $\partial_x I$ and $\partial_y I$.
3.  Calculate the smoothness loss using the formula below.

**Formula:**
The smoothness loss $\mathcal{L}_{smooth}$ is defined as [S1]:

$$
\mathcal{L}_{smooth} = \frac{1}{HW} \sum_{r} \left( \|\partial_x \hat{D}\| (e^{-|\partial_x I(r)|} + e^{-|\partial_x \tilde{D}_{mono}(r)|}) + \|\partial_y \hat{D}\| (e^{-|\partial_y I(r)|} + e^{-|\partial_y \tilde{D}_{mono}(r)|}) \right)
$$

Where:
*   $r$ iterates over all pixels $(x, y)$.
*   $\|\cdot\|$ denotes the magnitude of the gradient.
*   $e^{-|\cdot|}$ acts as a weight that suppresses the loss at edges (high gradient magnitude) in the RGB image or monocular depth map.
*   $H, W$ are the height and width of the image.

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Gradients near zero can cause numerical instability in the exponential weights if not clamped.
*   **Boundaries:** Gradient computation at image borders requires padding or boundary handling.
*   **Degenerate Input:** If the monocular depth estimate $\tilde{D}_{mono}$ is noisy or flat, the smoothness prior may be ineffective.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_smoothness_loss(rendered_depth: Tensor, mono_depth: Tensor, rgb_image: Tensor) -> float:
    """
    Computes the self-supervised smoothness loss.
    
    Args:
        rendered_depth: Tensor of shape (H, W), depth map D_hat.
        mono_depth: Tensor of shape (H, W), monocular depth estimate D_tilde_mono.
        rgb_image: Tensor of shape (H, W, 3), aligned RGB image I.
        
    Returns:
        Scalar loss value.
    """
    H, W = rendered_depth.shape
    
    # Compute gradients for rendered depth
    # Using central differences for interior, forward/backward for boundaries
    dx_d = rendered_depth[:, 1:] - rendered_depth[:, :-1]
    dy_d = rendered_depth[1:, :] - rendered_depth[:-1, :]
    
    # Compute gradients for RGB image
    # Assuming RGB image is normalized or handled appropriately
    dx_i = rgb_image[:, 1:, :] - rgb_image[:, :-1, :]
    dy_i = rgb_image[1:, :, :] - rgb_image[:-1, :, :]
    
    # Compute gradients for monocular depth
    dx_m = mono_depth[:, 1:] - mono_depth[:, :-1]
    dy_m = mono_depth[1:, :] - mono_depth[:-1, :]
    
    # Calculate magnitudes
    mag_dx_d = torch.abs(dx_d)
    mag_dy_d = torch.abs(dy_d)
    
    mag_dx_i = torch.mean(torch.abs(dx_i), dim=2) # Average over channels
    mag_dy_i = torch.mean(torch.abs(dy_i), dim=2)
    
    mag_dx_m = torch.abs(dx_m)
    mag_dy_m = torch.abs(dy_m)
    
    # Calculate weights
    # e^(-|grad|)
    w_dx_i = torch.exp(-mag_dx_i)
    w_dy_i = torch.exp(-mag_dy_i)
    w_dx_m = torch.exp(-mag_dx_m)
    w_dy_m = torch.exp(-mag_dy_m)
    
    # Combine weights
    # Note: The formula sums the exponential terms
    weight_dx = w_dx_i + w_dx_m
    weight_dy = w_dy_i + w_dy_m
    
    # Compute loss terms
    term_dx = mag_dx_d * weight_dx
    term_dy = mag_dy_d * weight_dy
    
    # Sum over all pixels
    # The original formula divides by HW. The gradients are computed on (H-1)xW and Hx(W-1)
    # We sum all valid gradient pixels and divide by total pixels HW as per source
    total_loss = torch.sum(term_dx) + torch.sum(term_dy)
    
    loss = total_loss / (H * W)
    
    return loss
```
## Subtopics

### Gradient-Aware Depth Loss
A loss function that combines RGB image gradient weighting with L1 loss between rendered depth $\hat{D}$ and ground truth depth $D$ to regularize depth estimation [S2]. It is set apart by its use of ground truth depth (when available) and an L1 norm rather than smoothness priors on monocular estimates [S2]. Choose it when ground truth depth is available and texture complexity varies significantly [S2]. Formula: $\mathcal{L}_{\hat{D}} = g_{\text{rgb}} \frac{1}{|\hat{D}|} \sum \log(1+\|\hat{D}-D\|_1)$, where $g_{\text{rgb}} = \exp(-\nabla I)$ [S2].

## Operating characteristics
**Time and Memory Complexity:**
Linear in the number of pixels $HW$ for gradient computation and summation [S1, S2]. Memory overhead is minimal, requiring storage for gradient tensors [S1, S2].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Scales linearly with image resolution [S1, S2].

**Strengths and Limitations:**
*   **Strengths:** Self-supervised, does not require ground truth depth [S1]. Robust to edges via gradient weighting [S1, S2].
*   **Limitations:** Relies on the quality of monocular depth estimates [S1]. May smooth out fine details if not properly weighted [S1].

**When to Use:**
Use when ground truth depth is unavailable and monocular depth estimates are reliable [S1]. Use when texture complexity varies and gradient weighting is beneficial [S2].

**When Not to Use:**
Do not use when ground truth depth is available and a direct L1/L2 loss is preferred [S2]. Do not use if monocular depth estimates are highly inaccurate [S1].
## Background
Originated in NeRF-SDP (GRF, MVSNeRF, ENeRF, IBRNet) presented at MMAsia ’23 [S1]. Developed to efficiently generalize neural radiance fields with scene depth perception without ground truth [S1]. Also used in DN-Splatter for Gaussian Splatting and Meshing [S2].
## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S2] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
