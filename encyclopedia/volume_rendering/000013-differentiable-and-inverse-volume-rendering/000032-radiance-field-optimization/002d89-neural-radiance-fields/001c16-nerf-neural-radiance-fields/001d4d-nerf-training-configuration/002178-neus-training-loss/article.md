# NeuS Training Loss

Also called: NeUDF loss, Neural Implicit Surface training objective, Training Loss, optimization objective

## Overview
The NeuS training loss is a composite objective function used to optimize neural implicit surface representations (specifically Signed Distance Fields or Unsigned Distance Fields) from multi-view images without 3D ground-truth supervision. It solves the problem of recovering high-fidelity open surfaces by combining photometric consistency with geometric regularization. It distinguishes itself from standard NeRF training [NeRF Training Configuration](1203-nerf-training-configuration.md) by explicitly enforcing the Eikonal equation on the SDF/UDF gradients and using a mask-based binary cross-entropy loss to drive opacity toward binary values, thereby stabilizing the thin-surface approximation.

## How it works
The method optimizes a neural network that outputs a scalar field value (SDF or UDF) and an opacity (or density) value for any 3D point. The total loss $\mathcal{L}$ is a weighted sum of color, geometric, and mask losses.

### Inputs and Outputs
*   **Inputs**:
    *   Ground truth RGB images $I_{gt}$ for each camera pose.
    *   Ground truth binary foreground masks $M_{gt}$ (optional, but required for $\mathcal{L}_m$).
    *   Ray origins $o$ and directions $d$ sampled from the image plane.
    *   Sample points $x_i$ along the rays, sampled according to the volume rendering integration strategy (e.g., stratified sampling or adaptive sampling based on the SDF).
*   **Outputs**:
    *   Rendered RGB images $I_{pred}$.
    *   Rendered opacity/mask values $\alpha_{pred}$.
    *   Network gradients $\nabla S(x)$ at sample points.

### Data Structures
The implementation relies on tensors representing the batch of rays:
*   `rays_valid`: Boolean mask indicating which rays have valid ground truth data (non-background).
*   `comp_rgb_full`: Tensor of shape $(N_{rays}, 3)$ containing the accumulated RGB color for each ray.
*   `sdf_grad_samples`: Tensor of shape $(N_{samples}, 3)$ containing the gradient $\nabla SDF$ for each sample point.
*   `opacity`: Tensor of shape $(N_{rays})$ or $(N_{samples})$ depending on the specific accumulation step, representing the transmittance/opacity contribution.

### Formulas and Parameters
The total loss is defined as [S1]:

$$ \mathcal{L} = \mathcal{L}_c + \alpha \mathcal{L}_e + \beta \mathcal{L}_m $$

Where:
*   $\mathcal{L}_c$: Color loss (photometric consistency).
*   $\mathcal{L}_e$: Eikonal loss (geometric regularization).
*   $\mathcal{L}_m$: Mask loss (binary opacity enforcement).
*   $\alpha$: Weight for Eikonal loss.
*   $\beta$: Weight for Mask loss.

**1. Color Loss ($\mathcal{L}_c$)**
Measures the difference between rendered and ground truth images. Sources indicate the use of L1 loss [S1] or MSE [S2, unofficial]. The implementation in [S2, unofficial] uses MSE masked by valid rays.
$$ \mathcal{L}_c = \text{MSE}(I_{pred}[rays\_valid], I_{gt}[rays\_valid]) $$
*   Parameter: `lambda_rgb_mse` (weight for MSE component) [S2, unofficial].

**2. Eikonal Loss ($\mathcal{L}_e$)**
Enforces that the gradient of the SDF/UDF has a unit norm everywhere, which is a property of distance fields.
$$ \mathcal{L}_e = \text{mean}(\|\nabla SDF(x_i)\|_2 - 1)^2 $$
*   Parameter: `lambda_eikonal` (weight for Eikonal loss) [S2, unofficial].

**3. Mask Loss ($\mathcal{L}_m$)**
Encourages the predicted opacity to match the ground truth foreground mask. It uses Binary Cross Entropy (BCE). To prevent numerical instability ($\log(0)$), opacity values are clamped to $[10^{-3}, 1-10^{-3}]$ [S2, unofficial].
$$ \mathcal{L}_m = \text{BCE}(\text{clamp}(\alpha_{pred}, 10^{-3}, 1-10^{-3}), M_{gt}) $$
*   Parameter: `lambda_mask` (weight for mask loss) [S2, unofficial]. Applied only if masks are provided.

**4. Additional Regularization Terms (Optional/Advanced)**
Source [S2, unofficial] lists several additional terms often used in modern implementations (e.g., Instant-NSR-PL) to improve stability and quality:
*   **Opaque Loss**: Encourages opacity to be 0 or 1.
    $$ \mathcal{L}_{opaque} = \text{BCE}(\alpha_{pred}, \alpha_{pred}) $$
    *   Parameter: `lambda_opaque` [S2, unofficial].
*   **Sparsity Loss**: Encourages the SDF to be positive (empty space) outside the surface.
    $$ \mathcal{L}_{sparsity} = \text{mean}(\exp(-\lambda_{sparsity} \cdot |SDF|)) $$
    *   Parameter: `sparsity_scale` (used as $\lambda_{sparsity}$ in the exponent) [S2, unofficial].
*   **Curvature Loss**: Uses the absolute mean of SDF Laplace samples.
    *   Requirement: Geometry model must compute `sdf_laplace_samples` using finite differences (`geometry.grad_type = 'finite_difference'`) [S2, unofficial].
    *   Parameter: `lambda_curvature` [S2, unofficial].
*   **Distortion Loss**: Based on MipNeRF360, penalizing the distortion of the ray sampling intervals.
    *   Parameter: `lambda_distortion` [S2, unofficial].
*   **Background Distortion Loss**: Similar to distortion loss but for learned background components.
    *   Parameter: `lambda_distortion_bg` [S2, unofficial].

### Pseudocode
The following Python-like pseudocode implements the core loss calculation as described in [S2, unofficial], incorporating the optional terms.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_neus_loss(out, batch, config):
    """
    out: dict containing 'comp_rgb_full', 'rays_valid_full', 'sdf_grad_samples', 'opacity'
    batch: dict containing 'rgb', 'mask' (if available)
    config: object containing system.loss parameters
    """
    loss = 0.0
    
    # 1. Color Loss (MSE)
    # Mask valid rays
    valid_mask = out['rays_valid_full'][..., 0]
    pred_rgb = out['comp_rgb_full'][valid_mask]
    gt_rgb = batch['rgb'][valid_mask]
    
    loss_rgb_mse = F.mse_loss(pred_rgb, gt_rgb)
    loss += loss_rgb_mse * config.lambda_rgb_mse
    
    # 2. Eikonal Loss
    # Gradient norm should be 1
    grad_norm = torch.linalg.norm(out['sdf_grad_samples'], ord=2, dim=-1)
    loss_eikonal = ((grad_norm - 1.0)**2).mean()
    loss += loss_eikonal * config.lambda_eikonal
    
    # 3. Mask Loss (BCE)
    if config.has_mask:
        # Clamp opacity to avoid log(0)
        opacity_clamped = torch.clamp(out['opacity'], 1e-3, 1 - 1e-3)
        gt_mask = batch['mask'][valid_mask] # Assuming mask is aligned with valid rays
        loss_mask = F.binary_cross_entropy(opacity_clamped, gt_mask)
        loss += loss_mask * config.lambda_mask
        
        # 4. Opaque Loss (Optional)
        # Encourages opacity to be 0 or 1
        loss_opaque = F.binary_cross_entropy(out['opacity'], out['opacity'])
        loss += loss_opaque * config.lambda_opaque
        
        # 5. Sparsity Loss (Optional)
        # Encourages SDF to be positive (empty)
        # Note: SDF values must be available in 'out'
        sdf_values = out['sdf_samples'] # Shape: (N_samples,)
        sparsity_scale = config.sparsity_scale
        loss_sparsity = torch.exp(-sparsity_scale * torch.abs(sdf_values)).mean()
        loss += loss_sparsity * config.lambda_sparsity
        
        # 6. Curvature Loss (Optional)
        # Requires geometry.grad_type == 'finite_difference'
        if 'sdf_laplace_samples' in out:
            loss_curvature = torch.abs(out['sdf_laplace_samples']).mean()
            loss += loss_curvature * config.lambda_curvature
            
        # 7. Distortion Loss (Optional)
        # Requires weights, points, intervals, ray_indices from volume rendering step
        if config.lambda_distortion > 0:
            distortion_loss = compute_distortion_loss(
                weights=out['weights'],
                points=out['points'],
                intervals=out['intervals'],
                ray_indices=out['ray_indices']
            )
            loss += distortion_loss * config.lambda_distortion
            
        if config.lambda_distortion_bg > 0:
            bg_distortion_loss = compute_distortion_loss(
                weights=out['bg_weights'],
                points=out['bg_points'],
                intervals=out['bg_intervals'],
                ray_indices=out['ray_indices']
            )
            loss += bg_distortion_loss * config.lambda_distortion_bg

    return loss
```

### Edge Cases and Pitfalls
*   **Numerical Stability**: Opacity values must be clamped away from 0 and 1 before computing Binary Cross Entropy to avoid $\log(0)$ instability [S2, unofficial].
*   **Curvature Loss Prerequisite**: The curvature loss requires the geometry model to explicitly compute SDF Laplace samples. The code must assert that `geometry.grad_type` is set to `'finite_difference'` to ensure `out['sdf_laplace_samples']` is populated [S2, unofficial].
*   **Mask Availability**: The mask loss $\mathcal{L}_m$ is only applied if foreground masks are provided in the dataset. If not, $\beta$ is effectively 0 or the term is skipped [S2, unofficial].
*   **Dynamic Ray Sampling**: Some implementations use a dynamic update rule for the number of rays $N_{rays}$ during training to balance computation and quality:
    $$ N_{rays}^{new} = \min\left(0.9 N_{rays}^{old} + 0.1 \left\lfloor N_{rays}^{old} \cdot \frac{N_{samples}}{\sum N_{samples}^{pred}} \right\rfloor, N_{rays}^{max}\right) $$
    [S2, unofficial].

## Subtopics

### [PG-NeuS](4251-pg-neus.md)
PG-NeuS is a point-guided neural implicit surface reconstruction method that enhances the base NeuS framework by modeling aleatoric uncertainty to capture point cloud noise distribution, using a Neural Projection module for geometric constraints, and employing a Bias Network to compensate for geometric bias between volume rendering and point modeling. It distinguishes itself from siblings like RegSDF and TUVR by estimating point reliability during geometry optimization via uncertainty modeling rather than using uniform smoothing or external MVS confidence maps, enabling robust handling of noisy and sparse point data.

## Operating characteristics
*   **Time Complexity**: Dominated by the volume rendering integration step and the computation of gradients for the Eikonal loss. The Eikonal loss requires computing $\nabla SDF$ for all sample points, which is typically done via automatic differentiation, adding a backward pass cost proportional to the number of samples.
*   **Memory Complexity**: Requires storing gradients for all sample points for the Eikonal loss backpropagation. This can be significant for high-resolution scenes with many samples per ray.
*   **Strengths**:
    *   Produces high-quality, watertight surfaces without explicit 3D supervision.
    *   The Eikonal loss stabilizes the SDF learning, preventing "fuzzy" surfaces.
    *   The mask loss helps in separating foreground from background effectively.
*   **Limitations**:
    *   Sensitive to the choice of loss weights ($\alpha, \beta$).
    *   Requires careful handling of numerical stability (clamping).
    *   May struggle with highly transparent or thin structures if the sampling strategy is not adaptive.
*   **When to use**: Ideal for reconstructing detailed, open surfaces from multi-view images where 3D ground truth is unavailable.
*   **When not to use**: If 3D ground truth (e.g., LiDAR, structured light) is available, direct SDF regression might be more efficient. If real-time inference is critical, the gradient computation overhead might be prohibitive.

## Background
The NeuS training loss was introduced in the context of Neural Implicit Surface (NeuS) reconstruction, building upon the foundations of Neural Radiance Fields (NeRF) [NeRF](1200-nerf.md). The original NeuS paper [53] (cited in [S1]) proposed using volume rendering with a signed distance field to achieve high-fidelity surface reconstruction. The loss function combines photometric consistency (color loss) with geometric regularization (Eikonal loss) and mask supervision. The NeUDF variant [S1] extends this to Unsigned Distance Fields, maintaining the same loss structure but adapting the SDF properties. Modern implementations like Instant-NSR-PL [S2, unofficial] have expanded the loss with additional terms like distortion and sparsity losses to improve robustness and training stability.

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
