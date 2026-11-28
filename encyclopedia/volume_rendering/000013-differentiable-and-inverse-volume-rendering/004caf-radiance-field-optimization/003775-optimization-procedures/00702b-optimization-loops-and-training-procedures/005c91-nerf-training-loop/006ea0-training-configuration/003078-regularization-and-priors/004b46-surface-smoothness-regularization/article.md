# Surface Smoothness Regularization

Also called: Depth Smoothness Regularization, Surface Smoothness on Integrated Unobserved Rays

## Overview
Surface Smoothness Regularization imposes piece-wise smoothness constraints on rendered depth from unobserved rays to alleviate spatial ambiguity and stabilize the joint optimization of blur kernels and radiance fields in sparse-view DeRF [S1]. It distinguishes itself from standard NeRF regularization by integrating "hidden rays" derived from the estimated blur kernel alongside standard "unseen rays," specifically targeting the coupled uncertainty of defocus and geometry [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of target camera poses $\{P_i^{\text{target}}\}$, an estimated blur kernel $h$, and the current radiance field parameters [S1].
*   **Outputs:** A scalar loss value $\mathcal{L}_{ss}$ added to the total training objective [S1].

**Data Structures**
*   **Target Poses:** $P_i^{\text{target}} = [R_i^{\text{target}} | t_i^{\text{target}}] \in \text{SE}(3)$ [S1].
*   **Sampled Poses:** $S_P$, a set of camera poses $[R|t]$ [S1].
*   **Integrated Unobserved Rays:** $r_{iu}$, a union of hidden rays $r_q$ and unseen rays $r_{us}$ [S1].
*   **Depth Map:** $\hat{d}$, computed per ray [S1].
*   **Color Map:** $\hat{C}$, computed per ray [S1].

**Step-by-Step Method**

1.  **Define Sample Space for Unseen Rays:**
    Determine the bounding box for camera translation $t$ based on target poses:
    $$ t_{\min} = \min(\{t_i^{\text{target}}\}), \quad t_{\max} = \max(\{t_i^{\text{target}}\}) $$
    Sample translation $t$ uniformly within $[t_{\min}, t_{\max}]$ [S1].

2.  **Compute Mean Focus Point:**
    Calculate the mean focus point $\bar{p}_f$ by solving a least squares problem to find the central point of the scene [S1].

3.  **Sample Camera Rotation:**
    Calculate the normalized mean over the up axes of all target poses, denoted $\bar{p}_u$. Sample a rotation matrix $R$ using:
    $$ S_R|t = \{R(\bar{p}_u, \bar{p}_f + \epsilon, t) \mid \epsilon \sim \mathcal{N}(0, 0.125)\} $$
    where $\epsilon$ is a small jitter value added to the focus point to ensure the camera roughly focuses on the scene center [S1].

4.  **Generate Unseen Rays ($r_{us}$):**
    Form the set of sampled camera poses:
    $$ S_P = \{[R|t] \mid R \sim S_R|t, t \sim S_t\} $$
    Generate rays $r_{us}$ from these poses [S1].

5.  **Generate Hidden Rays ($r_q$):**
    Derive rays $r_q$ from the estimated blur kernel $h$. The kernel transforms rays to approximate the color composition process of the blur. These rays are defined as:
    $$ r_q \sim h(r) $$
    where $r$ represents the main rays. These hidden rays provide supplementary information exclusively present in blurry inputs [S1].

6.  **Form Integrated Unobserved Rays:**
    Combine the two sets:
    $$ r_{iu} = \{r_q, r_{us}\} $$
    Note that $r_{us}$ remains unchanged during training, while $r_q$ changes as the blur kernel $h$ is optimized [S1].

7.  **Compute Expected Depth and Color:**
    For each ray $r \in r_{iu}$, compute the expected depth $\hat{d}$ and color $\hat{C}$:
    $$ \hat{d} = \int_{t_n}^{t_f} \sigma(r(t))t \, dt $$
    $$ \hat{C} = \int_{t_n}^{t_f} \sigma(r(t))c(r(t)) \, dt $$
    where $\sigma$ is density and $c$ is color [S1].

8.  **Calculate Weighted Depth Differences:**
    For a patch of size $S_{ptc} \times S_{ptc}$ centered at ray $r_{i,j}$, compute horizontal and vertical weighted depth differences:
    $$ \Delta \hat{d}_x(r_{i,j}) = \omega_{i+1,j} (\hat{d}(r_{i+1,j}) - \hat{d}(r_{i,j}))^2 $$
    $$ \Delta \hat{d}_y(r_{i,j}) = \omega_{i,j+1} (\hat{d}(r_{i,j+1}) - \hat{d}(r_{i,j}))^2 $$
    The weights $\omega$ are based on pixel-wise color difference:
    $$ \omega_{i+k,j+l} = \exp(-|\hat{C}(r_{i,j}) - \hat{C}(r_{i+k,j+l})|) $$
    This ensures smoothness is enforced where colors are similar, preserving edges [S1].

9.  **Compute Loss:**
    Sum the differences over all rays in $r_{iu}$ and all positions in the patch:
    $$ \mathcal{L}_{ss}\{r_{iu}\} = \sum_{r \in r_{iu}} \sum_{i,j=1}^{S_{ptc}-1} (\Delta \hat{d}_x(r_{i,j}) + \Delta \hat{d}_y(r_{i,j})) $$

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_surface_smoothness_loss(target_poses, blur_kernel, radiance_field, patch_size):
    """
    Computes the Surface Smoothness Regularization loss.
    
    Args:
        target_poses: List of SE(3) poses [R|t] used for training.
        blur_kernel: The current estimated blur kernel h.
        radiance_field: The radiance field model (NeRF/DeRF).
        patch_size: Integer size of the depth patch for smoothness calculation.
        
    Returns:
        Scalar loss value.
    """
    # 1. Define sample space for translations
    t_min = min(t for _, t in target_poses)
    t_max = max(t for _, t in target_poses)

    # 2. Compute mean focus point [G]
    # Standard least-squares fit to scene center is used.
    p_f = solve_least_squares_focus(target_poses)

    # 3. Compute mean up axis
    up_axes = [R[:, 2] for R, _ in target_poses]  # Third column of rotation matrix
    p_u = normalize(sum(up_axes) / len(up_axes))

    # 4. Generate unseen rays
    unseen_rays = []
    num_unseen_samples = 100  # [G] Sample count not specified in sources
    for _ in range(num_unseen_samples):
        t = uniform_sample(t_min, t_max)
        epsilon = normal_sample(0, 0.125)
        R = compute_rotation(p_u, p_f + epsilon, t)
        unseen_rays.append((R, t))

    # 5. Generate hidden rays
    hidden_rays = []
    for R, t in target_poses:
        r_q = transform_by_blur_kernel((R, t), blur_kernel)
        hidden_rays.append(r_q)

    # 6. Integrated unobserved rays
    integrated_rays = hidden_rays + unseen_rays

    # 7-9. Compute loss over patches
    total_loss = 0.0
    # Assume rays are indexed in a 2D grid of size patch_size x patch_size
    for r_idx in range(len(integrated_rays)):
        # Map 1D index to 2D grid coordinates (i, j)
        i, j = divmod(r_idx, patch_size)
        if i >= patch_size - 1 or j >= patch_size - 1:
            continue  # Skip boundary rays per summation limits

        ray_curr = integrated_rays[r_idx]
        d_curr, c_curr = radiance_field.integrate(ray_curr)

        # Horizontal neighbor (i+1, j)
        idx_h = r_idx + 1
        ray_h = integrated_rays[idx_h]
        d_h, c_h = radiance_field.integrate(ray_h)
        w_h = exp(-abs(c_curr - c_h))
        total_loss += w_h * (d_h - d_curr)**2

        # Vertical neighbor (i, j+1)
        idx_v = r_idx + patch_size
        ray_v = integrated_rays[idx_v]
        d_v, c_v = radiance_field.integrate(ray_v)
        w_v = exp(-abs(c_curr - c_v))
        total_loss += w_v * (d_v - d_curr)**2

    return total_loss
```

**Parameters**
*   **Jitter Standard Deviation:** $0.125$ for the normal distribution $\mathcal{N}(0, 0.125)$ [S1].
*   **Patch Size ($S_{ptc}$):** Not given in the sources.
*   **Number of Unseen Ray Samples:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Blur Kernel Convergence:** Hidden rays $r_q$ depend on the blur kernel $h$ and change during training [S1].
*   **Boundary Conditions:** The summation limits $i,j=1$ to $S_{ptc}-1$ imply that boundary rays in the patch do not contribute to the loss in the same way, or require padding [S1].
*   **Precision:** Computing depth integrals and exponential weights requires stable numerical integration to avoid gradient instability during backpropagation [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Stabilizes joint optimization of blur and geometry in sparse-view DeRF; alleviates spatial ambiguity by leveraging unobserved rays [S1].
*   **Limitations:** Relies on accurate estimation of the blur kernel for hidden rays; requires careful sampling of the camera pose space [S1].
*   **When to Use:** Sparse-view DeRF training where spatial ambiguity is high and blur kernel optimization is coupled with radiance field optimization [S1].
*   **When Not to Use:** Not given in the sources.

## Background
Originated in the context of Deblurred Neural Radiance Fields (DeRF) from sparse views [S1]. The method was introduced in "Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View" (2025) to address the difficulty of regularizing complex joint optimization in DeRF, where standard NeRF regularization fails [S1]. It builds on the concept of using unseen rays for regularization as seen in RegNeRF [6], but extends it with blur-kernel-derived hidden rays [S1].

## Sources
[S1] Sparse-DeRF: Deblurred Neural Radiance Fields From Sparse View, https://doi.org/10.1109/tpami.2025.3563398, fetched 2026-10-27
