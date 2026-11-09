# Reliable Depth Estimation via Reprojection
Also called: Reprojection-based Depth Reliability Masking

## Overview
This method identifies reliable depth estimates for differentiable volume rendering by comparing the photometric consistency of depth maps reprojected into a reference view. It solves the problem of noisy monocular depth priors by selecting the depth estimate (from a main model or an augmented model) that yields the lowest reprojection error, filtered by a strict intensity similarity threshold. It differs from its sibling [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md) by using explicit geometric reprojection and photometric verification rather than direct pseudo-depth regression.

## How it works
**Inputs:**
- $z_m$: Depth map predicted by the main model.
- $z_a$: Depth map predicted by an augmented model (e.g., smoothing or Lambertian augmentation).
- $I_{ref}$: Intensity image of the nearest training view (reference).
- $I_{curr}$: Intensity image of the current view.
- $T$: Threshold for reprojection MSE ($e_\tau$).

**Outputs:**
- $m_m$: Reliability mask for the main model depth.
- $m_a$: Reliability mask for the augmented model depth.
- $L_{aug}$: Depth supervision loss using reliable depths.

**Data Structures:**
- Masks $m_m, m_a$: Binary maps of same spatial dimensions as $z_m, z_a$.
- MSE values $e_m, e_a$: Scalar or per-pixel mean squared error of intensity differences.

**Steps:**
1. **Reprojection:** For each pixel $q$ in the current view, reproject the patch around $q$ using depth $z_m$ to the reference view coordinates. Compute the Mean Squared Error (MSE) of intensities between this reprojected patch and the corresponding patch in $I_{ref}$. Let this be $e_m$.
2. **Augmented Reprojection:** Repeat step 1 using depth $z_a$. Let this be $e_a$.
3. **Mask Calculation for Augmented Model ($m_a$):**
   $$
   m_a = \begin{cases} 
   1 & \text{if } (e_a \le e_m) \text{ and } (e_a \le e_\tau) \\
   0 & \text{otherwise}
   \end{cases}
   $$
   This indicates the augmented model is more reliable if its reprojection error is lower than the main model's AND below the absolute threshold.
4. **Mask Calculation for Main Model ($m_m$):**
   Compute $m_m$ similarly, checking if $e_m \le e_a$ and $e_m \le e_\tau$.
5. **Loss Computation:**
   Compute the loss using stop-gradient ($\nabla$) on the target depth to prevent backpropagation through the unreliable source.
   $$
   L_{aug} = \mathbb{1}\{m_a = 1\} \odot \| z_m - \nabla(z_a) \|_2 + \mathbb{1}\{m_m = 1\} \odot \| \nabla(z_m) - z_a \|_2
   $$
   Where $\odot$ is element-wise product. The final depth supervision loss is the sum of losses from different augmentations (e.g., smoothing and Lambertian).

**Parameters:**
- $e_\tau$: Threshold for reprojection MSE. Value not given in the sources.

**Edge Cases:**
- **Specular Regions:** In specular regions, reprojected intensities may not match, causing both masks to be zero. This results in no supervision for those pixels, avoiding incorrect gradients rather than applying a noisy penalty [S1].
- **Both Incorrect:** If both models predict incorrect depth, both $e_m$ and $e_a$ may exceed $e_\tau$, resulting in zero supervision [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reliable_depth_estimation(z_m, z_a, I_ref, I_curr, T, reprojection_func):
    """
    z_m: Tensor [H, W] depth from main model
    z_a: Tensor [H, W] depth from augmented model
    I_ref: Tensor [H, W, 3] reference view intensity
    I_curr: Tensor [H, W, 3] current view intensity
    T: float, threshold for MSE
    reprojection_func: Function that takes (depth, I_curr, camera_params) -> (reprojected_I, coords)
    """
    # 1. Reproject using main model depth
    # Assume reprojection_func returns reprojected intensity patch and valid mask
    # For simplicity, assume it returns MSE directly or we compute it
    # Note: Exact reprojection geometry (camera intrinsics/extrinsics) is assumed available
    e_m = compute_reprojection_mse(z_m, I_curr, I_ref, reprojection_func)
    
    # 2. Reproject using augmented model depth
    e_a = compute_reprojection_mse(z_a, I_curr, I_ref, reprojection_func)
    
    # 3. Compute masks
    # m_a: 1 if augmented is better AND below threshold
    mask_a = ((e_a <= e_m) & (e_a <= T)).float()
    
    # m_m: 1 if main is better AND below threshold
    mask_m = ((e_m <= e_a) & (e_m <= T)).float()
    
    # 4. Compute Loss
    # Use stop-gradient on the target depth
    # L_aug = mask_a * ||z_m - stop_grad(z_a)|| + mask_m * ||stop_grad(z_m) - z_a||
    
    # Element-wise L2 norm (Euclidean distance between scalars is absolute difference)
    # Assuming z_m, z_a are scalar depth maps
    loss_a = mask_a * torch.abs(z_m - z_a.detach())
    loss_m = mask_m * torch.abs(z_m.detach() - z_a)
    
    L_aug = loss_a + loss_m
    
    return L_aug, mask_m, mask_a

def compute_reprojection_mse(depth, I_curr, I_ref, reprojection_func):
    """
    Computes MSE between reprojected patch and reference patch.
    """
    # Reproject current view to reference view coordinates
    # This typically involves unprojecting depth to 3D, transforming to reference camera, 
    # and projecting back to 2D.
    # For this pseudocode, we assume a helper that returns the reprojected intensity image
    # and a validity mask.
    
    # Placeholder for complex geometric reprojection logic
    # In practice, this uses camera intrinsics and extrinsics
    I_reproj, valid_mask = reprojection_func(depth, I_curr)
    
    # Compute MSE only on valid pixels
    # Align I_reproj with I_ref (handling translation/rotation)
    # Simplified: assume perfect alignment for MSE calculation on valid pixels
    diff = (I_reproj - I_ref) ** 2
    mse = torch.mean(diff * valid_mask) / (torch.mean(valid_mask) + 1e-8)
    
    return mse
```

## Subtopics

### [Depth Supervision Loss (Laug)](8111-depth-supervision-loss-laug.md)
Uses dynamic masking based on reliability estimates to stabilize training, addressing noise in single-view depth priors.

### [Depth Consistency Check / Depth Outlier Removal](9363-depth-consistency-check-depth-outlier-removal.md)
Filters noisy depth data by verifying geometric consistency across multiple views, rejecting outliers rather than applying continuous penalties.

## Operating characteristics
- **Time Complexity:** Requires reprojection for both main and augmented models, doubling the reprojection cost per step.
- **Memory Complexity:** Requires storing depth maps for both models and intermediate reprojection buffers.
- **Strengths:** Robust to noisy monocular depth estimates by selecting the most photometrically consistent source. Avoids supervision in specular regions where depth is ambiguous.
- **Limitations:** Relies on the quality of the augmented models. If both models are poor, no supervision is provided. Requires access to training views for reprojection.
- **When to use:** When sparse or noisy monocular depth priors are available and photometric consistency can be leveraged.
- **When not to use:** In scenes with significant specularities or textureless regions where reprojection error is high for all models.

## Background
Originated in Simple-RF [S1], which regularizes sparse input radiance fields with simpler solutions. The method leverages the idea that correct depth estimates should yield consistent appearances when reprojected to other views.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
