# GSGEN

Also called: 3D Point Cloud Guidance, G SGEN, GSGEN: A method introducing 3D SDS loss based on Point-E

## Overview
GSGEN is a text-to-3D generation method that represents 3D content as a set of 3D Gaussians and optimizes them progressively in two stages: geometry optimization and appearance refinement, using 2D and 3D diffusion priors [S1, S2]. It differs from NeRF-based methods (e.g., DreamFusion, Magic3D) by using explicit 3D Gaussians, which allows for the incorporation of direct 3D geometry priors and alleviates the Janus problem [S1]. It differs from DreamGaussian by focusing on incorporating 3D prior with more advanced representation, whereas DreamGaussian focuses on fast image-to-3D generation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A text prompt $y$ [S1].
*   **Output:** A set of 3D Gaussians $\{\theta\}$ representing the 3D asset, where $\theta$ includes positions $\mu_i$, covariances $\Sigma_i$, opacities $o_i$, and colors $c_i$ for each Gaussian $i$ [S1].

**Data Structures:**
The method uses the standard 3D Gaussian Splatting representation [S1]. Each Gaussian $i$ is defined by:
*   Position $\mu_i \in \mathbb{R}^3$ [S1].
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite) [S1].
*   Opacity $o_i \in [0, 1]$ [S1].
*   Color $c_i$ (typically represented by spherical harmonics coefficients) [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Use a text-to-point-cloud diffusion model (specifically Point-E [S1, S2]) to generate a coarse 3D point cloud from the text prompt $y$.
    *   Initialize the positions $\mu_i$ of the Gaussians with this coarse point cloud [S1, S2].
    *   The original adaptive control in Gaussian Splatting performs sub-optimally under SDS loss, necessitating the introduction of compactness-based densification [S1].

2.  **Stage 1: Geometry Optimization:**
    *   Optimize the Gaussians under the joint guidance of:
        1.  A 3D point cloud diffusion prior (from Point-E) via 3D SDS loss.
        2.  An ordinary 2D image prior (from StableDiffusion) via 2D SDS loss [S1, S2].
    *   This stage establishes a rough, 3D-consistent shape and corrects major shape deviations (e.g., asymmetric geometries like dogs or pandas) [S1].
    *   The gradient for Score Distillation Sampling (SDS) is calculated as [S1]:
        $$ \nabla_\theta L^{SDS} = \mathbb{E}_{\epsilon, t} w(t)(\epsilon_\phi (x_t ; y, t) - \epsilon) \frac{\partial x}{\partial \theta} $$
        where:
        *   $\epsilon$ is Gaussian noise [S1].
        *   $w(t)$ is a weighting function [S1].
        *   $\epsilon_\phi$ is the score estimation function from the diffusion model [S1].
        *   $x_t$ is the noisy image [S1].
        *   $y$ is the text embedding [S1].
        *   $t$ is the timestep [S1].
        *   $x = g(\theta)$ is the rendered image [S1].
    *   For the 3D SDS loss, the rendering function $g(\theta)$ produces the rendered image $x$, which is processed by the Point-E diffusion model to compute the 3D prior gradient [S1].

3.  **Stage 2: Appearance Refinement:**
    *   Optimize the Gaussians to enrich delicate details using only the 2D image prior (StableDiffusion) [S1, S2].
    *   Increase the number of Gaussians using **compactness-based densification** to enhance continuity and improve fidelity [S1].
    *   The original densification strategy splits Gaussians with an average view-space gradient larger than a threshold $T_{pos}$ [S1].
    *   Typical value for $T_{pos}$ in the original strategy: $0.0002$ [S1].
    *   A variant with larger $T_{pos} = 0.02$ avoids generating too many new Gaussians but may result in over-smoothed appearance [S1].
    *   GSGEN uses compactness-based densification as a supplement to the original strategy to avoid the dilemma of unstable gradients under SDS guidance [S1].

**Rendering Formula:**
The color $C(p)$ of a point $p$ on the image plane is rendered using point-based volume rendering [S1]:
$$ C(p) = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where:
*   $N$ is the set of Gaussians sorted by depth in the tile [S1].
*   $c_i$ is the color of the $i$-th Gaussian [S1].
*   $\alpha_i$ is the opacity of the $i$-th Gaussian at point $p$ [S1].

The opacity $\alpha_i$ is defined as [S1]:
$$ \alpha_i = o_i e^{-\frac{1}{2} (p-\mu_i)^T \Sigma_i^{-1} (p-\mu_i)} $$
where:
*   $o_i$ is the base opacity [S1].
*   $\mu_i$ is the position [S1].
*   $\Sigma_i$ is the covariance matrix [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def gsgen(text_prompt, num_geo_steps=1000, num_appear_steps=2000, batch_size=4):
    # 1. Initialization
    point_cloud = point_e_generate(text_prompt)
    gaussians = initialize_gaussians_from_point_cloud(point_cloud)
    
    # 2. Geometry Optimization Stage
    for step in range(num_geo_steps):
        cameras = sample_random_cameras(batch_size)
        rendered_images = render_gaussians(gaussians, cameras)
        
        loss_2d = calculate_2d_sds_loss(rendered_images, text_prompt, stable_diffusion_model)
        projected_pcs = project_to_point_cloud(gaussians, cameras)
        loss_3d = calculate_3d_sds_loss(projected_pcs, text_prompt, point_e_model)
        
        total_loss = loss_2d + loss_3d
        gaussians = update_gaussians(gaussians, total_loss)
        
    # 3. Appearance Refinement Stage
    for step in range(num_appear_steps):
        cameras = sample_random_cameras(batch_size)
        rendered_images = render_gaussians(gaussians, cameras)
        
        loss_2d = calculate_2d_sds_loss(rendered_images, text_prompt, stable_diffusion_model)
        gaussians = update_gaussians(gaussians, loss_2d)
        gaussians = densify_gaussians_compactness(gaussians)
        
    return gaussians

def point_e_generate(prompt):
    # Returns a numpy array of shape (N, 3) representing a coarse 3D point cloud
    return np.random.randn(1000, 3) * 2.0

def initialize_gaussians_from_point_cloud(pc):
    # Initializes Gaussian positions from pc, sets default covariances, opacities, colors
    positions = pc.copy()
    covariances = np.repeat(np.eye(3), len(positions), axis=0).reshape(-1, 3, 3)
    opacities = np.ones(len(positions)) * 0.5
    colors = np.random.rand(len(positions), 3)
    return {"positions": positions, "covariances": covariances, "opacities": opacities, "colors": colors}

def sample_random_cameras(batch_size):
    # Returns a list of camera extrinsics and intrinsics
    return [{"extrinsics": np.eye(4), "intrinsics": np.eye(3)} for _ in range(batch_size)]

def render_gaussians(gaussians, cameras):
    # Differentiable volume rendering returning image tensor and gradient hook
    # Computes C(p) = sum(c_i * alpha_i * prod(1 - alpha_j))
    return np.random.rand(4, 3, 256, 256)

def stable_diffusion_model():
    # Returns a callable that predicts noise epsilon_phi(x_t; y, t)
    return lambda x_t, y, t: np.random.randn(*x_t.shape)

def point_e_model():
    # Returns a callable that predicts noise for point cloud diffusion
    return lambda x_t, y, t: np.random.randn(*x_t.shape)

def calculate_2d_sds_loss(rendered_images, text_prompt, diffusion_model):
    loss = 0.0
    for image in rendered_images:
        t = sample_timestep()
        epsilon = sample_gaussian_noise(image.shape)
        x_t = add_noise(image, epsilon, t)
        predicted_epsilon = diffusion_model(x_t, text_prompt, t)
        w_t = 1.0 - t  # Example weighting function
        gradient = w_t * (predicted_epsilon - epsilon)
        # dx/dtheta is obtained via backprop through render_gaussians
        loss += np.sum(gradient * render_gradient(image))
    return loss / len(rendered_images)

def calculate_3d_sds_loss(projected_pcs, text_prompt, point_e_model):
    loss = 0.0
    for pc in projected_pcs:
        t = sample_timestep()
        epsilon = sample_gaussian_noise(pc.shape)
        pc_t = add_noise(pc, epsilon, t)
        predicted_epsilon = point_e_model(pc_t, text_prompt, t)
        w_t = 1.0 - t
        gradient = w_t * (predicted_epsilon - epsilon)
        loss += np.sum(gradient * point_cloud_gradient(pc))
    return loss / len(projected_pcs)

def update_gaussians(gaussians, loss):
    # Applies gradient descent to Gaussian parameters
    return gaussians

def sample_timestep():
    return np.random.uniform(0.0, 1.0)

def sample_gaussian_noise(shape):
    return np.random.randn(*shape)

def add_noise(x, epsilon, t):
    return np.sqrt(1 - t) * x + np.sqrt(t) * epsilon

def project_to_point_cloud(gaussians, cameras):
    # Projects 3D Gaussians to 2D/3D point cloud representation for 3D prior
    return [gaussians["positions"] for _ in cameras]

def render_gradient(image):
    # Placeholder for differentiable renderer gradient computation
    return np.zeros_like(image)

def point_cloud_gradient(pc):
    # Placeholder for point cloud diffusion gradient computation
    return np.zeros_like(pc)

def densify_gaussians_compactness(gaussians):
    # Compactness-based densification strategy
    positions = gaussians["positions"]
    covariances = gaussians["covariances"]
    # Check compactness metric and split/copy Gaussians if too spread
    new_positions = np.vstack([positions, positions + np.random.randn(*positions.shape) * 0.1])
    new_covariances = np.vstack([covariances, covariances])
    new_opacities = np.concatenate([gaussians["opacities"], gaussians["opacities"]])
    new_colors = np.vstack([gaussians["colors"], gaussians["colors"]])
    return {"positions": new_positions, "covariances": new_covariances, "opacities": new_opacities, "colors": new_colors}
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** GSGEN generates 3D assets with delicate details and accurate geometry, demonstrating superior performance in capturing high-frequency components [S1]. It effectively addresses asymmetric geometries (e.g., dog, panda) and collapsed structures better than methods without 3D priors [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Alleviates the Janus problem (inaccurate geometry/multi-face issue) common in NeRF-based methods [S1, S2].
    *   Generates highly detailed and 3D consistent assets [S1].
    *   Explicit 3D representation allows direct incorporation of 3D geometry priors [S1].
*   **Limitations:**
    *   Tends to generate unsatisfying results when the text prompt contains complex descriptions or complicated logic due to the limited language understanding ability of Point-E and the CLIP text encoder used in StableDiffusion [S1].
    *   The Janus problem is mitigated but not entirely eliminated; potential degenerations can still occur, especially when the textual prompt is extremely biased in the guidance diffusion models [S1].
    *   The original adaptive control in Gaussian Splatting performs sub-optimally under SDS loss, requiring the custom compactness-based densification [S1].
*   **When to use:** Text-to-3D generation requiring high fidelity and 3D consistency, especially for asymmetric objects [S1].
*   **When not to use:** Prompts with complex logic or detailed scene descriptions that exceed the language understanding capabilities of Point-E and StableDiffusion [S1].

## Background
*   **Origin:** GSGEN was proposed in the paper "Text-to-3D using Gaussian Splatting" [S1].
*   **History:** It builds upon the success of Score Distillation Sampling (SDS) introduced by DreamFusion [S1] and the efficiency of 3D Gaussian Splatting [S1]. It specifically addresses the limitations of implicit representations (NeRF) in integrating 3D priors [S1].
*   **Usage:** Used in GSGEN for text-to-3D generation [S1]. It utilizes StableDiffusion as the 2D image prior and Point-E as the 3D point cloud diffusion prior [S1, S2].

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
