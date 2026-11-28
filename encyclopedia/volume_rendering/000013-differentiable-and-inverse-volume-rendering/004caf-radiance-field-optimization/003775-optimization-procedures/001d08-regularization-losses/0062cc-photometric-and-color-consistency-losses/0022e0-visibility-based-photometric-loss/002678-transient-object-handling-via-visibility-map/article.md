# Transient Object Handling via Visibility Map

Also called: Semantic Segmentation Masking, Transient Object Masking, Transient Objects Handling, Visibility Map Loss for Transient Objects

## Overview
This method mitigates artifacts caused by transient objects (e.g., moving people, cars) in static scene reconstruction by using a 2D visibility map to weight the photometric loss [S1]. It prevents the model from attempting to reconstruct transient geometry, which typically leads to floating artifacts or depth map inconsistencies [S1, S2]. Unlike methods that explicitly model transient appearance in 3D space, this approach operates by suppressing the influence of transient regions in the 2D image plane during optimization [S1]. It builds on [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) by introducing a learnable or pre-computed mask to dynamically exclude unreliable pixels [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $I_r$, ground truth reference image $I_{gt}$, and a 2D visibility map $V_M$ [S1].
*   **Outputs:** A weighted photometric loss value and an updated visibility map (if learned) [S1].

**Data Structures:**
*   $V_M \in \mathbb{R}^{1 \times H \times W}$: A single-channel float tensor where each element represents the visibility weight for the corresponding pixel [S1]. Values typically range from 0 (transient/occluded) to 1 (static/visible) [G].

**Steps:**
1.  **Generate Visibility Map:** Obtain $V_M$ from a semantic segmentation model (e.g., a U-Net) or a dedicated transient detection network [S1, S2]. The model identifies regions occupied by moving objects (pedestrians, cars) [S2].
2.  **Compute Weighted Loss:** Calculate the photometric loss between $I_r$ and $I_{gt}$ using the Hadamard product ($\odot$) with $V_M$ [S1].
3.  **Apply Regularization:** Compute a regularization loss $L_{vm}$ to prevent the visibility map from collapsing to zero (marking all pixels as transient) [S1].
4.  **Optimize:** Update network parameters using the combined loss [G].

**Formulas:**
The weighted photometric loss is computed as:
$$ L_{photometric} = \frac{1}{HW} \sum_{h,w} V_M(h,w) \cdot \|I_r(h,w) - I_{gt}(h,w)\|^2 $$
*(Note: The specific norm, typically L2, is inherited from the parent photometric loss definition [G].)*

The regularization loss $L_{vm}$ ensures the visibility map does not suppress all pixels:
$$ L_{vm} = L_2(V_M, 1) $$
where $L_2(V_M, 1)$ denotes the L2 loss between the visibility map $V_M$ and a tensor of ones of the same shape [S1].

**Parameters:**
*   Regularization weight for $L_{vm}$: Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Collapse to Zero:** Without regularization, the model may learn to set $V_M$ to 0 for all pixels to minimize the photometric error caused by transient objects, resulting in a black or empty scene reconstruction [S1].
*   **Precision:** The visibility map must be differentiable if it is learned end-to-end; otherwise, it must be pre-computed and treated as a constant mask during the photometric loss calculation [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transient_visibility_loss(I_r, I_gt, V_M, reg_weight=1.0):
    """
    Computes the transient-aware photometric loss.
    
    Args:
        I_r: Rendered image tensor (H, W, 3).
        I_gt: Ground truth image tensor (H, W, 3).
        V_M: Visibility map tensor (1, H, W) or (H, W).
        reg_weight: Weight for the visibility map regularization loss.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # Expand V_M to (H, W) if it has a batch/channel dimension
    if V_M.ndim == 3 and V_M.shape[0] == 1:
        V_M = V_M.squeeze(0)
    # Broadcast V_M to (H, W, 1) for element-wise multiplication with RGB images
    V_M_expanded = V_M[:, :, None]
    # Compute pixel-wise squared difference
    diff_sq = (I_r - I_gt) ** 2
    # Weighted photometric loss (mean over H, W, C dimensions)
    photometric_loss = mean(diff_sq * V_M_expanded)
    # Regularization loss: L2 distance between V_M and ones
    ones_tensor = ones_like(V_M)
    reg_loss = mean((V_M - ones_tensor) ** 2)
    # Combined loss
    total_loss = photometric_loss + reg_weight * reg_loss
    return total_loss
```

## Subtopics

### Isolated Transient Network
A separate network $T_\phi$ operating at the 2D image level to model transient objects (occluders) and their opacity, preventing noisy gradients from flowing into pose parameters [S3, unofficial]. It maps deep features $F_i(r)$ to transient opacity $\alpha^{(\tau)}$ and latent code $z^{(\tau)}$ using a sub-network $T_{\phi1}$ [S3, unofficial]. Another sub-network $T_{\phi2}$ encodes transient color $\hat{C}_i^{(\tau)}$ and uncertainty $\tilde{\beta}_i$ using the latent code and transient embedding $\ell_i^{(\tau)}$ [S3, unofficial]. The final color is alpha-blended: $\hat{C}_i(r) = (1 - \alpha^{(\tau)}(r))\hat{C}_i^{(a)}(r) + \alpha^{(\tau)}(r)\hat{C}_i^{(\tau)}(r)$ [S3, unofficial]. It uses parameters $\beta_{min}=0.1$ (minimum uncertainty) and $\lambda_\alpha=1.0$ (weight to suppress transient opacity) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds the inference time of the segmentation model (U-Net) or transient network per frame if computed dynamically [S1, S2][S3, unofficial]. If pre-computed, the overhead is negligible during training [S1, S2][S3, unofficial].
*   **Memory Complexity:** Requires storing the visibility map $V_M$ ($H \times W$ floats) or the parameters of the transient network [S1][S3, unofficial].
*   **Strengths:** Effectively removes artifacts from moving objects without requiring explicit 3D modeling of dynamics [S1, S2]. Improves depth map consistency by excluding inconsistent regions [S2].
*   **Limitations:** Relies on the accuracy of the segmentation/transient detection model [S1, S2]. If the model fails to detect a transient object, artifacts may persist [S1, S2]. The regularization term prevents the visibility map from collapsing to zero, which would discard all pixels [S1].
*   **When to use:** Unconstrained image collections with significant transient content (e.g., street scenes with pedestrians) [S1, S2].

## Background
The concept of handling transient objects in NeRF and 3D Gaussian Splatting emerged to address the limitation of these methods in assuming static scenes [S1, S2][S3, unofficial]. Early works like NeRF-W introduced uncertainty modeling to handle transients [S3, unofficial]. Subsequent works integrated semantic segmentation masks to explicitly exclude moving objects [S2]. The visibility map approach, as detailed in "Gaussian in the Wild," applies this concept to 3D Gaussian Splatting for unconstrained image collections, using a U-Net to generate the map [S1]. The Isolated Transient Network variant was proposed in UP-NeRF to decouple transient modeling from pose optimization [S3, unofficial].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S2] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
[S3] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
