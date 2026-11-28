# GSGEN Appearance Refinement


## Overview
GSGEN Appearance Refinement is an iterative optimization stage that improves fine-grained visual details in text-to-3D generation while preserving the fundamental geometry established during the preceding geometry optimization phase [S1]. It addresses the limitations of standard gradient-based densification by introducing a compactness-based strategy to fill geometric holes and employs specific regularization losses to prevent overfitting and maintain 3D consistency [S1]. Unlike sibling methods that rely primarily on positional gradients or opacity thresholds, GSGEN explicitly incorporates a 3D Stable Diffusion Score Distillation Sampling (SDS) loss and a mean position penalty to ensure structural integrity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}$ initialized from the geometry optimization stage, where each Gaussian $g_i$ has position $p_i$, opacity $o_i$, scale, rotation, and spherical harmonic coefficients [S1]. A pre-trained 2D diffusion model (Stable Diffusion) [S1].
*   **Output:** Refined set of 3D Gaussians with improved appearance and denser, more complete geometry [S1].

**Data Structures:**
*   **KD-Tree:** Stores Gaussian positions for efficient nearest-neighbor queries [S1].
*   **Gaussian Attributes:** Position $p_i \in \mathbb{R}^3$, Opacity $o_i \in \mathbb{R}$, radius $r_i$, scale, rotation, SH coefficients [S1].

**Parameters:**
*   $\lambda_{SDS} = 0.1$: Weight for the 3D SDS loss [S1].
*   $\lambda_{mean} = 1.0$: Weight for the mean position regularization loss [S1].
*   $\lambda_{opacity} = 100.0$: Weight for the opacity regularization loss [S1].
*   $\alpha_{min}$: Threshold for pruning Gaussians with low opacity [S1].
*   $K$: Number of nearest neighbors considered for compactness-based densification [S1].
*   $N_{views}$: Number of views sampled per iteration using stratified sampling on azimuth [S1].

**Steps:**

1.  **Loss Computation:**
    For each iteration, compute the total refinement loss gradient $\nabla_\theta L_{refine}$ for each Gaussian parameter $\theta$:
    $$
    \nabla_\theta L_{refine} = \lambda_{SDS} \mathbb{E}_{\epsilon_I, t} \left[ w_I(t) (\epsilon_\phi(x_t; y, t) - \epsilon_I) \frac{\partial x}{\partial \theta} \right] + \lambda_{mean} \sum_i \nabla_\theta ||p_i|| + \lambda_{opacity} \sum_i \nabla_\theta \text{sg}(||p_i||) \cdot o_i
    $$
    Where:
    *   $\epsilon_\phi(x_t; y, t)$ is the noise prediction from the diffusion model $\phi$ given noisy image $x_t$, text prompt $y$, and timestep $t$ [S1].
    *   $\epsilon_I$ is the ground truth noise sampled from a standard normal distribution [S1].
    *   $w_I(t)$ is the weighting function for the SDS loss [S1].
    *   $\frac{\partial x}{\partial \theta}$ is the gradient of the rendered image $x$ with respect to the Gaussian parameters $\theta$ [S1].
    *   $||p_i||$ is the Euclidean norm of the position of the $i$-th Gaussian [S1].
    *   $\text{sg}(\cdot)$ denotes the stop-gradient operation [S1].
    *   $o_i$ is the opacity of the $i$-th Gaussian [S1].

2.  **Compactness-Based Densification:**
    *   For each Gaussian $g_i$, query its $K$ nearest neighbors using the KD-Tree [S1].
    *   For each neighbor $g_j$, if the distance $d(p_i, p_j)$ between the centers is smaller than the sum of their radii ($r_i + r_j$), perform densification [S1]:
        *   Add a new Gaussian positioned between $g_i$ and $g_j$.
        *   The radius of the new Gaussian is equal to the residual distance ($r_i + r_j - d(p_i, p_j)$) [S1].
    *   This step aims to "fill the holes" and create a more complete geometry structure [S1].

3.  **Opacity Regularization and Pruning:**
    *   Add an extra loss term to regularize opacity, proportional to the distance of the Gaussian to the center of the scene [S1].
    *   Periodically (every $N$ iterations, consistent with parent [Parent]), remove Gaussians with opacity $o_i < \alpha_{min}$ [S1].

4.  **Geometric Consistency Penalty:**
    *   The term $\lambda_{mean} \sum_i \nabla_\theta ||p_i||$ penalizes Gaussians that deviate significantly from their positions obtained during the preceding geometry optimization stage [S1]. This ensures that the refinement process does not distort the fundamental 3D structure [S1].

5.  **Optimization Step:**
    *   Update Gaussian parameters $\theta$ using the computed gradients $\nabla_\theta L_{refine}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gsgen_appearance_refinement(gaussians, diffusion_model, text_prompt, num_iterations, K, alpha_min, lambda_sds, lambda_mean, lambda_opacity, prune_interval=100):
    """
    GSGEN Appearance Refinement stage.
    
    Args:
        gaussians: List of dicts with keys: pos (3,), opacity (float), radius (float), params (np.array)
        diffusion_model: Object with method predict_noise(x_t, prompt, t) and num_timesteps
        text_prompt: Text prompt for generation
        num_iterations: Number of refinement iterations
        K: Number of nearest neighbors for compactness-based densification
        alpha_min: Opacity threshold for pruning
        lambda_sds: Weight for SDS loss
        lambda_mean: Weight for mean position regularization
        lambda_opacity: Weight for opacity regularization
        prune_interval: Iterations between pruning steps
        
    Returns:
        Refined gaussians list
    """
    positions = np.array([g['pos'] for g in gaussians])
    radii = np.array([g['radius'] for g in gaussians])
    
    for iteration in range(num_iterations):
        gradients = {g: np.zeros_like(g['params']) for g in gaussians}
        
        # Stratified sampling on azimuth for camera positions [S1]
        views = np.linspace(0, 2 * np.pi, len(gaussians), endpoint=False)
        
        for g in gaussians:
            # Render image and compute gradients w.r.t Gaussian parameters [S1]
            rendered_img = render_image(gaussians, views)
            grad_render = compute_renderer_gradient(rendered_img, g['params'])
            
            # 1. 3D SDS Loss term [S1]
            epsilon_I = np.random.randn(*rendered_img.shape)
            t = np.random.randint(0, diffusion_model.num_timesteps)
            x_t = add_noise(rendered_img, t)
            epsilon_pred = diffusion_model.predict_noise(x_t, text_prompt, t)
            w_t = compute_sds_weight(t)
            grad_sds = w_t * (epsilon_pred - epsilon_I) * grad_render
            gradients[g] += lambda_sds * grad_sds
            
            # 2. Mean position regularization: ||p_i|| [S1]
            pos_norm = np.linalg.norm(g['pos'])
            grad_mean = lambda_mean * (g['pos'] / pos_norm) if pos_norm > 1e-8 else np.zeros(3)
            gradients[g] += grad_mean
            
            # 3. Opacity regularization: sg(||p_i||) * o_i [S1]
            # sg stops gradient through position, so this only affects opacity parameter
            opacity_reg = np.abs(pos_norm) * g['opacity']
            grad_opacity = lambda_opacity * np.ones_like(g['params']) # Gradient w.r.t opacity
            gradients[g] += grad_opacity
            
        # Update parameters
        for g in gaussians:
            g['params'] -= gradients[g]
            
        # Compactness-based densification [S1]
        new_gaussians = []
        for i, g in enumerate(gaussians):
            # Find K nearest neighbors
            dists = np.linalg.norm(positions - g['pos'], axis=1)
            dists[i] = np.inf # Exclude self
            neighbor_indices = np.argsort(dists)[:K]
            for j in neighbor_indices:
                neighbor = gaussians[j]
                dist = np.linalg.norm(g['pos'] - neighbor['pos'])
                sum_radii = g['radius'] + neighbor['radius']
                if dist < sum_radii:
                    residual_radius = sum_radii - dist
                    mid_pos = (g['pos'] + neighbor['pos']) / 2.0
                    new_g = {'pos': mid_pos, 'opacity': 0.5, 'radius': residual_radius, 'params': g['params'].copy()}
                    new_gaussians.append(new_g)
        gaussians.extend(new_gaussians)
        positions = np.array([g['pos'] for g in gaussians])
        radii = np.array([g['radius'] for g in gaussians])
        
        # Pruning [S1]
        if iteration % prune_interval == 0:
            gaussians = [g for g in gaussians if g['opacity'] >= alpha_min]
            
    return gaussians

def render_image(gaussians, views):
    # Differentiable rasterization of 3D Gaussians to 2D images [G]
    return np.zeros((len(views), 256, 256, 3))

def compute_renderer_gradient(rendered_img, params):
    # Backpropagation through the differentiable rasterizer [G]
    return np.zeros_like(params)

def add_noise(img, t):
    # Linear noise schedule addition [G]
    return img + 0.1 * np.random.randn(*img.shape)

def compute_sds_weight(t):
    # Standard SDS weighting function [G]
    return 1.0

def sample_views_stratified_azimuth(n):
    # Returns evenly spaced azimuth angles [S1]
    return np.linspace(0, 2 * np.pi, n, endpoint=False)
```

**Edge Cases and Pitfalls:**
*   **Blurry Appearance:** A large threshold for compactness-based densification leads to a blurry appearance, which the method mitigates by supplementing positional gradient-based splitting [S1].
*   **Geometry Distortion:** Without the mean position regularization, the SDS loss might cause the geometry to drift or distort, exacerbating the Janus problem [S1].
*   **Opacity Instability:** Low opacity Gaussians can cause noise in the rendering; pruning them periodically is essential for stability [S1].

## Subtopics

### [GAINS Stage II: Material Properties](17159-gains-stage-ii-material-properties.md)
Joint optimization of material properties M = {(aj, rj, mj)} and environment lighting L over 7000 additional iterations, using re-rendering loss Lcolor augmented with segmentation, intrinsic image decomposition, and diffusion priors to resolve ill-posedness and overfitting.

## Operating characteristics
*   **Time Complexity:** The compactness-based densification step requires $O(N \log N)$ for KD-Tree construction and $O(N \cdot K)$ for neighbor queries per iteration, where $N$ is the number of Gaussians and $K$ is the number of neighbors [S1]. The SDS loss computation involves rendering and backpropagation through the diffusion model, which is computationally expensive [S1].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians, which can increase significantly due to compactness-based densification [S1].
*   **Performance:** GSGEN produces more complete geometry structures and finer details compared to methods relying solely on gradient-based densification [S1]. It effectively mitigates the Janus problem [S1].
*   **Strengths:** Preserves geometry while improving appearance; fills holes in the Gaussian representation; reduces overfitting to specific views [S1].
*   **Limitations:** Computationally intensive due to the diffusion model guidance and densification steps; requires careful tuning of hyperparameters ($\lambda_{SDS}, \lambda_{mean}, \lambda_{opacity}$) [S1].
*   **When to use:** When generating 3D assets from text prompts where geometric consistency and fine detail are critical, and the initial geometry optimization has produced a sparse or incomplete representation [S1].

## Background
GSGEN (Gaussian Splatting for Generation) was introduced in the paper "Text-to-3D using Gaussian Splatting" [S1]. The appearance refinement stage was developed to address the challenges of overfitting and geometric inconsistency in text-to-3D generation methods that use 3D Gaussian Splatting as the underlying representation [S1]. The method integrates a 3D SDS loss with a pre-trained point cloud diffusion model to shape a 3D-consistent geometry, mitigating the Janus problem and generating more sensible geometry [S1].

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
