# Reliable Depth Estimation via Reprojection

Also called: Reprojection-based Depth Reliability Masking

## Overview
This method selects reliable monocular depth estimates for radiance field supervision by comparing the reprojection consistency of depth predictions from two different models (a main model and an augmented model) against a nearest training view. It solves the problem of noisy or incorrect monocular depth priors by discarding depth values where the reprojection error exceeds a threshold or where neither model provides a consistent geometric match. It differs from its siblings by using a dual-model comparison to determine reliability rather than a single model's confidence or static thresholds, and by operating as a pre-supervision masking step rather than a direct differentiable loss term itself.

## How it works
**Inputs:**
- $z_m$: Depth map predicted by the main monocular depth estimation model.
- $z_a$: Depth map predicted by an augmented monocular depth estimation model (e.g., using smoothing or Lambertian augmentation).
- $V_{train}$: A set of training views with known camera poses and intrinsics.
- $q$: A pixel coordinate in the current view.
- $e_\tau$: A threshold for reprojection error (MSE).

**Outputs:**
- $m_m$: Binary reliability mask for the main model ($1$ if reliable, $0$ otherwise).
- $m_a$: Binary reliability mask for the augmented model ($1$ if reliable, $0$ otherwise).
- $z_{reliable}$: The selected reliable depth value for each pixel, derived from either $z_m$ or $z_a$.

**Data Structures:**
- Depth maps $z_m, z_a$: 2D arrays of floating-point depth values, same resolution as the input image.
- Masks $m_m, m_a$: 2D boolean or binary integer arrays, same resolution.

**Steps:**
1. **Reprojection:** For each pixel $q$ in the current view, unproject the depth value $z_m(q)$ and $z_a(q)$ into 3D space using the current camera's intrinsic and extrinsic parameters. Transform these 3D points into the coordinate system of the nearest training view $V_{nearest} \in V_{train}$. Project these transformed 3D points back onto the image plane of $V_{nearest}$ to obtain pixel coordinates $q'_m$ and $q'_a$.
2. **Patch Extraction:** Extract a patch of pixels around $q'_m$ in $V_{nearest}$ corresponding to the neighborhood of $q$ in the current view. Similarly, extract the patch around $q'_a$. Let $I_{curr}(q)$ be the intensity patch around $q$ in the current view. Let $I_{nearest}(q'_m)$ and $I_{nearest}(q'_a)$ be the intensity patches in the nearest training view.
3. **Compute Reprojection Error:** Calculate the Mean Squared Error (MSE) of intensities between the current view patch and the reprojected patches in the nearest view.
   - $e_m = \text{MSE}(I_{curr}(q), I_{nearest}(q'_m))$
   - $e_a = \text{MSE}(I_{curr}(q), I_{nearest}(q'_a))$
   [G] The MSE is computed over the pixel intensities in the patch.
4. **Compute Reliability Masks:** Determine which model provides a more reliable depth estimate based on the reprojection errors and the threshold $e_\tau$.
   - The augmented model is considered reliable if its reprojection error is lower than the main model's AND below the threshold:
     $$ m_a(q) = \begin{cases} 1 & \text{if } (e_a(q) \le e_m(q)) \land (e_a(q) \le e_\tau) \\ 0 & \text{otherwise} \end{cases} $$
   - The main model is considered reliable if its reprojection error is lower than the augmented model's AND below the threshold:
     $$ m_m(q) = \begin{cases} 1 & \text{if } (e_m(q) \le e_a(q)) \land (e_m(q) \le e_\tau) \\ 0 & \text{otherwise} \end{cases} $$
   [G] If both errors are equal and below threshold, both masks may be 1, or a tie-breaking rule is needed. The source implies mutual exclusivity in selection for supervision but defines masks independently.
5. **Select Reliable Depth:** Use the masks to select the depth value for supervision. If $m_a(q)=1$, use $z_a(q)$. If $m_m(q)=1$, use $z_m(q)$. If neither is reliable, the pixel is excluded from supervision.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_reprojection_masks(z_m, z_a, current_view, nearest_train_view, e_tau):
    """
    Computes reliability masks for main and augmented depth maps.
    """
    # Unproject and reproject depths
    # Assume function unproject_reproject(depth_map, from_cam, to_cam) returns 
    # the reprojected depth values and the MSE of intensity patches.
    # For simplicity, we assume we have access to the intensity patches.
    
    # Get intensity patches for current view and nearest training view
    # This requires aligning the views. 
    # Let's assume we have a function get_reprojection_mse that computes e_m and e_a
    # for all pixels.
    
    e_m, e_a = get_reprojection_mse(z_m, z_a, current_view, nearest_train_view)
    
    # Initialize masks
    m_m = np.zeros_like(z_m, dtype=bool)
    m_a = np.zeros_like(z_a, dtype=bool)
    
    # Compute masks based on conditions
    # Condition for augmented model reliability
    cond_a = (e_a <= e_m) & (e_a <= e_tau)
    m_a = cond_a
    
    # Condition for main model reliability
    cond_m = (e_m <= e_a) & (e_m <= e_tau)
    m_m = cond_m
    
    return m_m, m_a, e_m, e_a

def get_reprojection_mse(z_m, z_a, current_view, nearest_train_view):
    """
    Computes the MSE of intensity patches after reprojection.
    """
    # 1. Unproject z_m and z_a to 3D points
    points_m = unproject_depth(z_m, current_view.intrinsics, current_view.extrinsics)
    points_a = unproject_depth(z_a, current_view.intrinsics, current_view.extrinsics)
    
    # 2. Transform points to nearest training view coordinate system
    points_m_trans = transform_points(points_m, current_view.extrinsics.inverse() @ nearest_train_view.extrinsics)
    points_a_trans = transform_points(points_a, current_view.extrinsics.inverse() @ nearest_train_view.extrinsics)
    
    # 3. Project back to nearest training view image plane
    coords_m = project_points(points_m_trans, nearest_train_view.intrinsics)
    coords_a = project_points(points_a_trans, nearest_train_view.intrinsics)
    
    # 4. Extract intensity patches from current view and nearest training view
    # Assume patch size is fixed, e.g., 5x5
    patch_size = 5
    current_img = current_view.image
    nearest_img = nearest_train_view.image
    
    # Compute MSE for each pixel
    # This is computationally expensive and usually done in a GPU kernel
    # Here we outline the logic
    
    e_m = np.zeros_like(z_m)
    e_a = np.zeros_like(z_a)
    
    for y in range(z_m.shape[0]):
        for x in range(z_m.shape[1]):
            # Get reprojected coordinates
            xm, ym = coords_m[y, x]
            xa, ya = coords_a[y, x]
            
            # Check if coordinates are within bounds
            if not is_valid_pixel(xm, ym, nearest_img.shape) or not is_valid_pixel(xa, ya, nearest_img.shape):
                e_m[y, x] = np.inf
                e_a[y, x] = np.inf
                continue
            
            # Extract patches
            patch_curr = get_patch(current_img, x, y, patch_size)
            patch_nearest_m = get_patch(nearest_img, xm, ym, patch_size)
            patch_nearest_a = get_patch(nearest_img, xa, ya, patch_size)
            
            # Compute MSE
            e_m[y, x] = np.mean((patch_curr - patch_nearest_m) ** 2)
            e_a[y, x] = np.mean((patch_curr - patch_nearest_a) ** 2)
            
    return e_m, e_a
```

**Parameters:**
- $e_\tau$: Threshold for reprojection error. Not given in the sources. Typical values depend on the intensity range and patch size.

**Edge Cases and Pitfalls:**
- **Specular Regions:** In specular regions, intensity mismatches may cause both $e_m$ and $e_a$ to be high, resulting in both masks being zero. This is intended, as it excludes unreliable supervision rather than applying incorrect supervision.
- **Occlusions:** If the reprojected point falls into an occluded region in the nearest training view, the intensity comparison may be invalid. The source does not specify how occlusions are handled, but the high MSE would likely lead to mask=0.
- **Precision:** Reprojection involves floating-point arithmetic. Precision errors in coordinate transformation may affect the accuracy of the patch alignment.

## Subtopics

### [Depth Supervision Loss (Laug)](8111-depth-supervision-loss-laug.md)
Uses the reliability masks generated by this method to apply a weighted L1 loss, dynamically adapting to reliability estimates.

### [Depth Consistency Check / Depth Outlier Removal](9363-depth-consistency-check-depth-outlier-removal.md)
Filters depth outliers using a fixed threshold on geometric consistency, acting as a pre-processing step rather than a differentiable loss.

### [Multi-view Consistency Depth Filtering](12319-multi-view-consistency-depth-filtering.md)
Uses the learned NeRF's rendered depth as a proxy to filter monocular depth estimates, leveraging geometric consistency across multiple viewpoints.

## Operating characteristics
- **Time Complexity:** High, due to the need to unproject, transform, and reproject depth maps for every pixel, and to extract and compare intensity patches. The complexity is $O(N \cdot P)$ where $N$ is the number of pixels and $P$ is the patch area.
- **Memory Complexity:** $O(N)$ to store the depth maps, masks, and intermediate reprojection coordinates.
- **Strengths:** Effectively filters out noisy depth estimates by leveraging geometric consistency. The dual-model approach increases robustness.
- **Limitations:** Computationally expensive. Relies on the availability of a nearest training view. Performance depends on the quality of the monocular depth estimators.
- **When to use:** When training radiance fields with sparse input views and noisy monocular depth priors, and when computational resources allow for the additional reprojection overhead.
- **When not to use:** When real-time performance is critical, or when no training views are available for reprojection.

## Background
This method is described in "Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions" [S1]. It is part of the regularization strategy for sparse input radiance fields, specifically addressing the issue of noisy monocular depth estimates. The method was developed to improve the reliability of depth supervision by using reprojection consistency as a proxy for depth accuracy.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
