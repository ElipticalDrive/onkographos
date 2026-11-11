# Overall Training Loss

Also called: Composite Training Loss, End-to-End Training Objective, Multi-term Loss Function, PDRF Training Loss

## Overview
The Overall Training Loss is the composite objective function used to optimize differentiable volume rendering and radiance field models end-to-end. It aggregates multiple loss terms—such as photometric reconstruction, geometric consistency, and regularization constraints—to guide the learning of scene representations. It differs from sibling losses by serving as the final summation of configured components, where specific implementations vary based on the underlying representation (e.g., Gaussian Splatting vs. Neural Radiance Fields) and the training stage.

## How it works
The method defines the total loss $L_{tot}$ as a weighted sum of specific loss components. The exact composition depends on the specific radiance field architecture and training protocol.

### Inputs and Outputs
*   **Inputs:** Rendered images $\hat{I}$, ground truth images $I$, predicted depth maps $\hat{D}$, ground truth depth (if available), Gaussian attributes (position $p$, opacity $\tilde{o}$, rotation $\tilde{r}$, scale $\tilde{s}$, color $c$), and camera parameters.
*   **Outputs:** A scalar loss value $L_{tot}$ used for backpropagation.

### Data Structures
The loss calculation relies on the following data structures defined in the rendering pipeline:
*   **Rendered Image $\hat{I}$:** 2D tensor of shape $(H, W, 3)$ representing the RGB output of the differentiable renderer.
*   **Ground Truth Image $I$:** 2D tensor of shape $(H, W, 3)$, often gamma-corrected.
*   **Predicted Depth $\hat{D}$:** 2D tensor of shape $(H, W)$ representing the expected depth along each ray.
*   **Gaussian Attributes:**
    *   $p_n \in \mathbb{R}^3$: Position.
    *   $\tilde{o}_n \in \mathbb{R}$: Opacity.
    *   $\tilde{r}_n \in \mathbb{R}^4$: Rotation (quaternion).
    *   $\tilde{s}_n \in \mathbb{R}^3$: Scale.
    *   $c_n(\cdot)$: View-dependent color (e.g., spherical harmonics coefficients or neural field output).

### Formulas and Steps
The specific formulation varies by method. Below are the primary definitions found in the sources.

#### 1. PDRF (Progressively Deblurring Radiance Field) [S1]
Used for deblurring scene reconstruction.
$$ L_{tot} = L_{CRR\_photo} + L_{FVR\_photo} + \lambda (L_{TV\_coar} + L_{TV\_fine}) $$
*   $L_{CRR\_photo}$: Photometric loss for the Coarse Radiance Renderer (CRR).
*   $L_{FVR\_photo}$: Photometric loss for the Fine Voxel Renderer (FVR).
*   $L_{TV\_coar}, L_{TV\_fine}$: Total Variation constraints on the coarse and fine explicit feature volumes to enforce smoothness.
*   $\lambda$: Weight for TV losses.

#### 2. TD-NeRF (Truncated Depth Prior) [S2]
Used for joint camera pose and NeRF optimization.
$$ L = L_{rgb} + \lambda_1 L_{self-depth} + \lambda_2 L_{GPC} + \lambda_3 L_{reproj} $$
*   $L_{rgb}$: RGB reconstruction loss.
*   $L_{self-depth}$: Self-supervised depth loss.
*   $L_{GPC}$: Gaussian Point Constraint loss.
*   $L_{reproj}$: Reprojection loss.
*   Weights: $\lambda_1 = 0.04$, $\lambda_2 = 1.0$, $\lambda_3 = 1.0$.

#### 3. Compact 3D Gaussian Representation [S3]
Used for compact Gaussian splatting with codebooks.
$$ L = L_{ren} + \lambda_m L_m + L_r + L_s $$
*   $L_{ren}$: Rendering loss (weighted sum of L1 and SSIM between ground truth and rendered images).
*   $L_m$: Masking loss.
*   $L_r$: Rotation codebook loss.
*   $L_s$: Scale codebook loss.
*   $\lambda_m$: Hyperparameter to regularize the number of Gaussians.
*   **Codebook Losses:**
    $$ L_r = \sum_{n=1}^N \sum_{k=1}^K \| \text{sg}[\hat{r}_n - \hat{r}_n^{k-1}] - Z_k[i_n^k] \|_2^2 $$
    $$ L_s = \sum_{n=1}^N \sum_{k=1}^K \| \text{sg}[\hat{s}_n - \hat{s}_n^{k-1}] - Z_k[i_n^k] \|_2^2 $$
    Where $\text{sg}[\cdot]$ is the stop-gradient operator, $Z_k$ is the codebook vector, and $i_n^k$ is the index. $L_r$ and $L_s$ are applied only for the last 1K training iterations [S3].

#### 4. ReSplat (Recurrent Gaussian Splatting) [S4]
Uses a two-stage training process.

**Stage 1: Initial Reconstruction**
$$ L_{1st} = \sum_{v=1}^V \ell_{render}(\hat{I}_v, I_v) + \alpha \cdot \sum_{i=1}^N \ell_{smooth}(I_i, \hat{D}_i) $$
*   $\ell_{render}(\hat{I}, I) = \ell_1(\hat{I}, I) + \lambda \cdot \ell_{percept}(\hat{I}, I)$
*   $\ell_{smooth}(I, \hat{D}) = |\partial_x \hat{D}| e^{-|\partial_x I|} + |\partial_y \hat{D}| e^{-|\partial_y I|}$
*   $\alpha = 0.01$, $\lambda = 0.5$.

**Stage 2: Recurrent Model Training**
The initial model is frozen. The recurrent model is trained end-to-end.
$$ L_{2nd} = \sum_{t=0}^{T-1} \gamma^{T-1-t} \sum_{v=1}^V \ell_{render}(\hat{I}_v^t, I_v) $$
*   $\gamma = 0.9$: Exponential weight decay factor.
*   $T$: Number of recurrent iterations.

### Parameters
*   **Batch Size:** 1024 rays [S1].
*   **Gamma Correction:** 2.2 used as camera response function [S1].
*   **Loss Weights:**
    *   PDRF: $\lambda$ for TV (value not specified in sources).
    *   TD-NeRF: $\lambda_1=0.04, \lambda_2=1.0, \lambda_3=1.0$ [S2].
    *   ReSplat: $\alpha=0.01, \lambda=0.5, \gamma=0.9$ [S4].
    *   Compact 3DGS: $\lambda_m$ (hyperparameter) [S3].

### Edge Cases and Pitfalls
*   **Codebook Application:** In Compact 3DGS, $L_r$ and $L_s$ are zero for the majority of training (only active in the last 1K iterations) to avoid heavy computation [S3].
*   **Depth Smoothness:** The smoothness loss $\ell_{smooth}$ does not require ground-truth depth and relies on image gradients to regularize estimated depth [S4].
*   **Perceptual Loss:** Uses VGG feature space distance [S4].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_overall_loss(method, rendered_images, gt_images, gt_depths=None, 
                         predicted_depths=None, gaussian_attrs=None, 
                         codebooks=None, iteration=None, total_iterations=None):
    """
    Computes the overall training loss based on the specified method.
    
    Args:
        method: String identifier ('PDRF', 'TDNeRF', 'Compact3DGS', 'ReSplat')
        rendered_images: Tensor of shape (B, H, W, 3)
        gt_images: Tensor of shape (B, H, W, 3)
        predicted_depths: Tensor of shape (B, H, W)
        gaussian_attrs: Dict containing 'rotation', 'scale', etc.
        codebooks: List of codebook tensors
        iteration: Current training iteration
        total_iterations: Total number of training iterations
        
    Returns:
        Scalar loss value
    """
    if method == 'PDRF':
        # L_tot = L_CRR_photo + L_FVR_photo + λ(L_TV_coar + L_TV_fine)
        # Note: Specific implementation of CRR/FVR/TV losses assumed from context
        l_crr = compute_photometric_loss(rendered_images, gt_images) # Placeholder for CRR
        l_fvr = compute_photometric_loss(rendered_images, gt_images) # Placeholder for FVR
        l_tv_coar = compute_total_variation(gaussian_attrs['coar_features'])
        l_tv_fine = compute_total_variation(gaussian_attrs['fine_features'])
        lambda_tv = 1.0 # Not given in sources
        return l_crr + l_fvr + lambda_tv * (l_tv_coar + l_tv_fine)

    elif method == 'TDNeRF':
        # L = L_rgb + λ1 L_self-depth + λ2 L_GPC + λ3 L_reproj
        l_rgb = compute_rgb_loss(rendered_images, gt_images)
        l_self_depth = compute_self_supervised_depth_loss(predicted_depths, gt_images)
        l_gpc = compute_gaussian_point_constraint_loss(predicted_depths)
        l_reproj = compute_reprojection_loss(rendered_images, gaussian_attrs['cameras'])
        
        lambda1 = 0.04
        lambda2 = 1.0
        lambda3 = 1.0
        
        return l_rgb + lambda1 * l_self_depth + lambda2 * l_gpc + lambda3 * l_reproj

    elif method == 'Compact3DGS':
        # L = L_ren + λm L_m + L_r + L_s
        l_ren = compute_rendering_loss(rendered_images, gt_images) # L1 + SSIM
        l_m = compute_masking_loss(gaussian_attrs['opacity'])
        
        # Codebook losses only active in last 1K iterations
        if total_iterations - iteration <= 1000:
            l_r = compute_codebook_loss(gaussian_attrs['rotation'], codebooks['rotation'])
            l_s = compute_codebook_loss(gaussian_attrs['scale'], codebooks['scale'])
        else:
            l_r = 0.0
            l_s = 0.0
            
        lambda_m = 1.0 # Not given in sources
        return l_ren + lambda_m * l_m + l_r + l_s

    elif method == 'ReSplat':
        # Two-stage logic handled by caller, this computes the specific stage loss
        # Assuming Stage 2 context for recurrent loss, or Stage 1 for initial
        # This function computes the base rendering loss component
        l_render = compute_rendering_loss(rendered_images, gt_images) # L1 + Perceptual
        return l_render

    else:
        raise ValueError(f"Unknown method: {method}")

def compute_rendering_loss(rendered, gt):
    """Computes L1 + Perceptual loss."""
    l1 = torch.mean(torch.abs(rendered - gt))
    # Perceptual loss requires VGG features, simplified here
    l_percept = 0.0 # Placeholder for VGG feature distance
    lambda_percept = 0.5 # From ReSplat
    return l1 + lambda_percept * l_percept

def compute_codebook_loss(attributes, codebook):
    """Computes codebook loss with stop-gradient."""
    # L = || sg[attr - attr_prev] - codebook[index] ||^2
    # Simplified: assumes index lookup and previous attribute are available
    # In practice, this requires tracking previous attributes and codebook indices
    pass
```

## Subtopics

### [Loss Function Configuration](6782-loss-function-configuration.md)
Defines how individual loss terms are selected and combined.

### [Total Variation Loss](7047-total-variation-loss.md)
Enforces smoothness on feature volumes.

### [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md)
Consistency with sparse depth measurements.

### [Discriminator Loss with Gradient Penalty](7695-discriminator-loss-with-gradient-penalty.md)
Adversarial training component.

### [Loss Weight Configuration](7883-loss-weight-configuration.md)
Specific hyperparameters for weighting terms.

### [Photometric and Image Reconstruction Losses](9789-photometric-and-image-reconstruction-losses.md)
L1, SSIM, and perceptual losses.

### [Specific Method Implementations](9794-specific-method-implementations.md)
Method-specific loss details.

### [Numerical Gradient Eikonal Loss](9954-numerical-gradient-eikonal-loss.md)
Eikonal regularization.

## Operating characteristics
*   **Complexity:** The computational cost depends on the number of loss terms and the complexity of the renderer. PDRF involves two rendering passes (coarse and fine) [S1]. ReSplat involves recurrent iterations [S4].
*   **Performance:** PDRF uses a batch size of 1024 rays [S1]. Compact 3DGS limits codebook loss computation to the last 1K iterations to ensure fast training [S3].
*   **Strengths:** Combines multiple constraints (photometric, geometric, regularization) for robust optimization.
*   **Limitations:** Requires careful tuning of hyperparameters (e.g., $\lambda$ weights). Some losses (like codebook losses) are computationally expensive and must be scheduled [S3].

## Background
The concept of an overall training loss in differentiable volume rendering evolved from early NeRF works that primarily used photometric loss. Subsequent works introduced geometric constraints (depth, normal consistency) and regularization (TV, Eikonal) to improve convergence and quality. Specific implementations like PDRF [S1], TD-NeRF [S2], Compact 3DGS [S3], and ReSplat [S4] represent recent advancements in handling specific challenges like blur, pose estimation, compactness, and temporal consistency.

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
[S3] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S4] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
