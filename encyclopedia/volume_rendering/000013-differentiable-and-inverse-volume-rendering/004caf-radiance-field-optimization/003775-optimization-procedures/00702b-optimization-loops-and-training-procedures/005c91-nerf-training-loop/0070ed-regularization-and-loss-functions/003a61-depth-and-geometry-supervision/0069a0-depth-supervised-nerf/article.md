# Depth-Supervised NeRF


## Overview
Depth-Supervised NeRF (DS-NeRF) is a training procedure for Neural Radiance Fields that incorporates sparse 3D depth supervision from Structure-from-Motion (SfM) point clouds to accelerate convergence and improve reconstruction quality with fewer input views [S1, S2][S3, unofficial]. Unlike methods relying on category-specific priors or dense multi-view stereo initialization, DS-NeRF treats the sparse keypoints generated during standard SfM preprocessing as free geometric constraints, enforcing that the rendered ray termination distribution aligns with these surface priors [S1]. This approach reduces overfitting in sparse-view settings and typically achieves target quality in 2–3× fewer training iterations than standard NeRF [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $\{I_j\}$ with known camera poses $\{P_j\}$ (estimated via SfM, e.g., COLMAP); sparse 3D keypoints $\{X\}$ and their visibility flags from the SfM pipeline; near and far clipping bounds $t_n, t_f$ for ray marching [S1].
*   **Outputs:** A trained NeRF model (MLP parameters $\theta$ for density $\sigma$ and color $\mathbf{c}$) capable of rendering novel views [S1].

**Data Structures:**
*   **Keypoints:** A set $X = \{x_i\}$ where each $x_i \in \mathbb{R}^3$ [S1].
*   **Visibility:** For each image $j$, a subset $X_j \subset X$ of keypoints visible from camera $P_j$ [S1].
*   **Depth Estimates:** For each visible keypoint $x_i \in X_j$, a scalar depth $D_{ij}$ and uncertainty $\hat{\sigma}_i$ [S1].
*   **Ray Termination Distribution:** Discretized values $h_k = h(t_k)$ computed during volume rendering [S1].

**Step-by-Step Method:**

1.  **SfM Preprocessing:** Run an SfM solver (e.g., COLMAP) on the input images to estimate camera poses $P_j$ and reconstruct a sparse 3D point cloud $X$ [S1][S3, unofficial].
2.  **Depth and Uncertainty Estimation:**
    *   For each image $I_j$ and camera $P_j$, identify visible keypoints $X_j$ [S1].
    *   Compute the projected depth $D_{ij}$ for each $x_i \in X_j$ by projecting $x_i$ with $P_j$ and taking the re-projected z-value (depth along the camera axis) [S1].
    *   Compute the depth uncertainty $\hat{\sigma}_i$ for each keypoint $x_i$ as the average reprojection error across all views where $x_i$ was detected [S1].
    *   Model the true depth of the first surface as a random variable $D_{ij} \sim \mathcal{N}(D_{ij}, \hat{\sigma}_i)$ [S1].
3.  **Ray Generation for Supervision:**
    *   For each detected keypoint $x_i \in X_j$, define a ray $\mathbf{r}$ originating from the camera center $o_j$ with direction $\mathbf{d}$ pointing towards the projection of $x_i$ on the image plane [S1].
    *   In local camera coordinates, the ray direction is derived from the keypoint's image coordinates $(u, v)$ and $-1$ in the camera axis [S1].
    *   To combine color and depth supervision efficiently, the method exploits the continuity of image coordinates. Rays corresponding to keypoints are sampled within the same ray marching procedure used for standard pixel rays, interpolating RGB supervision values for these specific rays [S1].
4.  **Volume Rendering:**
    *   Shoot rays $\mathbf{r}$ into the scene using standard NeRF sampling between $t_n$ and $t_f$ [S1].
    *   Compute the ray termination distribution $h(t) = T(t)\sigma(t)$, where $T(t) = \exp(-\int_0^t \sigma(s)ds)$ is the transmittance [S1].
    *   Discretize this distribution at sample points $t_k$ with step size $\Delta t_k$: $h_k = h(t_k)$ [S1].
    *   Compute the rendered color $\hat{\mathbf{C}}(\mathbf{r})$ and rendered depth $\hat{D}(\mathbf{r})$ [S1][S4, unofficial].
5.  **Loss Computation:**
    *   **Color Loss:** Standard photometric loss $L_{Color} = \|\hat{\mathbf{C}}(\mathbf{r}) - \mathbf{I}[u, v]\|_2^2$ [S1][S4, unofficial].
    *   **Depth Loss:** Minimize the KL divergence between the rendered ray termination distribution $h(t)$ and the noisy depth distribution $\mathcal{N}(D_{ij}, \hat{\sigma}_i)$ [S1]. The loss is approximated as:
        $$ L_{Depth} = \mathbb{E}_{x_i \in X_j} \left[ \sum_k -\log h_k \exp \left( -\frac{(t_k - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \Delta t_k \right] $$
        where $t_k$ are the sample depths along the ray [S1].
        *Alternative:* An MSE loss between rendered depth and $D_{ij}$ can be used, but KL divergence yields fewer artifacts [S1][S3, unofficial].
    *   **Total Loss:**
        $$ L = L_{Color} + \lambda_D L_{Depth} $$
        where $\lambda_D$ is a hyperparameter balancing the terms [S1].
6.  **Optimization:** Update network parameters $\theta$ using gradients backpropagated through the differentiable volume rendering process [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_ray_termination_distribution(sigma, t_starts, t_ends):
    """
    Computes the ray termination distribution h(t) = T(t) * sigma(t).
    Args:
        sigma: List of lists, shape (num_rays, num_samples), differential density.
        t_starts: List of lists, shape (num_rays, num_samples), start depths.
        t_ends: List of lists, shape (num_rays, num_samples), end depths.
    Returns:
        h: List of lists, shape (num_rays, num_samples), termination distribution values.
        T: List of lists, shape (num_rays, num_samples), accumulated transmittance.
    """
    h_list = []
    T_list = []
    for ray_sigma, ray_t_starts, ray_t_ends in zip(sigma, t_starts, t_ends):
        num_samples = len(ray_sigma)
        T = []
        h = []
        cum_sigma_delta = 0.0
        for i in range(num_samples):
            delta = ray_t_ends[i] - ray_t_starts[i]
            sigma_delta = ray_sigma[i] * delta
            T_i = math.exp(-cum_sigma_delta)
            T.append(T_i)
            h_i = T_i * ray_sigma[i]
            h.append(h_i)
            cum_sigma_delta += sigma_delta
        h_list.append(h)
        T_list.append(T)
    return h_list, T_list

def depth_supervised_nerf_loss(sigma, t_starts, t_ends, keypoints_depths, keypoint_uncertainties, 
                               rendered_colors, gt_colors, ray_indices_for_keypoints):
    """
    Computes the combined loss for DS-NeRF.
    Args:
        sigma: List of lists, shape (num_rays, num_samples)
        t_starts: List of lists, shape (num_rays, num_samples)
        t_ends: List of lists, shape (num_rays, num_samples)
        keypoints_depths: List of scalars, depth D_ij from SfM
        keypoint_uncertainties: List of scalars, uncertainty sigma_hat_i
        rendered_colors: List of lists, shape (num_rays, 3)
        gt_colors: List of lists, shape (num_rays, 3)
        ray_indices_for_keypoints: List of indices in the batch corresponding to keypoint rays
    Returns:
        total_loss: float
    """
    # 1. Color Loss for all rays
    color_loss = 0.0
    for r_c, g_c in zip(rendered_colors, gt_colors):
        color_loss += sum((r - g) ** 2 for r, g in zip(r_c, g_c))
    color_loss /= len(rendered_colors)
    
    # 2. Depth Loss for keypoint rays
    if not ray_indices_for_keypoints:
        return color_loss
        
    # Extract sigma and t for keypoint rays
    sigma_k = [sigma[i] for i in ray_indices_for_keypoints]
    t_starts_k = [t_starts[i] for i in ray_indices_for_keypoints]
    t_ends_k = [t_ends[i] for i in ray_indices_for_keypoints]
    
    # Compute h(t) for keypoint rays
    h_k, _ = compute_ray_termination_distribution(sigma_k, t_starts_k, t_ends_k)
    
    # Discretization step size
    deltas_k = [[t_ends_k[i][j] - t_starts_k[i][j] for j in range(len(t_starts_k[i]))] for i in range(len(t_starts_k))]
    
    # Sample points t_k (using midpoints as standard approximation)
    t_samples_k = [[t_starts_k[i][j] + deltas_k[i][j] / 2.0 for j in range(len(t_starts_k[i]))] for i in range(len(t_starts_k))]
    
    # KL Divergence approximation:
    # L_depth = sum_k -log(h_k) * exp(-(t_k - D_ij)^2 / (2 * sigma_hat_i^2)) * delta_t_k
    depths_k = [keypoints_depths[i] for i in ray_indices_for_keypoints]
    uncertainties_k = [keypoint_uncertainties[i] for i in ray_indices_for_keypoints]
    
    depth_loss_sum = 0.0
    num_keypoint_rays = len(ray_indices_for_keypoints)
    
    for i in range(num_keypoint_rays):
        ray_loss = 0.0
        for j in range(len(h_k[i])):
            h_val = h_k[i][j]
            # Clamp to avoid log(0)
            h_val = max(h_val, 1e-6)
            gaussian_term = math.exp(-((t_samples_k[i][j] - depths_k[i]) ** 2) / (2 * (uncertainties_k[i] ** 2)))
            ray_loss += -math.log(h_val) * gaussian_term * deltas_k[i][j]
        depth_loss_sum += ray_loss
        
    depth_loss = depth_loss_sum / num_keypoint_rays
    
    # Total Loss
    lambda_d = 1.0 # Empirically set to 1 [S4, unofficial]
    total_loss = color_loss + lambda_d * depth_loss
    
    return total_loss
```

**Parameters:**
*   $\lambda_D$: Depth loss weight. Empirically set to 1 [S4, unofficial].
*   $t_n, t_f$: Near and far clipping planes. $t_f$ is often treated as an opaque wall to ensure $\int h(t)dt = 1$ [S1].
*   Training Iterations: Typically $10,000 - 100,000$ [S3, unofficial].
*   Image Resolution: $400 \times 300$ in standard experiments [S1].

**Edge Cases and Pitfalls:**
*   **SfM Failure:** Optimization fails if SfM produces poor depth estimates or inaccurate depth measurements [S1].
*   **Gaussian Assumption:** The method assumes a Gaussian distribution models keypoint uncertainty, which may not hold for depth from non-SfM sources (e.g., noisy sensors) [S1].
*   **Domain Gap:** While DS-NeRF itself generalizes, combining it with other methods (like pixelNeRF) may suffer if there is a domain gap between pre-training data and test data [S1].
*   **MVSNeRF Compatibility:** Adding depth supervision did not improve MVSNeRF because it already possesses a strong geometry prior via PSV initialization [S1].

## Operating characteristics
*   **Time Complexity:** Training iterations are reduced by 2–3× compared to standard NeRF [S1]. Wall-clock time per iteration is comparable ($\sim 362.4$ ms vs $\sim 359.8$ ms on RTX A5000) [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the NeRF Real dataset with 5 input views, DS-NeRF achieves a test PSNR of 22.6 (vs 18.2 for NeRF) [S1]. It reaches target PSNR significantly faster, e.g., 13 hours faster in the 5-view case [S1].
*   **Strengths:** Accelerates training, improves quality with few views, compatible with other NeRF variants, and uses "free" SfM data [S1].
*   **Limitations:** Dependent on the quality of SfM keypoints; does not improve methods that already have strong geometric priors (like MVSNeRF) [S1].

## Background
DS-NeRF was introduced by Deng et al. in 2022 [S1]. It addresses the failure mode of NeRF when trained on insufficient input views, where the model fits incorrect geometries due to the lack of constraints enforcing empty space and opaque surfaces [S1]. The method leverages the fact that SfM pipelines (like COLMAP), which are standard prerequisites for NeRF training, already produce sparse 3D keypoints [S1].

## Sources
[S1] Depth-supervised NeRF: Fewer Views and Faster Training for Free, https://doi.org/10.1109/cvpr52688.2022.01254, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S4] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S5] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
