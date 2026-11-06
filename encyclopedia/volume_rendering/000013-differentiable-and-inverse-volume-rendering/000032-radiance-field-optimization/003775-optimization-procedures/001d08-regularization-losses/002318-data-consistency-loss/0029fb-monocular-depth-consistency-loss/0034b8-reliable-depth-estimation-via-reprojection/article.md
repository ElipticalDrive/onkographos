# Reliable Depth Estimation via Reprojection
Also called: Reprojection-based Depth Reliability Masking
## Overview
This method identifies reliable depth estimates for depth supervision by comparing the photometric consistency of reprojected patches from two different depth predictions ($z_m$ and $z_a$) against a reference training view. It sets itself apart by using a dual-model reprojection strategy to filter out incorrect depth estimates, ensuring that supervision is only applied where the depth prediction is geometrically consistent with observed image intensities. It builds on [Monocular Depth Consistency Loss](4668-monocular-depth-consistency-loss.md) by providing a mechanism to select *which* depth values are trustworthy before applying consistency constraints.
## How it works
**Inputs:**
- $z_m$: Depth map predicted by the main model.
- $z_a$: Depth map predicted by an augmented model (e.g., from a smoothing or Lambertian augmentation).
- $I_{ref}$: Intensity image from the nearest training view (reference view).
- $T_{m \to ref}$: Transformation matrix from the main model's camera pose to the reference view.
- $T_{a \to ref}$: Transformation matrix from the augmented model's camera pose to the reference view.
- $e_\tau$: Threshold for reprojection error (MSE).
- $q$: Pixel coordinates in the current view.

**Outputs:**
- $m_m$: Binary mask indicating where $z_m$ is reliable.
- $m_a$: Binary mask indicating where $z_a$ is reliable.
- $L_{aug}$: Depth supervision loss computed only on reliable pixels.

**Data Structures:**
- Masks $m_m, m_a$: Binary arrays of the same spatial dimensions as the depth maps, where 1 indicates reliability and 0 indicates unreliability.
- MSE values $e_m, e_a$: Scalar or per-pixel mean squared error values between reprojected patches and reference patches.

**Steps:**
1. **Reproject Patches:** For each pixel $q$ in the current view, extract a small patch around $q$ using depth $z_m$ and transform it to the reference view using $T_{m \to ref}$. Similarly, extract a patch using $z_a$ and transform it using $T_{a \to ref}$.
2. **Compute Reprojection Error:** Calculate the Mean Squared Error (MSE) between the reprojected patch from $z_m$ and the corresponding patch in $I_{ref}$. Let this be $e_m$. Repeat for $z_a$ to get $e_a$.
3. **Compute Reliability Masks:**
   - Compute mask $m_a$ for the augmented model:
     $$
     m_a = \begin{cases} 
     1 & \text{if } (e_a \le e_m) \text{ and } (e_a \le e_\tau) \\
     0 & \text{otherwise}
     \end{cases}
     $$
   - Compute mask $m_m$ for the main model similarly:
     $$
     m_m = \begin{cases} 
     1 & \text{if } (e_m \le e_a) \text{ and } (e_m \le e_\tau) \\
     0 & \text{otherwise}
     \end{cases}
     $$
   - Note: If both $e_m$ and $e_a$ exceed $e_\tau$, both masks are 0, meaning no supervision is applied for that pixel.
4. **Compute Supervision Loss:** Apply depth supervision only where masks are 1. The loss $L_{aug}$ is computed as:
   $$
   L_{aug} = \mathbb{1}\{m_a = 1\} \odot \| z_m - \nabla(z_a) \|_2^2 + \mathbb{1}\{m_m = 1\} \odot \| \nabla(z_m) - z_a \|_2^2
   $$
   where $\nabla$ is the stop-gradient operator, $\odot$ is element-wise multiplication, and $\mathbb{1}$ is the indicator function.
5. **Aggregate Loss:** Sum the losses from different augmentations (e.g., smoothing and Lambertian) to get the final depth supervision loss.

**Parameters:**
- $e_\tau$: Threshold for reprojection error. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Specular Regions:** In specular regions, reprojected patches may not match the reference view due to view-dependent appearance, leading to high MSE. This results in masks being 0, which correctly filters out unreliable supervision rather than applying incorrect supervision.
- **Both Models Incorrect:** If both $z_m$ and $z_a$ are incorrect, both $e_m$ and $e_a$ may exceed $e_\tau$, resulting in no supervision. This is a desired behavior to avoid propagating errors.
- **Precision:** Ensure that the reprojection and patch extraction are performed with sufficient precision to avoid artifacts in the MSE calculation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_reprojection_loss(z_m, z_a, I_ref, T_m_to_ref, T_a_to_ref, e_tau, patch_size):
    """
    Compute reliable depth supervision loss via reprojection.
    
    Args:
        z_m: Depth map from main model (H, W)
        z_a: Depth map from augmented model (H, W)
        I_ref: Reference view intensity image (H_ref, W_ref, C)
        T_m_to_ref: Transformation matrix from main model to reference view (4, 4)
        T_a_to_ref: Transformation matrix from augmented model to reference view (4, 4)
        e_tau: Threshold for reprojection error
        patch_size: Size of the patch around each pixel
        
    Returns:
        L_aug: Depth supervision loss
        m_m: Reliability mask for main model
        m_a: Reliability mask for augmented model
    """
    H, W = z_m.shape
    
    # Initialize masks
    m_m = np.zeros((H, W), dtype=np.float32)
    m_a = np.zeros((H, W), dtype=np.float32)
    
    # Initialize loss accumulator
    L_aug = 0.0
    
    # Precompute reprojection coordinates and patches
    # Note: This is a simplified representation. In practice, use differentiable rendering or ray casting.
    
    # Reproject z_m to reference view
    # For each pixel q in current view, compute reprojected coordinates in reference view
    # Extract patch from I_ref at reprojected coordinates using z_m
    # Compute MSE between reprojected patch and reference patch
    # Let e_m be the MSE for z_m
    
    # Reproject z_a to reference view
    # Similarly compute e_a for z_a
    
    # Compute masks
    # m_a = 1 if (e_a <= e_m) and (e_a <= e_tau) else 0
    # m_m = 1 if (e_m <= e_a) and (e_m <= e_tau) else 0
    
    # Compute loss
    # L_aug = sum(m_a * ||z_m - stop_gradient(z_a)||^2 + m_m * ||stop_gradient(z_m) - z_a||^2)
    
    # Return loss and masks
    return L_aug, m_m, m_a
```
## Subtopics
## Operating characteristics
**Time Complexity:** The method involves reprojection and patch extraction for each pixel, which is $O(H \times W)$ per pixel, where $H$ and $W$ are the image dimensions. The MSE computation is $O(P)$, where $P$ is the patch size.
**Memory Complexity:** Requires storing depth maps, reference image, and masks, which is $O(H \times W)$.
**Strengths:** Effectively filters out incorrect depth estimates, leading to sharper reconstructions. Handles specular regions gracefully by excluding them from supervision.
**Limitations:** Relies on the availability of a reference view and accurate camera poses. Performance may degrade in regions with low texture or high noise.
**When to Use:** When sparse depth supervision is available and the goal is to regularize the depth prediction using photometric consistency.
**When Not to Use:** When no reference views are available or when the scene has significant dynamic elements that violate photometric consistency.
## Background
The method was introduced in "Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions" [S1]. It is part of a broader effort to improve the quality of radiance field reconstructions from sparse input views by leveraging depth supervision and reprojection-based reliability masking.
## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
