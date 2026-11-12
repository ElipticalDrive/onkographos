# Patch-Based Depth Regularization

Also called: Bilateral Depth Regularization, Bilateral filter regularization, Joint Bilateral Depth Regularization, Joint bilateral filter regularization

## Overview
Patch-Based Depth Regularization is a technique applied during Neural Radiance Field (NeRF) training to enforce geometric consistency by smoothing rendered depth maps while preserving sharp structural edges [S1, unofficial]. It distinguishes itself from sibling methods like RegNeRF’s patch similarity constraints by utilizing bilateral or joint bilateral filters, which explicitly account for spatial proximity and intensity/depth similarity to reduce noise without blurring boundaries [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of rendered depth patches $P$ extracted from the NeRF’s depth output $\hat{D}$. Each patch $p$ has a defined size (e.g., $16 \times 16$ pixels).
*   **Output:** A scalar regularization loss $L_{reg}$ added to the total training loss.

**Data Structures**
*   **Depth Patch $\hat{D}(p)$:** A 2D tensor of shape $(H_p, W_p)$ containing depth values for a specific patch $p$.
*   **Set $P$:** The collection of all patches sampled from the rendered depth map during a training step.
*   **Filter Kernel:** Defined by spatial sigma $\sigma_{space}$ and range sigma $\sigma_{color}$.

**Parameters**
*   **Patch Size:** $16 \times 16$ pixels [S1, unofficial]. Larger sizes (32, 64) cause undertraining and excessive compute time [S1, unofficial].
*   **Kernel Size:** $9 \times 9$ [S1, unofficial].
*   **Spatial Sigma ($\sigma_{space}$):** $75 \times 75$ (defines spatial extent) [S1, unofficial].
*   **Range Sigma ($\sigma_{color}$):** $10$ (controls intensity/depth similarity) [S1, unofficial].
*   **Weight ($\lambda_{reg}$):** $10^{-7}$ [S1, unofficial].
*   **Color Weight ($\lambda_{color}$):** $1$ (used in joint bilateral filter) [S1, unofficial].

**Algorithm Steps**
1.  **Pre-training:** Train the NeRF model to convergence using only photometric (RGB) loss. This establishes a baseline geometry before introducing regularization [S1, unofficial].
2.  **Patch Sampling:** Extract patches $p \in P$ from the rendered depth map $\hat{D}$.
3.  **Filtering:** Apply a bilateral or joint bilateral filter $F$ to each patch $\hat{D}(p)$ to produce a filtered patch $F(\hat{D}(p))$ [S1, unofficial].
    *   *Bilateral Filter:* Weights pixels based on spatial distance and depth difference [S1, unofficial].
    *   *Joint Bilateral Filter:* Weights pixels based on spatial distance, depth difference, and RGB intensity similarity in the corresponding view [S1, unofficial].
4.  **Loss Calculation:** Compute the Mean Squared Error (MSE) between the original depth patch and the filtered patch for all pixels in the patch, then average over all patches [S1, unofficial].

**Formula**
The regularization loss is defined as:

$$
L_{reg} = \frac{1}{|P|} \sum_{p \in P} \frac{1}{|p|} \sum_{i,j} \| \hat{D}(p_{ij}) - F(\hat{D}(p))_{ij} \|^2
$$

Where:
*   $|P|$ is the number of patches.
*   $|p|$ is the number of pixels in patch $p$.
*   $\hat{D}(p_{ij})$ is the original depth value at pixel $(i,j)$ in patch $p$.
*   $F(\hat{D}(p))_{ij}$ is the filtered depth value at pixel $(i,j)$ in patch $p$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def patch_based_depth_regularization(depth_map, patches, filter_func, lambda_reg):
    """
    Computes the patch-based depth regularization loss.
    
    Args:
        depth_map (Tensor): Rendered depth map of shape (H, W).
        patches (List[Tensor]): List of depth patches extracted from depth_map.
        filter_func (Callable): Function applying bilateral or joint bilateral filter.
        lambda_reg (float): Weight for the regularization loss.
        
    Returns:
        Tensor: Scalar regularization loss.
    """
    total_loss = 0.0
    num_patches = len(patches)
    
    for p in patches:
        # p is a patch tensor, e.g., shape (16, 16)
        # Apply filter to the patch
        filtered_p = filter_func(p)
        
        # Compute MSE between original and filtered patch
        # Note: Using mean squared error over all pixels in the patch
        mse = torch.mean((p - filtered_p) ** 2)
        
        total_loss += mse
        
    # Average over all patches
    avg_loss = total_loss / num_patches
    
    # Scale by regularization weight
    return avg_loss * lambda_reg

def bilateral_filter(patch, sigma_space=75, sigma_color=10):
    """
    Simplified bilateral filter implementation for a single patch.
    Uses a 9x9 kernel.
    """
    # Implementation details depend on library (e.g., Kornia)
    # This is a placeholder for the actual filtering logic
    # which involves calculating spatial and range weights
    # for each pixel in the 9x9 neighborhood.
    pass

def joint_bilateral_filter(patch, rgb_patch, sigma_space=75, sigma_color=10, sigma_intensity=10):
    """
    Joint bilateral filter using RGB intensity for range weighting.
    """
    pass
```

**Edge Cases and Pitfalls**
*   **Training Time:** Adding patch-based regularization significantly increases training time. Models using this method converged in 280,000 iterations, compared to 200,000 for photometric-only training [S1, unofficial].
*   **Patch Size Sensitivity:** Larger patches (32x32, 64x64) remain undertrained even after 400,000 iterations and drastically increase compute cost [S1, unofficial].
*   **Occlusion:** Performance may degrade in scenes with significant occlusions (e.g., large furniture) where ray coverage is limited [S1, unofficial].
*   **Implementation:** The method relies on open-source libraries like Kornia for filter implementation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly higher than standard NeRF training due to the additional filtering step per patch and the need for longer training durations (280,000 iterations vs 200,000) [S1, unofficial].
*   **Memory Complexity:** Increases with the number of patches and patch size. Larger patches require more memory for intermediate filter computations [S1, unofficial].
*   **Performance:** Outperforms RegNeRF’s depth patch regularization in indoor scenes by reducing noise while preserving sharp edges [S1, unofficial]. Joint bilateral filtering provides additional gains over standard bilateral filtering [S1, unofficial].
*   **Strengths:** Effective at smoothing noise in architectural surfaces (floors, walls) while maintaining structural integrity [S1, unofficial].
*   **Limitations:** Computationally expensive; sensitive to patch size; less effective in highly occluded areas [S1, unofficial].
*   **When to Use:** Indoor scenes with planar architectural surfaces where geometric consistency is critical and computational resources allow for extended training [S1, unofficial].
*   **When Not to Use:** Real-time applications, outdoor scenes with complex natural geometry, or when training time is constrained [S1, unofficial].

## Background
The method was proposed in "Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios" (2025) [S1, unofficial]. It builds on the concept of depth regularization introduced in RegNeRF (Niemeyer et al., 2022) but replaces simple patch similarity constraints with bilateral and joint bilateral filters [S1, unofficial]. The bilateral filter was originally introduced by Tomasi and Manduchi (1998), and the joint bilateral filter by He et al. (2012) [S1, unofficial]. The implementation was tested using Instant-NGP within the Nerfstudio framework [S1, unofficial].

## Sources
[S1] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27 (unofficial)
