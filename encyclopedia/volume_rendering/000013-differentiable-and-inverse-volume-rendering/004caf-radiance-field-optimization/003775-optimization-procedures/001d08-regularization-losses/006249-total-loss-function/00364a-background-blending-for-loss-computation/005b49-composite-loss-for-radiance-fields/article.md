# Composite Loss for Radiance Fields
Also called: Optimizing Radiance Fields for Few-Shot Novel-View Synthesis, Total Loss L

## Overview
The Composite Loss is the primary optimization objective used in NeO 360 to train dedicated near and far background MLPs for few-shot novel-view synthesis [S1]. It addresses the problem of accurately modeling outdoor backgrounds where the background is not black, by combining a pixel-wise L2 color loss, a distortion regularization term, and a perceptual LPIPS loss [S1]. It differs from standard volumetric alpha-compositing losses by explicitly enforcing sparsity in the background MLP weights to ensure efficient rendering and distinct background geometry, rather than merely blending colors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $c_p$: The composited background color for a pixel, obtained by volumetrically rendering and compositing outputs from the near background MLP ($D_{near}$) and far background MLP ($D_{far}$) [S1].
    *   $\tilde{c}_t$: The target pixel color sampled from the ground truth image at the corresponding pixel location [S1].
    *   $\lambda_{reg}$: Weight for the regularization loss.
    *   $\lambda_{LPIPS}$: Weight for the perceptual loss.
    *   $t$: Current training epoch.
*   **Outputs:**
    *   $L$: The scalar total loss value used for backpropagation [S1].

**Data Structures:**
*   **MLPs:** Two dedicated Multi-Layer Perceptrons, $D_{near}$ and $D_{far}$, decode color ($c_i^p$) and density ($\sigma_i^p$) for background segments [S1].
*   **Coordinates:** Input coordinates are contracted ($x', y', z'$) before conditioning the MLPs [S1].

**Formulas:**
The total loss $L$ is defined as [S1]:

$$
L = \left\| c_p - \tilde{c}_t \right\|_2^2 + \lambda_{reg} L_{reg} + \lambda_{LPIPS} L_{LPIPS}
$$

Where:
*   $\left\| c_p - \tilde{c}_t \right\|_2^2$ is the L2 color loss between the composited color and the target color [S1].
*   $L_{reg}$ is the distortion regularization loss [S1].
*   $L_{LPIPS}$ is the perceptual similarity loss between patches of rendered color $c_p$ and ground truth color $\tilde{c}_t$ [S1].

**Parameters:**
*   $\lambda_{reg}$: Not given in the sources.
*   $\lambda_{LPIPS}$: Not given in the sources.
*   LPIPS Activation Threshold: The LPIPS loss is only enforced after 30 training epochs [S1].

**Numbered Steps:**
1.  **Render Background:** For each ray, sample points in the near and far background regions. Use contracted coordinates $(x', y', z')$ to condition the dedicated near background MLP $D_{near}$ and far background MLP $D_{far}$ [S1].
2.  **Decode Properties:** Decode color $c_i^p$ and density $\sigma_i^p$ for each sample point from the respective MLPs [S1].
3.  **Composite Color:** Perform volumetric rendering and compositing of the near and far background segments to obtain the final composited background color $c_p$ for the pixel [S1].
4.  **Compute L2 Loss:** Calculate the squared L2 norm between the composited color $c_p$ and the target pixel color $\tilde{c}_t$ [S1].
5.  **Compute Regularization Loss:** Calculate the distortion regularization loss $L_{reg}$ [S1]. This encourages sparsity in the weights of the near and far background MLPs for efficient rendering [S1].
6.  **Compute Perceptual Loss:** Calculate the LPIPS loss $L_{LPIPS}$ between $c_p$ and $\tilde{c}_t$ [S1].
7.  **Apply Conditional Weighting:** If the current training epoch $t < 30$, set the weight for $L_{LPIPS}$ to 0 (or exclude the term) to improve background modeling stability [S1].
8.  **Aggregate Loss:** Compute the total loss $L$ using the formula above [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_composite_loss(c_p, c_t, epoch, lambda_reg, lambda_lpips):
    """
    Computes the composite loss for background radiance fields.
    
    Args:
        c_p: Composited background color tensor (H, W, 3).
        c_t: Target ground truth color tensor (H, W, 3).
        epoch: Current training epoch (int).
        lambda_reg: Weight for distortion regularization (float).
        lambda_lpips: Weight for LPIPS loss (float).
        
    Returns:
        L: Total scalar loss.
    """
    # 1. L2 Color Loss
    l2_loss = torch.sum((c_p - c_t) ** 2)
    
    # 2. Distortion Regularization Loss
    # Note: The specific implementation of L_reg depends on the distortion regularization method [3].
    # It is applied to the weights of the near and far background MLPs.
    l_reg = compute_distortion_regularization_loss()
    
    # 3. LPIPS Loss
    # Enforced only after 30 epochs to improve background modeling [S1]
    if epoch >= 30:
        l_lpips = compute_lpips_loss(c_p, c_t)
    else:
        l_lpips = torch.tensor(0.0)
        
    # 4. Total Loss
    L = l2_loss + lambda_reg * l_reg + lambda_lpips * l_lpips
    
    return L
```

**Edge Cases and Pitfalls:**
*   **Early Training Instability:** Applying LPIPS loss too early can hinder background modeling; it must be disabled for the first 30 epochs [S1].
*   **Background Color Assumption:** This method is specifically designed for cases where the background is not black, requiring explicit modeling via near/far MLPs rather than assuming a default background color [S1].
*   **Coordinate Contraction:** Input coordinates must be contracted before conditioning the MLPs to handle the unbounded nature of outdoor scenes [S1].

## Subtopics

### [Radiance Field Loss](22746-radiance-field-loss.md)
Treats surface candidates as binary existence choices rather than volumetric opacities, minimizing the weighted sum of color losses for the candidate and background surfaces to promote distinct surface geometry. Unlike standard volumetric alpha-compositing which blends colors and can result in blended, non-binary surfaces, this method blends losses, forcing the optimizer to select the single surface (candidate or background) that minimizes the color difference.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Encourages sparsity in background MLP weights for efficient rendering; improves perceptual similarity of rendered backgrounds; handles non-black backgrounds effectively [S1].
*   **Limitations:** Requires dedicated MLPs for near and far backgrounds; LPIPS loss adds computational overhead and is only active after 30 epochs [S1].
*   **When to Use:** Few-shot novel-view synthesis of outdoor scenes where background modeling is critical and source views are sparse [S1].
*   **When Not to Use:** When the background is known to be black or simple, or when computational resources are extremely limited and MLP sparsity is not required [S1].

## Background
The Composite Loss was introduced in the NeO 360 paper for optimizing radiance fields for few-shot novel-view synthesis of outdoor scenes [S1]. It builds on the need to accurately model complex outdoor backgrounds that cannot be assumed to be black, using dedicated MLPs for near and far background regions [S1]. The method incorporates distortion regularization [3] to encourage sparsity and LPIPS loss for perceptual quality, with a specific training schedule for the perceptual loss [S1].

## Sources
[S1] NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes, https://doi.org/10.1109/iccv51070.2023.00843, fetched 2026-10-26
