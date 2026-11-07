# Depth-Supervised NeRF

Also called: DS-NeRF, Depth-Supervised NeRF (DS-NeRF) Training with Keypoint Supervision, Depth-Supervised NeRF Training, NeRF with Keypoint Depth Supervision

## Overview
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality with sparse input views by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing [S1]. Unlike methods relying on data-driven priors from other scenes or semantic consistency, DS-NeRF only requires the existence of 3D keypoints, making it flexible and not dependent on training data distribution assumptions [S1]. It builds on [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by enforcing that the ray termination distribution matches surface priors derived from SfM points, reducing overfitting and speeding up training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input images $I_j$, camera poses $P_j$ (estimated via SfM), sparse 3D keypoints $X$ and their visibility flags $X_j \subset X$ (estimated via SfM), and optionally dense depth maps for evaluation [S1].
*   **Outputs:** A trained NeRF model (MLP mapping $(x, d) \to (\sigma, c)$) and rendered novel views [S1].

**Data Structures:**
*   **Keypoints:** Set $X = \{x_1, x_2, \dots\} \subset \mathbb{R}^3$.
*   **Visibility:** For each camera $j$, a subset $X_j$ of keypoints visible from that camera.
*   **Depth Estimates:** For each keypoint $x_i \in X_j$, a scalar depth $D_{ij}$ and uncertainty $\hat{\sigma}_i$.
*   **Ray Termination Distribution:** $h(t) = T(t)\sigma(t)$, where $T(t) = \exp(-\int_0^t \sigma(s)ds)$ [S1].

**Parameters:**
*   $\lambda_D$: Hyper-parameter balancing color and depth supervision. Typical values not explicitly given, but standard NeRF losses are used [S1].
*   $t_n, t_f$: Near and far clipping bounds. $t_f$ is often treated as an opaque wall to ensure $\int_0^\infty \sigma(s)ds = \infty$ [S1].
*   Training iterations: $O(10k - 100k)$ [S2, unofficial].
*   Ray allocation: A portion of total training rays is dedicated to keypoint rays [S1].

**Steps:**

1.  **SfM Preprocessing:** Run an SfM solver (e.g., COLMAP) on the input images to estimate camera poses $P_j$ and sparse 3D keypoints $X$ with visibility flags [S1].
    *   *Note:* Ground truth calibrated poses can be used if available [S1].
2.  **Depth Estimation:** For each image $I_j$ with camera $P_j$, identify visible keypoints $X_j$. For each $x_i \in X_j$, compute the projected depth $D_{ij}$ by projecting $x_i$ with $P_j$ and taking the re-projected z-value [S1].
3.  **Uncertainty Calculation:** Compute the depth uncertainty $\hat{\sigma}_i$ for each keypoint $x_i$ as the average reprojection error across all views where the keypoint was detected [S1].
4.  **Ray Generation:**
    *   For a detected keypoint $x_i$ in image $I_j$, calculate the ray origin in local camera coordinates from the image coordinates of $x_i$, setting the camera axis coordinate to -1 [S1].
    *   Shoot this ray into the scene using the standard NeRF sampling procedure (sample points $t_k$ along the ray between $t_n$ and $t_f$) [S1].
    *   *Optimization:* To train with both color and depth supervision in a single ray marching step, exploit the continuity of image coordinates to interpolate RGB supervision for rays corresponding to detected keypoints [S1].
5.  **Rendering:** Render the color $\hat{C}$ and the ray termination distribution $h(t)$ for the generated rays using the current NeRF model [S1].
    *   $h(t) = \exp[-\int_0^t \sigma(s)ds]\sigma(t)$ [S1].
    *   $\sigma(s)$ must be real-valued and non-negative (e.g., ReLU or softplus) [S1].
6.  **Loss Computation:**
    *   **Color Loss:** Standard MSE loss between rendered color $\hat{C}$ and ground truth color $C$ [S1].
        $$L_{Color} = \mathbb{E}_{r \in R(P)} \|\hat{C}(r) - C(r)\|^2$$
    *   **Depth Loss:** Minimize the KL divergence between the rendered ray distribution $h_{ij}(t)$ and the noisy depth distribution $N(D_{ij}, \hat{\sigma}_i)$ [S1].
        $$L_{Depth} = \mathbb{E}_{x_i \in X_j} \left[ -\log h(t) \exp \left( -\frac{(t - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \right] dt$$
        Approximated discretely:
        $$L_{Depth} \approx \mathbb{E}_{x_i \in X_j} \sum_k -\log h_k \exp \left( -\frac{(t_k - D_{ij})^2}{2\hat{\sigma}_i^2} \right) \Delta t_k$$
        where $h_k = h(t_k)$, $t_k$ are sample points, and $\Delta t_k$ is the step size [S1].
    *   **Total Loss:**
        $$L = L_{Color} + \lambda_D L_{Depth}$$
        [S1]
7.  **Optimization:** Update NeRF parameters using gradient descent on $L$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_depth_loss(rendered_h, keypoint_depth, keypoint_uncertainty, dt):
    """
    Compute the depth supervision loss using KL divergence approximation.
    
    Args:
        rendered_h: Array of ray termination distribution values h(t_k) for the ray.
        keypoint_depth: Scalar D_ij, the COLMAP-estimated depth.
        keypoint_uncertainty: Scalar sigma_hat_i, the average reprojection error.
        dt: Array of step sizes Delta t_k.
        
    Returns:
        Scalar loss value.
    """
    # Gaussian term: exp(-(t - D)^2 / (2 * sigma^2))
    # Note: rendered_h corresponds to h(t_k) at sample points t_k.
    # We need t_k to compute the Gaussian. Assuming t_k is available or passed.
    # For this implementation, we assume t_k is passed or derived from dt and origins.
    # Here we assume 'rendered_h' is paired with 't_samples' in the caller context.
    # To keep this function self-contained as per rules, we require t_samples.
    pass

def train_ds_nerf(images, poses, keypoints, visibility, model, num_iterations, lambda_d, dt_samples):
    """
    Train DS-NeRF.
    
    Args:
        images: List of input images.
        poses: Camera poses P_j.
        keypoints: Set of 3D keypoints X.
        visibility: Visibility flags X_j for each camera.
        model: NeRF model (MLP).
        num_iterations: Number of training steps.
        lambda_d: Weight for depth loss.
        dt_samples: Array of sample points t_k along the ray.
    """
    for iteration in range(num_iterations):
        # 1. Select a batch of rays
        # Allocate a portion of rays to keypoint rays
        keypoint_rays = []
        color_rays = []
        
        for j, (image, pose) in enumerate(zip(images, poses)):
            # Get visible keypoints for this camera
            visible_kps = [x for x in keypoints if x in visibility[j]]
            
            for kp in visible_kps:
                # 2. Compute depth and uncertainty
                # Project kp with pose to get depth D_ij
                # In local camera coordinates, ray direction is determined by kp's image coords
                # and -1 in camera axis.
                ray_origin, ray_direction = get_ray_from_keypoint(kp, pose)
                
                # Compute D_ij: re-projected z-value
                D_ij = project_depth(kp, pose)
                
                # Compute sigma_hat_i: average reprojection error
                sigma_hat_i = compute_reprojection_error(kp, visibility)
                
                # 3. Render ray
                # Use standard NeRF sampling procedure
                t_samples = dt_samples # Pre-defined or sampled
                sigma, color = model(ray_origin, ray_direction, t_samples)
                
                # Compute transmittance T(t)
                # T(t) = exp(-int_0^t sigma(s) ds)
                # Discrete approximation:
                # T_k = exp(-sum_{m=0}^{k-1} sigma_m * dt_m)
                # Assuming dt is uniform or passed
                dt = np.diff(t_samples) if len(t_samples) > 1 else np.array([0.1])
                # Pad dt to match t_samples length for convenience, or handle carefully
                # Standard NeRF rendering:
                # weights_k = T_k * sigma_k * dt_k
                # T_0 = 1.0
                # T_{k+1} = T_k * exp(-sigma_k * dt_k)
                
                T = np.ones_like(sigma)
                for k in range(len(sigma) - 1):
                    T[k+1] = T[k] * np.exp(-sigma[k] * dt[k])
                
                # Ray termination distribution h(t) = T(t) * sigma(t)
                h = T * sigma
                
                # 4. Compute Losses
                # Color Loss: Interpolate RGB supervision for keypoint rays
                # Since image coordinates are continuous, interpolate from image
                rgb_supervision = interpolate_color(image, kp.image_coords)
                color_loss = np.mean((color - rgb_supervision)**2)
                
                # Depth Loss: KL divergence approximation
                # L_Depth = -sum(h_k * exp(-(t_k - D_ij)^2 / (2 * sigma_hat_i^2)) * dt_k)
                gaussian_term = np.exp(-((t_samples[:-1] - D_ij)**2) / (2 * sigma_hat_i**2))
                # Note: h corresponds to h(t_k). The integral is approx sum(h_k * gaussian_k * dt_k)
                # But the formula in source is: -log h(t) * exp(...) dt ?
                # Source says: L_Depth = E[ -log h(t) exp(...) ] dt
                # Wait, source formula: L_Depth = E [ -log h(t) exp( -(t-D)^2 / 2sigma^2 ) ] dt
                # This looks like cross-entropy or similar.
                # Let's re-read carefully: "minimize the KL divergence between ... h(t) and N(D, sigma)"
                # KL(N || h) = int N(t) log(N(t)/h(t)) dt = int N(t) log N(t) dt - int N(t) log h(t) dt
                # The constant term int N(t) log N(t) dt is dropped.
                # So we minimize - int N(t) log h(t) dt.
                # N(t) ~ exp(-(t-D)^2 / 2sigma^2).
                # So L_Depth = - sum_k [ exp(-(t_k-D)^2 / 2sigma^2) * log(h_k) * dt_k ]
                
                depth_loss = -np.sum(gaussian_term * np.log(h + 1e-8) * dt)
                
                keypoint_rays.append({
                    'color_loss': color_loss,
                    'depth_loss': depth_loss,
                    'h': h,
                    't': t_samples[:-1] # samples used for h
                })
                
            # Add standard color rays
            # ... (standard NeRF ray generation)
            
        # 5. Aggregate and Update
        total_color_loss = sum(r['color_loss'] for r in keypoint_rays)
        total_depth_loss = sum(r['depth_loss'] for r in keypoint_rays)
        
        loss = total_color_loss + lambda_d * total_depth_loss
        
        # Backpropagate and update model
        update_model(model, loss)

def get_ray_from_keypoint(kp, pose):
    # Convert 3D keypoint to 2D image coordinates
    # Then construct ray origin and direction in world/camera space
    # This is a standard camera projection inverse
    pass

def project_depth(kp, pose):
    # Project 3D kp with pose to get depth
    # Return z-value in camera frame
    pass

def compute_reprojection_error(kp, visibility):
    # Calculate average reprojection error across all views where kp is visible
    pass
```

**Subtopics:**
*   **Guided Optimization of NeRF:** Uses guided optimization techniques. [Link](7496-guided-optimization-of-nerf.md).
*   **Single-view NeRF Training with Depth Map:** Uses dense depth maps from single views. [Link](8679-single-view-nerf-training-with-depth-map.md).

## Operating characteristics
*   **Time Complexity:** DS-NeRF accelerates model training by 2-3x compared to standard NeRF [S1]. On a single RTX A5000 GPU, a training iteration takes ~362.4 ms for DS-NeRF vs ~359.8 ms for standard NeRF [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   On NeRF Real with 5 input views, DS-NeRF achieves NeRF's peak test PSNR approximately 13 hours faster [S1].
    *   DS-NeRF reaches target test PSNR in 2-3x fewer training iterations [S1].
    *   PSNR improvements with fewer views: 13.5 to 20.2 [S1].
    *   With 5 input views and varying keypoint sparseness: 21.5 (20% keypoints), 22.2 (50% keypoints), 22.6 (100% keypoints) vs NeRF baseline 18.2 [S1].
    *   Average keypoints per training view: 1615 (2 views), 2172 (5 views), 2621 (10 views) [S1].
*   **Scaling:** Performance gains are magnified at lower view counts [S1].
*   **Strengths:**
    *   Requires no additional training or category-specific priors [S1].
    *   Compatible with other NeRF methods (e.g., pixelNeRF, IBRNet) [S1].
    *   Can use depth from other sources (scanned depth sensors, RGB-D) [S1].
*   **Limitations:**
    *   Optimization fails if SfM produces poor depth estimates or inaccurate depth measurements [S1].
    *   Assumes Gaussian distribution for keypoint uncertainty, which may not hold for non-SfM sources [S1].
    *   Dense MVS depth is used only for evaluation, not training [S1].
    *   MVSNeRF did not improve with DS-NeRF loss due to existing geometry prior [S1].
    *   Domain gap: Methods pre-trained on DTU struggle on NeRF Real data [S1].

## Background
*   **Origin:** Introduced in "Depth-supervised NeRF: Fewer Views and Faster Training for Free" (CVPR 2022) [S1].
*   **History:** Developed to address the failure mode of NeRF fitting incorrect geometries with insufficient input views by leveraging "free" depth supervision from SfM keypoints [S1].
*   **Usage:** Applied to LLFF, NeRF Real, Redwood-3dscan, and DTU datasets [S1].

## Sources
[S1] Depth-supervised NeRF: Fewer Views and Faster Training for Free, https://doi.org/10.1109/cvpr52688.2022.01254, fetched 2026-10-26
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
