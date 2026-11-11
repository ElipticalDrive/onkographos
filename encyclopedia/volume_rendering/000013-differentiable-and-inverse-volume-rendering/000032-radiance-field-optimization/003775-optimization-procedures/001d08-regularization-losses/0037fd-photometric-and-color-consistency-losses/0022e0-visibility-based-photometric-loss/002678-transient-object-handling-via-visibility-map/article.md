# Transient Object Handling via Visibility Map

Also called: Semantic Segmentation Masking, Transient Object Masking, Transient Objects Handling, Visibility Map Loss for Transient Objects

## Overview
This method mitigates reconstruction artifacts from transient objects (e.g., moving pedestrians or cars) by using a 2D visibility map to weight the photometric loss, preventing the model from reconstructing transient geometry [S1]. It sets itself apart from standard static scene reconstruction by explicitly segmenting dynamic regions to avoid floating artifacts or depth map inconsistencies [S1, S2]. It builds on [Visibility-Based Photometric Loss](2917-visibility-based-photometric-loss.md) by introducing an explicit visibility mask rather than relying solely on geometric consistency or fixed thresholds.

## How it works
**Inputs and Outputs:**
- **Inputs:** Reference image $I_{gt}$, rendered image $I_r$, and a 2D visibility map $V_M$ [S1, S2].
- **Outputs:** A weighted photometric loss value and an updated visibility map $V_M$ [S1].

**Data Structures:**
- **Visibility Map ($V_M$):** A 2D tensor of shape $\mathbb{R}^{1 \times H \times W}$, where $H$ and $W$ are the image height and width [S1]. Each element $V_M(h, w) \in [0, 1]$ represents the visibility weight for pixel $(h, w)$ [G]. Values close to 1 indicate static, reconstructible regions; values close to 0 indicate transient or occluded regions [G].
- **Reference Image ($I_{gt}$):** The ground-truth input image [S1].
- **Rendered Image ($I_r$):** The image rendered by the current model state [S1].

**Numbered Steps:**
1. **Visibility Map Generation:** A U-Net model is employed to generate the 2D visibility map $V_M$, segmenting transient objects (moving people, cars) from static background elements [S1]. Alternatively, an integrated semantic segmentation model can generate masks for moving objects [S2].
2. **Loss Weighting:** The photometric loss calculation between $I_r$ and $I_{gt}$ is weighted using the visibility map $V_M$ via Hadamard product ($\odot$) [S1]. Regions with lower visibility contribute less to the total loss, preventing the optimizer from forcing the static scene representation to fit dynamic artifacts [S1].
3. **Regularization:** A regularization loss $L_{vm}$ is applied to $V_M$ to prevent the model from trivially setting all pixels to zero (invisible) to minimize the photometric loss [S1].

**Formulas:**
The weighted photometric loss applies the visibility map via Hadamard product to the pixel-wise difference before aggregation [S1]:
$$ L_{\text{weighted}} = \text{Loss}\left((I_r - I_{gt}) \odot V_M\right) $$
where $\text{Loss}$ denotes the standard photometric norm (e.g., L1 or L2) aggregated over pixels [G].

The regularization loss for the visibility map is defined as [S1]:
$$ L_{vm} = L_2(V_M, \mathbf{1}) $$
where $L_2(V_M, \mathbf{1})$ denotes the L2 loss between the visibility map $V_M$ and a tensor of ones of the same shape [S1]. This encourages $V_M$ to stay close to 1 for valid static regions [S1].

**Parameters:**
- **U-Net Architecture:** Used for generating $V_M$. Specific hyperparameters (layers, filters) are not detailed in the sources [S1].
- **Regularization Weight:** The weight for $L_{vm}$ relative to the photometric loss is not specified in the sources [S1].

**Edge Cases and Pitfalls:**
- **Trivial Solution:** Without $L_{vm}$, $V_M$ may converge to all zeros to minimize the photometric loss, effectively ignoring the entire image. The L2 penalty against ones prevents this [S1].
- **Segmentation Accuracy:** The effectiveness relies on the U-Net's or segmentation model's ability to accurately segment transient objects. Poor segmentation may lead to artifacts (if transient regions are marked as static) or loss of detail (if static regions are incorrectly masked) [S1, S2].
- **Depth Map Gaps:** Transient objects can cause gaps or inconsistencies in depth maps. Excluding these regions via $V_M$ mitigates this impact [S2].

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
    # Broadcast V_M to match I_gt/I_r dimensions for Hadamard product
    if len(V_M.shape) < len(I_gt.shape):
        V_M_expanded = V_M.repeat(1, 1, I_gt.shape[-1])
    else:
        V_M_expanded = V_M
        
    # Compute weighted photometric error (L2 norm over pixels)
    weighted_diff = (I_r - I_gt) * V_M_expanded
    L_photometric = mean(weighted_diff ** 2)
    
    # Compute regularization loss: L2 distance between V_M and ones
    ones_tensor = ones_like(V_M)
    L_vm = mean((V_M - ones_tensor) ** 2)
    
    # Combine losses
    total_loss = L_photometric + reg_weight * L_vm
    
    return total_loss, L_vm

def mean(tensor):
    return sum(val for row in tensor for val in row.flatten()) / len(tensor.flatten())

def ones_like(tensor):
    return [[1.0 for _ in range(tensor.shape[1])] for _ in range(tensor.shape[0])]
```

## Subtopics

### Semantic Segmentation for Transient Objects
Uses an integrated semantic segmentation model to generate masks for moving objects (pedestrians, cars) [S2]. This approach explicitly excludes these regions from scene training to mitigate depth map gaps [S2]. It is chosen when external segmentation priors are available, differing from the learnable U-Net approach by relying on pre-computed masks [S2].

### Learnable Visibility Map
Uses a U-Net to learn the visibility map end-to-end during training [S1]. This is chosen for unconstrained collections where pre-defined masks are unavailable, allowing the model to adaptively learn which regions are transient based on photometric consistency [S1].

## Operating characteristics
- **Time Complexity:** Adds the inference time of the U-Net or segmentation model for generating $V_M$ per training step. The loss computation itself is $O(H \times W)$ due to the Hadamard product and L2 norms [S1].
- **Memory Complexity:** Requires storing the visibility map $V_M$ of size $H \times W$ per image. The U-Net or segmentation model weights are also loaded into memory [S1].
- **Measured Performance:** The method outperforms previous methods (including 3DGS) on PhotoTourism scenes (Brandenburg, Gate, Sacre Coeur, Trevi, Fountain) in terms of PSNR, SSIM, and LPIPS [S1].
- **Strengths:** Effectively reduces artifacts from moving objects (floating points) and improves generalization by preventing the reconstruction of non-static geometry [S1, S2].
- **Limitations:** Relies on the accuracy of the visibility map generation (U-Net or semantic segmentation). If the visibility map is inaccurate, it may either retain artifacts or remove valid static details [S1, S2]. The regularization weight is not specified in the sources [S1].
- **When to Use:** Suitable for unconstrained image collections with significant transient objects where standard NeRF/Gaussian Splatting methods produce artifacts [S1].
- **When Not to Use:** Not necessary for static scenes with no moving objects, as it adds computational overhead without benefit [G].

## Background
The method originates from "Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections" [S1], which addresses the challenge of reconstructing scenes from unstructured image sets containing dynamic elements. It builds upon the concept of visibility-based photometric loss, extending it to handle transience explicitly [S1]. An alternative approach using semantic segmentation for transient object handling is described in "UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene" [S2], highlighting the broader context of handling dynamic objects in radiance field optimization [S2].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S2] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
