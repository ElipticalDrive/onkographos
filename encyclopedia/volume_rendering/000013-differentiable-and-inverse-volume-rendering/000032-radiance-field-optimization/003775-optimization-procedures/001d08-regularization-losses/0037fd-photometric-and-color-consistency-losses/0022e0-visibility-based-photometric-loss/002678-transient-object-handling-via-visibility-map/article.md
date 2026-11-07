# Transient Object Handling via Visibility Map

Also called: Transient Objects Handling, Visibility Map Loss for Transient Objects

## Overview
This method mitigates reconstruction artifacts caused by transient objects (e.g., moving pedestrians or cars) in unconstrained image collections by using a learned 2D visibility map to weight the photometric loss. It sets itself apart from static scene reconstruction by explicitly segmenting dynamic regions to prevent the model from attempting to reconstruct non-static geometry, which typically leads to floating artifacts or depth map inconsistencies. It builds on [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) by introducing a learnable mask rather than relying solely on geometric consistency or fixed thresholds.

## How it works
**Inputs and Outputs:**
- **Inputs:** Reference image $I_{gt}$, rendered image $I_r$, and a 2D visibility map $V_M$.
- **Outputs:** A weighted photometric loss value and an updated visibility map $V_M$.

**Data Structures:**
- **Visibility Map ($V_M$):** A 2D tensor of shape $R^{1 \times H \times W}$, where $H$ and $W$ are the height and width of the image. Each element $V_M(h, w) \in [0, 1]$ represents the visibility weight for pixel $(h, w)$. Values close to 1 indicate static, reconstructible regions; values close to 0 indicate transient or occluded regions.
- **Reference Image ($I_{gt}$):** The ground-truth input image.
- **Rendered Image ($I_r$):** The image rendered by the current model state.

**Numbered Steps:**
1. **Visibility Map Generation:** A U-Net model is employed to generate the 2D visibility map $V_M$. This model segments the image to distinguish between transient objects (moving people, cars) and static background elements [S1].
2. **Loss Weighting:** The photometric loss between the rendered image $I_r$ and the reference image $I_{gt}$ is computed using the Hadamard product ($\odot$) with the visibility map $V_M$. This ensures that regions identified as transient (low $V_M$) contribute less to the total loss, preventing the optimizer from forcing the static scene representation to fit dynamic artifacts [S1].
3. **Regularization:** A regularization loss $L_{vm}$ is applied to the visibility map to prevent the model from trivially setting all pixels to zero (invisible) to minimize the photometric loss. This forces the visibility map to remain meaningful [S1].

**Formulas:**
The weighted photometric loss is calculated as:
$$ L_{weighted} = \| (I_r - I_{gt}) \odot V_M \| $$
*(Note: The specific norm, typically L1 or L2, is inherited from the parent [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) method.)*

The regularization loss for the visibility map is defined as:
$$ L_{vm} = L_2(V_M, 1) $$
Where $L_2(V_M, 1)$ denotes the L2 loss between the visibility map $V_M$ and a tensor of ones of the same shape. This encourages $V_M$ to stay close to 1 for valid static regions [S1].

**Parameters:**
- **U-Net Architecture:** Used for generating $V_M$. Specific hyperparameters (layers, filters) are not detailed in the sources [S1].
- **Regularization Weight:** The weight for $L_{vm}$ relative to the photometric loss is not specified in the sources [S1].

**Edge Cases and Pitfalls:**
- **Trivial Solution:** Without the regularization term $L_{vm}$, the visibility map may converge to all zeros to minimize the photometric loss, effectively ignoring the entire image. The L2 penalty against ones prevents this [S1].
- **Dynamic Object Segmentation Accuracy:** The effectiveness relies on the U-Net's ability to accurately segment transient objects. Poor segmentation may lead to either artifacts (if transient regions are marked as static) or loss of detail (if static regions are incorrectly masked) [S1, S2].
- **Depth Map Gaps:** Transient objects can cause gaps or inconsistencies in depth maps. Excluding these regions via $V_M$ helps mitigate this, but the method assumes the static scene geometry is consistent across viewpoints [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transient_object_handling_loss(I_gt, I_r, V_M, reg_weight=1.0):
    """
    Computes the weighted photometric loss and regularization loss for transient object handling.
    
    Args:
        I_gt: Reference image tensor (H, W, C) or (1, H, W, C).
        I_r: Rendered image tensor (H, W, C) or (1, H, W, C).
        V_M: Visibility map tensor (1, H, W). Values in [0, 1].
        reg_weight: Weight for the visibility map regularization loss.
        
    Returns:
        total_loss: Combined loss value.
        L_vm: Regularization loss value.
    """
    # Ensure shapes are compatible for Hadamard product
    # V_M is (1, H, W), I_gt and I_r are likely (H, W, C) or (1, H, W, C)
    # Broadcasting V_M to match I_gt/I_r dimensions if necessary
    if V_M.shape != I_gt.shape:
        # Expand V_M to match channels if needed
        V_M_expanded = V_M.expand_as(I_gt)
    else:
        V_M_expanded = V_M
        
    # Compute weighted photometric error
    # Using L2 norm for the difference, weighted by visibility
    # Note: The source specifies L2 for regularization, but photometric norm 
    # is inherited from parent. We assume L2 for consistency with L_vm definition.
    weighted_diff = (I_r - I_gt) * V_M_expanded
    L_photometric = torch.mean(weighted_diff ** 2)
    
    # Compute regularization loss: L2 distance between V_M and ones
    ones_tensor = torch.ones_like(V_M)
    L_vm = torch.mean((V_M - ones_tensor) ** 2)
    
    # Combine losses
    total_loss = L_photometric + reg_weight * L_vm
    
    return total_loss, L_vm
```

## Subtopics

### Semantic Segmentation for Transient Objects
Uses a pre-trained or integrated semantic segmentation model to generate masks for moving objects (pedestrians, cars). This approach explicitly excludes these regions from scene training to mitigate depth map gaps. It is chosen when high-accuracy semantic masks are available or can be integrated into the pipeline, differing from the learnable U-Net approach by relying on external segmentation priors [S2].

### Learnable Visibility Map
Uses a U-Net to learn the visibility map end-to-end during training. This is chosen for unconstrained image collections where pre-defined semantic masks are not available, allowing the model to adaptively learn which regions are transient based on photometric consistency [S1].

## Operating characteristics
- **Time Complexity:** Adds the inference time of the U-Net model for generating $V_M$ per training step. The loss computation itself is $O(H \times W)$ due to the Hadamard product and L2 norms.
- **Memory Complexity:** Requires storing the visibility map $V_M$ of size $H \times W$ per image. The U-Net model weights are also loaded into memory.
- **Measured Performance:** The method outperforms previous methods (including 3DGS) on PhotoTourism scenes (Brandenburg, Gate, Sacre Coeur, Trevi, Fountain) in terms of PSNR, SSIM, and LPIPS [S1].
- **Strengths:** Effectively reduces artifacts from moving objects (floating points) and improves generalization by preventing the reconstruction of non-static geometry [S1, S2].
- **Limitations:** Relies on the accuracy of the visibility map generation (U-Net or semantic segmentation). If the visibility map is inaccurate, it may either retain artifacts or remove valid static details. The regularization weight is a hyperparameter that requires tuning [S1].
- **When to Use:** Suitable for unconstrained image collections with significant transient objects where standard NeRF/Gaussian Splatting methods produce artifacts.
- **When Not to Use:** Not necessary for static scenes with no moving objects, as it adds computational overhead without benefit.

## Background
The method originates from "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1], which addresses the challenge of reconstructing scenes from unstructured image sets containing dynamic elements. It builds upon the concept of visibility-based photometric loss, extending it to handle transience explicitly. An alternative approach using semantic segmentation for transient object handling is described in "UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene" [S2], highlighting the broader context of handling dynamic objects in radiance field optimization.

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S2] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
