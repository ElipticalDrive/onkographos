# Reliable Depth Estimation via Reprojection

Also called: Depth Supervision Loss (Laug), Depth Consistency Check, Depth Outlier Removal

## Overview
This method provides a data consistency mechanism for monocular depth estimation in radiance fields by leveraging geometric reprojection to identify reliable depth estimates and filter outliers. It solves the problem of noisy or incorrect depth predictions in sparse-view settings by verifying consistency across multiple camera viewpoints. It differs from [Monocular Depth Consistency Loss](4668-monocular-depth-consistency-loss.md) by using reprojection error of image intensities to determine reliability rather than a closed-form least-squares scale/shift fit, and from [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md) by explicitly filtering unreliable pixels via multi-view geometric checks.

## How it works
The method consists of two primary components: a Depth Consistency Check for outlier removal and a Depth Supervision Loss for training.

**Inputs:**
- Depth maps $D_i, D_j$ for frames $i$ and $j$.
- Camera poses and intrinsics for frames $i$ and $j$.
- Image intensities for the nearest training view.
- Depth predictions from a main model ($z_m$) and an augmented model ($z_a$).

**Outputs:**
- A geometric consistency mask $M_d$ for outlier removal.
- A reliability mask $m_a$ (and $m_m$) for loss weighting.
- The augmentation loss $L_{aug}$.

**Data Structures:**
- Depth maps: 2D arrays of float values.
- Masks: 2D boolean or binary arrays of same spatial dimensions as depth maps.
- Pixel coordinates $\mathbf{u}_i$: 2D integer arrays representing grid positions.

**Step 1: Depth Outlier Removal (Consistency Check)**
This step filters noisy depth data by verifying geometric consistency between frames [S2].
1. Unproject the depth map $D_i$ of frame $i$ to 3D points using camera intrinsics.
2. Reproject these 3D points into the coordinate system of a nearby view $j$ to obtain the projected depth $D_{i \to j}$.
3. Compare $D_{i \to j}$ with the actual depth map $D_j$ of view $j$.
4. Generate a mask $M_d$ where pixels are kept if the absolute difference is below a threshold $\sigma$.

The formula for the mask is:
$$ M_d = |D_{i \to j} - D_j| < \sigma $$
where $D_{i \to j} = D_j(\pi_j \pi_i^{-1}(\mathbf{u}_i))$, $\mathbf{u}_i$ are pixel coordinates of frame $i$, and $\pi$ denotes projection functions [S2].

**Step 2: Reliability Masking via Reprojection Error**
This step determines which depth estimate (main or augmented) is more reliable by comparing reprojection errors [S1].
1. For each pixel, compute the Mean Squared Error (MSE) of intensities between a patch reprojected using $z_m$ and the corresponding patch in the nearest training view. Let this be $e_m$.
2. Repeat for $z_a$ to get $e_a$.
3. Define a threshold $e_\tau$.
4. Compute mask $m_a$ indicating if the augmented model is more reliable:
   $$ m_a = \begin{cases} 1 & \text{if } (e_a \le e_m) \land (e_a \le e_\tau) \\ 0 & \text{otherwise} \end{cases} $$
5. Compute mask $m_m$ for the main model similarly (reliable if $e_m \le e_a$ and $e_m \le e_\tau$) [S1].

**Step 3: Depth Supervision Loss Calculation**
Compute the loss using the reliability masks and stop-gradient operators [S1].
$$ L_{aug} = \mathbb{1}\{m_a=1\} \odot \| z_m - \hat{\nabla}(z_a) \|^2 + \mathbb{1}\{m_m=1\} \odot \| \hat{\nabla}(z_m) - z_a \|^2 $$
where $\odot$ is element-wise product, $\mathbb{1}$ is the indicator function, and $\hat{\nabla}$ is the stop-gradient operator [S1].

**Parameters:**
- $\sigma = 0.2$m: Empirical threshold for absolute relative error in depth reprojection [S2].
- $e_\tau$: Threshold for reprojection MSE to filter incorrect depths [S1]. Value not given in sources.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_depth_consistency_mask(D_i, D_j, pose_i, pose_j, intrinsics, sigma=0.2):
    """
    Computes geometric consistency mask between frame i and j.
    """
    # Unproject D_i to 3D
    points_3d_i = unproject_depth_to_3d(D_i, pose_i, intrinsics)
    
    # Reproject to view j
    # Note: pi_j * pi_i^-1(u_i) logic handled by reprojection function
    D_i_to_j = reproject_depth(points_3d_i, pose_j, intrinsics)
    
    # Calculate absolute difference
    diff = abs(D_i_to_j - D_j)
    
    # Apply threshold
    M_d = diff < sigma
    return M_d

def compute_reliability_masks(z_m, z_a, nearest_view_image, patch_size, e_tau):
    """
    Computes reliability masks based on reprojection intensity MSE.
    """
    # Reproject patches using z_m and z_a to nearest view
    # Assume function returns MSE per pixel
    e_m = compute_reprojection_mse(z_m, nearest_view_image, patch_size)
    e_a = compute_reprojection_mse(z_a, nearest_view_image, patch_size)
    
    # Mask for augmented model reliability
    m_a = (e_a <= e_m) & (e_a <= e_tau)
    
    # Mask for main model reliability
    m_m = (e_m <= e_a) & (e_m <= e_tau)
    
    return m_a, m_m

def compute_depth_supervision_loss(z_m, z_a, m_a, m_m):
    """
    Computes the augmentation loss Laug.
    """
    # Stop-gradient operator
    z_m_sg = stop_gradient(z_m)
    z_a_sg = stop_gradient(z_a)
    
    # Term 1: If augmented is reliable, supervise main with augmented (stop-grad)
    term1 = m_a * (z_m - z_a_sg)**2
    
    # Term 2: If main is reliable, supervise augmented with main (stop-grad)
    term2 = m_m * (z_a - z_m_sg)**2
    
    L_aug = term1 + term2
    return L_aug
```

**Edge Cases and Pitfalls:**
- Specular regions may cause intensity mismatches in reprojection, resulting in zero masks. This reduces supervision coverage but prevents incorrect depth supervision [S1].
- If both models predict incorrect depth, both masks may be zero, providing no supervision for those pixels [S1].
- Precision issues in unprojection/reprojection can affect $M_d$ accuracy near depth discontinuities.

## Subtopics

### Depth Supervision Loss (Laug)
A specific formulation of the reliability-weighted loss where the main and augmented depth predictions supervise each other based on reprojection consistency. It sets apart from standard L1/L2 depth losses by dynamically masking unreliable pixels using multi-view intensity reprojection errors. Choose this when training radiance fields with sparse inputs where monocular depth priors are noisy. Link: [Reliable Depth Estimation via Reprojection](52-reliable-depth-estimation-via-reprojection.md).

## Operating characteristics
- **Time Complexity:** Dominated by reprojection operations. Unprojecting and reprojecting depth maps is $O(N)$ where $N$ is the number of pixels. Computing MSE over patches adds a constant factor based on patch size.
- **Memory Complexity:** Requires storing depth maps, masks, and intermediate 3D points. $O(N)$ per frame.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Effectively filters noisy depth data and specular artifacts. Improves reconstruction quality in sparse-view settings by ensuring only geometrically consistent depths are used for supervision [S1, S2].
- **Limitations:** Relies on the availability of nearby training views for reprojection. Fails in regions with no texture (specular/blank) where intensity MSE is unreliable.
- **When to use:** Sparse-view NeRF/Gaussian Splatting training where monocular depth estimates are used as regularizers.
- **When not to use:** Dense-view settings where depth consistency is trivial, or real-time applications where reprojection overhead is prohibitive.

## Background
The method originates from research into regularizing sparse input radiance fields. The reliability masking via reprojection error was introduced in "Simple-RF" [S1] to handle sparse inputs. The depth consistency check for outlier removal was detailed in "EVolSplat" [S2] for efficient volume-based Gaussian Splatting in urban environments.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
