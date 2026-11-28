# Nerfbusters

Also called: Density Score Distillation Sampling (DSDS), Nerfbusters method

## Overview
Nerfbusters is a 3D diffusion-based regularization method that improves scene geometry and removes ghostly artifacts (floaters) from Neural Radiance Fields (NeRFs) trained on casually captured images [S2]. It distinguishes itself from sibling methods like DreamFusion by using a **local 3D geometric prior** learned from synthetic surface patches rather than a global scene prior or 2.5D RGB-D priors [S2]. This local approach is category-agnostic, simpler to model, and explicitly 3D-consistent, making it suitable for arbitrary scenes without requiring complex conditioning [S2]. The core innovation is the **Density Score Distillation Sampling (DSDS)** loss, which regularizes NeRF densities against a pre-trained 3D diffusion model without backpropagating through the diffusion network [S2].

## How it works
The method consists of two phases: training a local 3D diffusion prior on synthetic data, and applying this prior to a NeRF during optimization via the DSDS loss and a visibility loss [S2].

### Inputs and Outputs
*   **Inputs:**
    *   Training images $I_j$ and camera poses $C_j$ [S2].
    *   Pre-trained 3D diffusion model $\epsilon_\theta$ (trained on ShapeNet cubes) [S2].
    *   Hyperparameters: Diffusion timestep $t \in [10, 50]$, density threshold $\tau = 0.01$, density weight $w = 0.01$, visibility threshold $V_{thresh} = 2$ (in normalized bounds) [S2].
*   **Outputs:**
    *   Optimized NeRF parameters $\theta$ (color and density fields) [S2].

### Data Structures
1.  **Diffusion Model Input/Output:** A voxelized 3D cube of occupancy $x$ with resolution $32 \times 32 \times 32$. Values are binary: $1$ (occupied) or $-1$ (empty) [S2].
2.  **Diffusion Model Architecture:** A small 3D U-Net with 7.2 million parameters (28 MB) [S2].
3.  **Importance Sampling Grid:** A low-resolution $20 \times 20 \times 20$ grid storing NeRF densities clamped to $[0, 1]$. This grid is updated with Exponential Moving Average (EMA) decay during training to inform sampling probabilities [S2].

### Training the 3D Prior (Offline)
Before NeRF optimization, a diffusion model is trained to learn the distribution of local 3D surface patches [S2].

1.  **Data Curation:** Extract $N$ local cube-bounded patches from ShapeNet meshes. Cube sizes vary randomly between 1-10% of the object's bounding volume [S2].
2.  **Voxelization:** Voxelize patches at resolution $32 \times 32 \times 32$ [S2].
3.  **Augmentation:** Apply random rotations and random dilation online during training to increase diversity [S2].
4.  **Diffusion Training:** Train a Denoising Diffusion Probabilistic Model (DDPM) using the standard diffusion loss [S2]:
    $$ \mathcal{L}_{Diff} = \| \epsilon - \epsilon_\theta (\sqrt{\bar{\alpha}_t} x_0 + \sqrt{1-\bar{\alpha}_t} \epsilon, t) \|_2^2 $$
    where $t \sim U(0, 1000)$, $\epsilon \sim \mathcal{N}(0, I)$, and $\bar{\alpha}_t$ follows a linear noise schedule [S2].

### NeRF Optimization Loop
During NeRF training, the DSDS loss and Visibility Loss are applied alongside the standard RGB rendering loss [S2].

1.  **Cube Importance Sampling:**
    *   Query the NeRF densities $\sigma$ at random 3D locations [S2].
    *   Use the $20 \times 20 \times 20$ importance sampling grid to select cube centers [S2].
    *   Define a cube size randomly between 1-10% of the scene size [S2].
    *   Extract a $32 \times 32 \times 32$ cube of densities $\sigma$ from the NeRF at the selected center [S2].

2.  **Discretization:**
    *   Convert the density cube $\sigma$ into a binary occupancy cube $x_t$ using threshold $\tau$:
        $$ x_t = \begin{cases} 1 & \text{if } \sigma > \tau \\ -1 & \text{if } \sigma \le \tau \end{cases} $$
    *   Note: Unlike SDS, no noise is added to $x_t$ before passing it to the diffusion model [S2].

3.  **Denoising:**
    *   Pass the discretized cube $x_t$ through the pre-trained diffusion model $\epsilon_\theta$ at timestep $t$ to predict the denoised cube $x_0$:
        $$ x_0 = \text{Denoise}(x_t, t, \epsilon_\theta) $$
    *   This step is a single forward pass; gradients do not flow through the diffusion model [S2].

4.  **Mask Computation:**
    *   Compute the mask $m$ indicating predicted empty voxels:
        $$ m = \mathbb{I}\{x_0 < 0\} $$
    *   Here, $m_i = 1$ if the $i$-th voxel is predicted empty, and $0$ if occupied [S2].

5.  **DSDS Loss Calculation:**
    *   Compute the loss $\mathcal{L}_{DSDS}$ to penalize densities in empty regions and encourage densities in occupied regions:
        $$ \mathcal{L}_{DSDS} = \sum_i \left( m_i \sigma_i + (1 - m_i) \max(w - \sigma_i, 0) \right) $$
    *   If $m_i=1$ (empty), the term $m_i \sigma_i$ penalizes the density $\sigma_i$ [S2].
    *   If $m_i=0$ (occupied), the term $(1 - m_i) \max(w - \sigma_i, 0)$ increases density if $\sigma_i < w$ [S2].

6.  **Visibility Loss Calculation:**
    *   To remove floaters outside the convex hull of training views, compute $\mathcal{L}_{vis}$ for 3D locations $q_i$ not seen by any training view:
        $$ \mathcal{L}_{vis} = \sum_i V(q_i) \sigma_i $$
    *   $V(q_i) = 1$ if location $q_i$ is not visible from any training view $j$ (approximated via frustum checks), and $0$ otherwise [S2].
    *   The visibility threshold is set to two times the largest distance between any two camera origins in the training/evaluation splits [S2]. In Nerfstudio, where bounds are $(-1,1)$, this value is 2 [S2].

7.  **Gradient Update:**
    *   Compute total loss: $\mathcal{L}_{total} = \mathcal{L}_{RGB} + \lambda_{DSDS} \mathcal{L}_{DSDS} + \lambda_{vis} \mathcal{L}_{vis}$ [S2].
    *   Update NeRF parameters $\theta$ using gradients from $\mathcal{L}_{RGB}$ and $\mathcal{L}_{DSDS}$ (and $\mathcal{L}_{vis}$) [S2]. Note that $\mathcal{L}_{DSDS}$ gradients flow through the NeRF density query but not the diffusion model [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_cube_center(importance_grid, scene_bounds):
    """Sample a 3D center coordinate weighted by the importance grid densities."""
    probs = importance_grid / np.sum(importance_grid)
    flat_idx = np.random.choice(importance_grid.size, p=probs.flatten())
    grid_idx = np.unravel_index(flat_idx, importance_grid.shape)
    voxel_size = (scene_bounds[1] - scene_bounds[0]) / importance_grid.shape[0]
    return scene_bounds[0] + np.array(grid_idx) * voxel_size + voxel_size / 2

def query_density_grid(nerf, center, size, resolution):
    """Query NeRF density at a 3D grid of points."""
    x = np.linspace(center[0] - size/2, center[0] + size/2, resolution)
    y = np.linspace(center[1] - size/2, center[1] + size/2, resolution)
    z = np.linspace(center[2] - size/2, center[2] + size/2, resolution)
    points = np.array(np.meshgrid(x, y, z)).reshape(3, -1).T
    densities = np.array([nerf.query_density(p) for p in points])
    return densities.reshape(resolution, resolution, resolution)

def is_visible(point, view, threshold):
    """Check if point is within camera frustum and within depth threshold.
    Approximates visibility via frustum check, overestimating visibility [S2]."""
    cam_pos = view.position
    cam_dir = view.direction
    to_point = point - cam_pos
    dist = np.linalg.norm(to_point)
    dot = np.dot(to_point, cam_dir) / dist
    return dot > 0.5 and dist < threshold  # [G] Standard frustum/depth approximation

def nerfbusters_step(nerf, diffusion_model, training_views, importance_grid, hyperparams, scene_bounds, ray_march_points, l_rgb):
    """
    One optimization step for Nerfbusters.
    """
    # 1. Cube Importance Sampling
    cube_center = sample_cube_center(importance_grid, scene_bounds)
    scene_extent = np.max(scene_bounds) - np.min(scene_bounds)
    cube_size = np.random.uniform(0.01, 0.10) * scene_extent
    density_cube = query_density_grid(nerf, cube_center, cube_size, resolution=32)
    
    # 2. Discretization
    tau = hyperparams['tau']
    occupancy_cube = np.where(density_cube > tau, 1, -1)
    
    # 3. Denoising
    t = hyperparams['t']
    denoised_cube = diffusion_model.forward(occupancy_cube, t)
    
    # 4. Mask Computation
    mask = (denoised_cube < 0).astype(float)
    
    # 5. DSDS Loss
    w = hyperparams['w']
    term_empty = mask * density_cube
    term_occupied = (1 - mask) * np.maximum(w - density_cube, 0)
    l_dsd = np.sum(term_empty + term_occupied)
    
    # 6. Visibility Loss
    vis_loss = 0.0
    vis_thresh = hyperparams['vis_thresh']
    for point in ray_march_points:
        visible = False
        for view in training_views:
            if is_visible(point, view, threshold=vis_thresh):
                visible = True
                break
        if not visible:
            sigma = nerf.query_density(point)
            vis_loss += sigma
            
    # 7. Total Loss and Gradients
    total_loss = l_rgb + l_dsd + vis_loss
    return total_loss
```

### Edge Cases and Pitfalls
*   **Differentiable Mapping Issues:** Using a differentiable function (e.g., sigmoid) to map densities to the diffusion range and backpropagating through it causes issues due to steep or zero gradients [S2]. DSDS avoids this by using hard thresholding and a single forward pass [S2].
*   **Accumulated Weights:** Using accumulated weights for sampling is more expensive and requires specifying a view, limiting regularization to visible regions [S2]. DSDS uses densities, which allow sampling of occluded regions [S2].
*   **Visibility Approximation:** The visibility check approximates occlusion by only checking frustum intersection, overestimating visibility. This is a simplification for efficiency [S2].

## Operating characteristics
*   **Time Complexity:** The DSDS loss requires one forward pass through the diffusion model per sampled cube [S2]. The diffusion model is small (7.2M params), making it computationally feasible [S2]. Importance sampling adds negligible overhead as it reuses ray marching data [S2].
*   **Memory Complexity:** Requires storing the diffusion model (28 MB) and the $20 \times 20 \times 20$ importance sampling grid [S2].
*   **Performance:** Achieves state-of-the-art performance for casual captures compared to other geometry regularizers on the proposed Nerfbusters dataset [S2].
*   **Strengths:** Removes floaters effectively; works on arbitrary scenes; 3D-consistent; no need for complex conditioning [S2].
*   **Limitations:** Requires decent starting densities (local prior needs context); visibility loss is approximate [S2].
*   **When to Use:** When training NeRFs on casually captured images with sparse viewpoints or floaters [S2].
*   **When Not to Use:** When high-fidelity global geometry is required without local priors, or when computational resources are extremely limited (though overhead is low) [S2].

## Background
Nerfbusters was proposed by Weber et al. in 2023 [S2]. It addresses the issue of floaters and flawed geometry in NeRFs trained on casually captured images, which are often not evaluated properly in standard protocols that only assess views near the training trajectory [S2]. The method builds on the idea of using diffusion models for 3D generation, adapting the SDS loss to work directly on 3D densities using a local 3D prior [S2]. It is implemented in the open-source Nerfstudio repository [S2].

## Sources
[S1] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S2] Nerfbusters: Removing Ghostly Artifacts from Casually Captured NeRFs, https://doi.org/10.1109/iccv51070.2023.01661, fetched 2026-10-27
