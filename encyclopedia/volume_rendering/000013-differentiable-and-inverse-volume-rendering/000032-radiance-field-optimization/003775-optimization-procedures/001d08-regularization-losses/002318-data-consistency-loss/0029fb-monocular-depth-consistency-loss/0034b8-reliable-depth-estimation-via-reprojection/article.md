# Reliable Depth Estimation via Reprojection

Also called: Reprojection-based Depth Reliability Masking

## Overview
This method determines the reliability of monocular depth estimates by reprojecting them into a nearest training view and comparing the resulting image intensities against the actual captured view. It solves the problem of noisy or incorrect depth predictions (e.g., in specular regions or textureless areas) by generating a binary mask that selects only those depth values where the reprojection error is low. It differs from its siblings, [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md) and [Depth Prior Loss](10801-depth-prior-loss.md), by using geometric consistency via reprojection error rather than direct pixel-wise regression or static priors to weight the supervision signal.

## How it works
**Inputs:**
- Rendered depth map from the main model, $z_m$.
- Rendered depth map from an augmented model (e.g., smoothing or Lambertian augmentation), $z_a$.
- The nearest training view image $I_{train}$ corresponding to the current camera pose.
- Threshold parameter $e_\tau$ (typical value: Not given in the sources).

**Outputs:**
- Reliability masks $m_m$ (for main model) and $m_a$ (for augmented model).
- A weighted depth supervision loss.

**Data Structures:**
- Depth maps $z_m, z_a$: 2D arrays of floats, same resolution as $I_{train}$.
- Masks $m_m, m_a$: 2D binary arrays (0 or 1), same resolution.
- Reprojection error maps $e_m, e_a$: 2D arrays of floats (MSE).

**Steps:**
1. **Reprojection:** For each pixel $p$ in the current view, unproject the depth $z_m(p)$ and $z_a(p)$ to 3D points. Transform these points into the coordinate system of the nearest training view. Project them back to 2D coordinates $q_m$ and $q_a$ in the training view.
2. **Patch Extraction:** Extract a small patch of intensities from the training view image $I_{train}$ centered at $q_m$ and $q_a$. Let these patches be $P_m$ and $P_a$.
3. **Error Calculation:** Compute the Mean Squared Error (MSE) between the reprojected patch and the corresponding patch in the current view (or a reference patch if the current view is not the training view, but the source implies comparing reprojected patch to the *nearest training view* patch directly). Specifically, compute $e_m$ as the MSE of intensities for $z_m$ and $e_a$ for $z_a$.
   $$ e_m = \text{MSE}(P_m, I_{train}[q_m]) $$
   $$ e_a = \text{MSE}(P_a, I_{train}[q_a]) $$
   *Note: The source describes computing MSE between the reprojected patch and the corresponding patch in the nearest training view.*
4. **Mask Generation:** Compute the reliability mask $m_a$ for the augmented model. The depth $z_a$ is considered reliable if its reprojection error is lower than or equal to the main model's error AND below the threshold $e_\tau$.
   $$ m_a = \begin{cases} 1 & \text{if } (e_a \le e_m) \land (e_a \le e_\tau) \\ 0 & \text{otherwise} \end{cases} $$
   Similarly, compute $m_m$ for the main model:
   $$ m_m = \begin{cases} 1 & \text{if } (e_m \le e_a) \land (e_m \le e_\tau) \\ 0 & \text{otherwise} \end{cases} $$
5. **Loss Computation:** Compute the depth supervision loss using the stop-gradient operator $\nabla$ to prevent gradient flow through the reference depth. The loss is the sum of losses from the main and augmented models, weighted by their respective masks.
   $$ L_{aug} = \mathbb{1}\{m_a = 1\} \odot \| z_m - \nabla(z_a) \|_2 + \mathbb{1}\{m_m = 1\} \odot \| \nabla(z_m) - z_a \|_2 $$
   Where $\odot$ is element-wise multiplication and $\mathbb{1}$ is the indicator function. The final loss is the sum of such terms for all augmentations (e.g., smoothing and Lambertian).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_reprojection_loss(z_m, z_a, I_train, nearest_view_transform, patch_size, e_tau):
    """
    z_m: Tensor [H, W] depth from main model
    z_a: Tensor [H, W] depth from augmented model
    I_train: Tensor [H, W, 3] image from nearest training view
    nearest_view_transform: Function to transform 3D points from current view to training view
    patch_size: int, size of the patch for MSE calculation
    e_tau: float, threshold for reprojection error
    """
    # 1. Unproject and Reproject
    # Get 3D points for z_m and z_a
    pts_m = unproject_depth(z_m, current_camera_intrinsics, current_camera_pose)
    pts_a = unproject_depth(z_a, current_camera_intrinsics, current_camera_pose)
    
    # Transform to training view
    pts_m_train = nearest_view_transform(pts_m)
    pts_a_train = nearest_view_transform(pts_a)
    
    # Project to 2D coordinates in training view
    q_m = project_to_2d(pts_m_train, nearest_view_intrinsics)
    q_a = project_to_2d(pts_a_train, nearest_view_intrinsics)
    
    # 2. Extract Patches and Compute MSE
    # Extract patches from I_train at q_m and q_a
    # Note: In practice, handle boundary checks and interpolation
    patch_m = extract_patch(I_train, q_m, patch_size)
    patch_a = extract_patch(I_train, q_a, patch_size)
    
    # Compute MSE between reprojected content and training view patch
    # The source implies comparing the reprojected patch to the corresponding patch 
    # in the nearest training view. Assuming the "corresponding patch" refers to 
    # the patch in I_train at the projected location.
    e_m = mean_squared_error(patch_m, I_train[q_m]) # Conceptual indexing
    e_a = mean_squared_error(patch_a, I_train[q_a])
    
    # 3. Generate Masks
    # Mask for augmented model reliability
    m_a = ((e_a <= e_m) & (e_a <= e_tau)).float()
    
    # Mask for main model reliability
    m_m = ((e_m <= e_a) & (e_m <= e_tau)).float()
    
    # 4. Compute Loss
    # Use stop-gradient on the reference depth
    z_a_no_grad = z_a.detach()
    z_m_no_grad = z_m.detach()
    
    loss_term_a = m_a * torch.norm(z_m - z_a_no_grad, p=2)
    loss_term_m = m_m * torch.norm(z_m_no_grad - z_a, p=2)
    
    total_loss = loss_term_a + loss_term_m
    
    return total_loss, m_m, m_a
```

**Edge Cases and Pitfalls:**
- **Specular Regions:** Intensities may not match during reprojection, causing both masks to be zero. This results in no supervision for those pixels, which is intended to avoid incorrect depth estimates [S1].
- **Threshold Selection:** The value of $e_\tau$ is critical. If too low, few pixels are supervised; if too high, noisy depths are included. Not given in the sources.
- **Patch Size:** The size of the patch affects the robustness to small misalignments. Not given in the sources.

## Subtopics

### [Depth Supervision Loss (Laug)](8111-depth-supervision-loss-laug.md)
Uses dynamic reliability masks to stabilize training in early stages.

### [Depth Consistency Check / Depth Outlier Removal](9363-depth-consistency-check-depth-outlier-removal.md)
Filters noisy depth via geometric consistency across multiple views using fixed thresholds.

### [Multi-view Consistency Depth Filtering](12319-multi-view-consistency-depth-filtering.md)
Generates binary confidence maps by comparing scaled monocular depth with rendered NeRF depth across viewpoints.

## Operating characteristics
- **Time Complexity:** Depends on the number of pixels and the cost of reprojection and patch extraction. Typically $O(N)$ where $N$ is the number of pixels.
- **Memory Complexity:** Requires storing depth maps and intermediate reprojection coordinates. $O(N)$.
- **Strengths:** Robust to specularities and textureless regions by masking out unreliable pixels. Leverages geometric consistency.
- **Limitations:** Requires access to nearest training view images. Sensitive to the threshold $e_\tau$.
- **When to use:** When monocular depth estimates are noisy and need to be filtered before supervision.
- **When not to use:** When training views are not available or reprojection is computationally prohibitive.

## Background
Originated in the Simple-RF paper [S1], which regularizes sparse input radiance fields with simpler solutions. The method is part of the depth supervision strategy to handle the ambiguity of monocular depth estimation.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
