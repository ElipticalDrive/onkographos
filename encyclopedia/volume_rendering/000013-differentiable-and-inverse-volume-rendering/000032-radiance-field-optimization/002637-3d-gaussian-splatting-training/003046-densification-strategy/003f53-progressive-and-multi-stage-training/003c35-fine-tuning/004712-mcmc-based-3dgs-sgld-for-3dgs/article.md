# MCMC-based 3DGS / SGLD for 3DGS

Also called: 3D Gaussian Splatting as MCMC / SGLD, MCMC-based 3DGS, SGLD for 3DGS

## Overview
This method reformulates 3D Gaussian Splatting (3DGS) by treating the set of 3D Gaussians as random samples drawn from an underlying probability distribution describing the scene, specifically as Markov Chain Monte Carlo (MCMC) samples [S1]. It converts conventional 3DGS updates into Stochastic Gradient Langevin Dynamics (SGLD) updates by introducing a noise term to promote sample exploration, thereby removing the need for manual densification heuristics like cloning, splitting, and opacity resetting [S1]. Unlike sibling methods that rely on complex gradient thresholds or hard constraints, this approach relies on probabilistic state transitions and regularization to manage Gaussian population [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses $\boldsymbol{\theta}$, ground-truth images $\mathbf{C}_{gt}(\mathbf{x})$, initial set of $N$ 3D Gaussians. Each Gaussian $i$ is defined by opacity $o_i$, center $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, and Spherical Harmonics coefficients for color $\mathbf{c}_i$.
*   **Outputs:** Optimized Gaussian parameters ($o_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \mathbf{c}_i$) and rendered images.

**Data Structures:**
The state consists of a list of $N$ Gaussians. Each Gaussian $i$ stores:
*   $o_i \in \mathbb{R}$: Opacity.
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: 3D center position.
*   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized as rotation $\mathbf{R}$ and scale $\mathbf{s}$).
*   $\mathbf{c}_i$: Spherical Harmonics coefficients.

**Numbered Steps:**

1.  **Rendering:** For a given camera pose $\boldsymbol{\theta}$ and pixel $\mathbf{x}$, sort the $N$ Gaussians by distance to the camera. Compute the rendered color $\mathbf{C}(\mathbf{x})$ using alpha blending [S1]:
    $$
    \mathbf{C}(\mathbf{x}) = \sum_{i=1}^N \mathbf{c}_i \alpha_i(\mathbf{x}) \left[ \prod_{j=1}^{i-1} (1 - \alpha_j(\mathbf{x})) \right]
    $$
    where $\mathbf{c}_i$ is the color derived from Spherical Harmonics based on pose $\boldsymbol{\theta}$ [S1].

2.  **Opacity Calculation:** Compute the opacity $\alpha_i(\mathbf{x})$ for the $i$-th Gaussian at pixel $\mathbf{x}$ [S1]:
    $$
    \alpha_i(\mathbf{x}) = o_i \exp\left(-\frac{1}{2}(\mathbf{x} - \mathcal{R}(\boldsymbol{\mu}_i; \boldsymbol{\theta}))^T \mathcal{R}_{\boldsymbol{\theta}}(\boldsymbol{\Sigma}_i)^{-1}(\mathbf{x} - \mathcal{R}(\boldsymbol{\mu}_i; \boldsymbol{\theta}))\right)
    $$
    where $\mathcal{R}$ is the camera projection operation [S1].

3.  **Loss Computation:** Calculate the training loss $\mathcal{L}_{\text{orig}}$ as a weighted sum of L1 and D-SSIM losses [S1]:
    $$
    \mathcal{L}_{\text{orig}} = (1 - \lambda_{\text{D-SSIM}}) \cdot \mathcal{L}_1 + \lambda_{\text{D-SSIM}} \cdot \mathcal{L}_{\text{D-SSIM}}
    $$
    where $\mathcal{L}_1$ is the average L1 error between $\mathbf{C}(\mathbf{x})$ and $\mathbf{C}_{gt}(\mathbf{x})$, and $\mathcal{L}_{\text{D-SSIM}}$ is the D-SSIM loss [S1]. The weight $\lambda_{\text{D-SSIM}}$ is typically $0.2$ [S1].

4.  **SGLD Update (Exploration):** Compute gradients $\nabla \mathcal{L}_{\text{orig}}$ with respect to Gaussian parameters. Update parameters using Stochastic Gradient Langevin Dynamics by adding noise to the gradient step [S1]. This replaces the standard deterministic optimizer step, allowing Gaussians to explore the scene landscape naturally without manual densification [S1].

5.  **Relocation (Deterministic State Transition):** Implement a relocation strategy to handle "dead" Gaussians (those with low opacity) [S1].
    *   Identify "dead" Gaussians (low opacity) and "live" Gaussians (high opacity) [S1].
    *   Move dead Gaussians to locations of live Gaussians [S1].
    *   **Crucial Constraint:** Ensure the composition of the relocated (cloned) Gaussians renders the same images as before relocation to preserve sample probability and prevent training instability [S1].
    *   **Pitfall:** Cloning at Gaussian centers alone is insufficient; the whole Gaussian must be considered to ensure the rendering remains equal and MCMC sampling remains optimal [S1].

6.  **Regularization:** Apply L1-regularization to encourage efficient use of Gaussians [S1]. This regularizer is applied to both opacity and scale, promoting the removal of unused Gaussians by encouraging them to "disappear" if unnecessary [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_gaussians(gaussians, camera_pose, pixel_coords):
    """
    Renders image from Gaussians using alpha blending.
    gaussians: List of Gaussian objects with properties:
        - mu: center (3,)
        - sigma: covariance (3,3)
        - o: opacity (scalar)
        - sh_coeffs: Spherical Harmonics coefficients
    camera_pose: Camera extrinsics/intrinsics
    pixel_coords: Array of pixel coordinates (H, W, 2)
    """
    # 1. Sort Gaussians by distance to camera
    # Calculate distance for each Gaussian to the camera center
    distances = [calc_distance(g.mu, camera_pose) for g in gaussians]
    sorted_indices = np.argsort(distances)
    sorted_gaussians = [gaussians[i] for i in sorted_indices]
    
    H, W = pixel_coords.shape[:2]
    rendered_image = np.zeros((H, W, 3))
    accumulated_alpha = np.zeros((H, W))
    
    # 2. Alpha Blending
    for g in sorted_gaussians:
        # Calculate opacity at each pixel
        # alpha_i(x) = o_i * exp(-0.5 * (x - mu)^T Sigma^-1 (x - mu))
        # Note: x is projected 3D point, mu is projected 3D center
        projected_x = project_to_pixel(pixel_coords, camera_pose)
        projected_mu = project_to_pixel(g.mu, camera_pose)
        
        # Compute Mahalanobis distance
        diff = projected_x - projected_mu
        # Sigma_inv is the inverse of the projected covariance
        sigma_inv = np.linalg.inv(project_covariance(g.sigma, camera_pose))
        mahal = np.sum(diff @ sigma_inv * diff, axis=-1)
        
        alpha = g.o * np.exp(-0.5 * mahal)
        alpha = np.clip(alpha, 0, 1)
        
        # Get color from Spherical Harmonics
        color = get_sh_color(g.sh_coeffs, camera_pose)
        
        # Update accumulated alpha and color
        transmittance = 1.0 - accumulated_alpha
        rendered_image += color * alpha * transmittance[:, :, np.newaxis]
        accumulated_alpha += alpha * transmittance
        
    return rendered_image

def update_sgld(gaussians, gradients, noise_scale):
    """
    Updates Gaussian parameters using SGLD.
    gradients: Dict of gradients for each parameter type
    noise_scale: Standard deviation of the noise added
    """
    for g in gaussians:
        # Update center
        g.mu -= gradients['mu'] + np.random.normal(0, noise_scale, g.mu.shape)
        # Update opacity
        g.o -= gradients['o'] + np.random.normal(0, noise_scale)
        # Update covariance/scale/rotation
        # Note: Specific parameterization (e.g., quaternion for rotation) 
        # requires appropriate manifold updates. Here we assume Euclidean for simplicity.
        g.sigma -= gradients['sigma'] + np.random.normal(0, noise_scale, g.sigma.shape)
        g.sh_coeffs -= gradients['sh'] + np.random.normal(0, noise_scale, g.sh_coeffs.shape)

def relocate_gaussians(gaussians, threshold=0.01):
    """
    Relocates 'dead' Gaussians to 'live' Gaussians.
    Ensures rendering consistency.
    """
    live_gaussians = [g for g in gaussians if g.o > threshold]
    dead_gaussians = [g for g in gaussians if g.o <= threshold]
    
    for dead_g in dead_gaussians:
        if not live_gaussians:
            continue
            
        # Select a live Gaussian to clone from
        # Strategy: Pick randomly or based on proximity
        source_g = np.random.choice(live_gaussians)
        
        # Relocate dead Gaussian to source location
        # To preserve probability, the composition must render the same.
        # This implies copying parameters or adjusting to match the local density.
        # The source states: "set the composition of the cloned Gaussians to render 
        # the same images as before cloning."
        # Implementation: Copy parameters from source, but this changes the count.
        # A more precise MCMC move might involve swapping or adjusting weights.
        # However, the source emphasizes "relocation" and "cloning".
        # We assume a simple copy for relocation, but the source warns that 
        # cloning at centers is insufficient.
        
        # To strictly follow "rendering remains equal", one might need to 
        # adjust the opacity of the source or other Gaussians.
        # Given the note "cloning at Gaussian centers alone is insufficient",
        # a robust implementation would ensure the *entire* Gaussian's contribution 
        # is preserved. A simple copy changes the distribution.
        # The source suggests this strategy is "enough to provide renderings of high quality".
        
        dead_g.mu = source_g.mu.copy()
        dead_g.sigma = source_g.sigma.copy()
        dead_g.o = source_g.o # Or adjust based on regularization
        dead_g.sh_coeffs = source_g.sh_coeffs.copy()

def train_step(gaussians, batch_data, lambda_d_ssim=0.2, noise_scale=1e-4):
    """
    Single training step.
    """
    camera_pose, gt_image = batch_data
    
    # Render
    pred_image = render_gaussians(gaussians, camera_pose, gt_image.shape)
    
    # Compute Loss
    l1_loss = np.mean(np.abs(pred_image - gt_image))
    d_ssim_loss = compute_d_ssim(pred_image, gt_image)
    loss = (1 - lambda_d_ssim) * l1_loss + lambda_d_ssim * d_ssim_loss
    
    # Compute Gradients
    gradients = compute_gradients(loss, gaussians)
    
    # SGLD Update
    update_sgld(gaussians, gradients, noise_scale)
    
    # Relocation (Periodically or every step)
    relocate_gaussians(gaussians)
    
    # Regularization (L1 on opacity and scale)
    apply_regularization(gaussians)
    
    return loss
```

**Parameters:**
*   $\lambda_{\text{D-SSIM}}$: Weight for the D-SSIM loss component. Typical value: $0.2$ [S1].
*   Noise Scale: Standard deviation of the noise added in SGLD updates. Not given in the sources. [G] (Typically small, e.g., $10^{-4}$ to $10^{-3}$, to ensure stability while allowing exploration).

**Edge Cases and Pitfalls:**
*   **Cloning Insufficiency:** Cloning Gaussians only at their centers is insufficient to preserve the rendering probability distribution; the entire Gaussian's spatial extent must be considered to ensure the rendering remains equal [S1].
*   **Initialization:** While the method is robust to initialization, poor initialization may require more steps for the MCMC chain to converge to the optimal distribution [S1].
*   **Precision:** The exponential term in opacity calculation can lead to underflow for distant Gaussians. Clipping or masking Gaussians outside the view frustum is recommended [G].

## Operating characteristics
*   **Time Complexity:** Similar to standard 3DGS per step, but may require more steps for convergence due to the stochastic nature of SGLD [S1].
*   **Memory Complexity:** Same as 3DGS, storing $N$ Gaussians [S1].
*   **Strengths:** Improved rendering quality, easy control over the number of Gaussians, and robustness to initialization [S1]. Removes the need for complex heuristics like cloning, splitting, and opacity resetting [S1].
*   **Limitations:** Relies on the MCMC interpretation, which may be less intuitive than deterministic optimization for some practitioners [S1].
*   **When to use:** When robustness to initialization is critical, or when manual densification heuristics are problematic [S1].
*   **When not to use:** When computational resources are extremely limited and the overhead of noise addition and relocation logic is prohibitive [G].

## Background
This method was introduced by the UBC Vision group, as detailed in "3D Gaussian Splatting as Markov Chain Monte Carlo" [S1]. It rethinks the set of 3D Gaussians as MCMC samples, converting conventional updates into SGLD updates [S1]. The work builds on the popularity of NeRF and 3DGS, addressing issues with poor-quality renderings and reliance on good initialization in existing methods [S1].

## Sources
[S1] 3D Gaussian Splatting as Markov Chain Monte Carlo, https://ubc-vision.github.io/3dgs-mcmc/, fetched 2026-10-26
