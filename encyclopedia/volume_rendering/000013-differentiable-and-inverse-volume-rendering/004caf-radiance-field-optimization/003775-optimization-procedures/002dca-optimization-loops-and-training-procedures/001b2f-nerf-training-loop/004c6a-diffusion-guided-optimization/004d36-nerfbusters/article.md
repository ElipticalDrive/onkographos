# Nerfbusters

Also called: Density Score Distillation Sampling, Nerfbusters method

## Overview
Nerfbusters is a 3D diffusion-based regularization method that improves scene geometry and removes ghostly artifacts (floaters) from Neural Radiance Fields (NeRFs) trained on casually captured images [S1]. It distinguishes itself from sibling methods like DreamFusion by employing a local 3D geometric prior learned via a diffusion model on synthetic surface patches, rather than a global scene prior or 2D image prior [S1]. This local approach is category-agnostic, simpler to model, and explicitly handles 3D occlusions, making it suitable for arbitrary scenes without requiring conditioning on specific object categories [S1].

## How it works
Nerfbusters operates in two phases: training a local 3D diffusion prior and applying it to a NeRF via Density Score Distillation Sampling (DSDS) and a Visibility Loss.

### Inputs and Outputs
*   **Inputs:** A set of training images with camera poses; a pre-trained 3D diffusion model $\epsilon_\theta$; a base NeRF $\mathcal{N}_\theta$ (typically built on the Nerfacto architecture [S1]).
*   **Outputs:** An optimized NeRF $\mathcal{N}_\theta$ with reduced floaters and improved geometry.

### Data Structures
1.  **Diffusion Model Input/Output:** A voxelized 3D cube of occupancy $x$ with resolution $32 \times 32 \times 32$. Values are binary: $1$ for occupied, $-1$ for empty [S1].
2.  **Diffusion Model Architecture:** A small 3D U-Net with 7.2 million parameters (28 MB) [S1].
3.  **Importance Sampling Grid:** A low-resolution grid of densities (clamped to $[0, 1]$) with resolution $20 \times 20 \times 20$. This grid is updated with Exponential Moving Average (EMA) decay during training to inform sampling probabilities [S1].

### Training the Diffusion Prior
Before NeRF optimization, a diffusion model is trained on synthetic data:
1.  **Data Curation:** Extract $N$ local cube-bounded patches from ShapeNet meshes. Cube sizes vary between 1-10% of the object's bounding volume [S1].
2.  **Voxelization:** Voxelize patches at $32^3$ resolution.
3.  **Augmentation:** Apply random rotations and random dilation online during training to increase diversity [S1].
4.  **Loss Function:** Train a Denoising Diffusion Probabilistic Model (DDPM) using the standard diffusion loss [S1]:
    $$ L_{Diff} = \|\epsilon - \epsilon_\theta(\sqrt{\bar{\alpha}_t} x_0 + \sqrt{1-\bar{\alpha}_t}\epsilon, t)\|^2_2 $$
    where $t \sim U(0, 1000)$, $\epsilon \sim \mathcal{N}(0, I)$, and $\bar{\alpha}_t$ follows a linear noise schedule [S1].

### NeRF Optimization Steps
During NeRF training, the following steps are performed iteratively:

1.  **Cube Importance Sampling:**
    *   Select a center point in the scene based on the Importance Sampling Grid.
    *   Define a cube size randomly chosen in the range of 1-10% of the scene size.
    *   Query the NeRF for densities $\sigma$ at the $32^3$ grid points within this cube [S1].

2.  **Discretization:**
    *   Convert densities $\sigma$ to binary occupancy $x_t$ using a threshold $\tau$:
        $$ x_t = \begin{cases} 1 & \text{if } \sigma_i > \tau \\ -1 & \text{otherwise} \end{cases} $$
    *   Typical value: $\tau = 0.01$ [S1].

3.  **Diffusion Denoising:**
    *   Pass the discretized cube $x_t$ through the pre-trained diffusion model $\epsilon_\theta$ at a specific timestep $t$.
    *   The model predicts the denoised cube $x_0$.
    *   Typical timestep: $t \in [10, 50]$. This acts as a learning rate control; smaller $t$ removes less noise [S1].

4.  **Mask Generation:**
    *   Compute the mask $m$ indicating predicted empty voxels:
        $$ m_i = \mathbb{1}\{x_{0,i} < 0\} $$
    *   If $m_i=1$, the diffusion model predicts the voxel is empty. If $m_i=0$, it predicts occupied [S1].

5.  **DSDS Loss Calculation:**
    *   Compute the loss $L_{DSDS}$ to penalize densities where the model predicts empty and push densities up where it predicts occupied:
        $$ L_{DSDS} = \sum_i \left( m_i \sigma_i + (1 - m_i) \max(w - \sigma_i, 0) \right) $$
    *   $w$ is the target density weight for occupied space. Typical value: $w = 0.01$ [S1].
    *   Note: Unlike SDS, DSDS does not add noise to the original sampled occupancy before providing it to the diffusion model [S1].

6.  **Visibility Loss Calculation:**
    *   Identify 3D locations $q_i$ not seen by any training view (or seen by fewer than a threshold).
    *   Visibility $V(q_i)$ is approximated by frustum checks: $V(q_i) = \mathbb{1}\{\sum_{j=1}^k v_{ij} < 1\}$, where $v_{ij}$ is the visibility of point $i$ in view $j$ [S1].
    *   Compute loss:
        $$ L_{vis} = \sum_i V(q_i) \sigma_i $$
    *   This penalizes densities in regions outside the convex hull of training views [S1].

7.  **Gradient Update:**
    *   Backpropagate the combined loss (typically $L_{RGB} + \lambda_{DSDS} L_{DSDS} + \lambda_{vis} L_{vis}$) through the NeRF to update parameters $\theta$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class NerfbustersOptimizer:
    def __init__(self, nerf, diffusion_model, tau=0.01, w=0.01, t_step=20):
        """
        Initialize Nerfbusters optimizer.
        
        Args:
            nerf: The NeRF model (e.g., Nerfacto) supporting density queries.
            diffusion_model: Pre-trained 3D U-Net diffusion model.
            tau: Density threshold for binarization.
            w: Target density weight for occupied voxels.
            t_step: Diffusion timestep for DSDS.
        """
        self.nerf = nerf
        self.diffusion_model = diffusion_model
        self.tau = tau
        self.w = w
        self.t_step = t_step
        # Importance sampling grid (20x20x20)
        self.sampling_grid = torch.zeros(20, 20, 20)
        self.grid_ema = 0.99
        
    def sample_cube(self, scene_bounds):
        """
        Sample a 3D cube of densities from the NeRF using importance sampling.
        
        Returns:
            densities: Tensor of shape (32, 32, 32)
            center: 3D coordinates of the cube center
        """
        # Select center based on sampling_grid probabilities
        # [G] Implementation detail: Normalize grid to probabilities and sample index
        grid_probs = self.sampling_grid / (self.sampling_grid.sum() + 1e-8)
        idx = torch.multinomial(grid_probs.flatten(), 1).item()
        center = index_to_3d_coord(idx, scene_bounds, grid_res=20)
        
        # Random cube size 1-10% of scene
        size_factor = torch.rand(1).item() * 0.09 + 0.01
        cube_size = size_factor * scene_bounds.extent
        
        # Query NeRF densities at 32x32x32 grid points around center
        # [G] Implementation detail: Generate grid points relative to center
        points = generate_grid_points(center, cube_size, resolution=32)
        densities = self.nerf.query_density(points)
        
        return densities, center

    def compute_dsd_loss(self, densities):
        """
        Compute Density Score Distillation Sampling loss.
        
        Args:
            densities: Tensor of shape (32, 32, 32)
            
        Returns:
            loss: Scalar DSDS loss
        """
        # Discretize densities
        # x_t = 1 if sigma > tau else -1
        x_t = torch.where(densities > self.tau, torch.ones_like(densities), torch.full_like(densities, -1.0))
        
        # Denoise using pre-trained diffusion model
        # [G] Implementation detail: diffusion_model expects batched input, add batch dim
        x_0_pred = self.diffusion_model(x_t.unsqueeze(0), t=self.t_step).squeeze(0)
        
        # Compute mask: 1 if predicted empty (x_0 < 0)
        m = (x_0_pred < 0).float()
        
        # L_DSDS = sum( m_i * sigma_i + (1 - m_i) * max(w - sigma_i, 0) )
        term_empty = m * densities
        term_occupied = (1 - m) * torch.clamp(self.w - densities, min=0)
        loss = (term_empty + term_occupied).sum()
        
        return loss

    def compute_visibility_loss(self, query_points):
        """
        Compute visibility loss for points not seen by training views.
        
        Args:
            query_points: Tensor of shape (N, 3)
            
        Returns:
            loss: Scalar visibility loss
        """
        # [G] Implementation detail: Check if points are within frustum of any training camera
        # V(q_i) = 1 if sum of visibilities < 1
        visibilities = check_frustum_visibility(query_points, self.nerf.training_cameras)
        is_not_visible = (visibilities.sum(dim=0) < 1).float()
        
        # Get densities at query points
        densities = self.nerf.query_density(query_points)
        
        loss = (is_not_visible * densities).sum()
        return loss

    def update(self, batch):
        """
        Perform one optimization step.
        """
        # 1. Sample cubes
        densities, center = self.sample_cube(self.nerf.scene_bounds)
        
        # 2. Compute DSDS Loss
        loss_dsd = self.compute_dsd_loss(densities)
        
        # 3. Compute Visibility Loss (optional, for regions outside convex hull)
        # [G] Implementation detail: Sample points specifically for visibility check
        vis_points = sample_points_outside_hull(self.nerf.training_cameras, self.nerf.scene_bounds)
        loss_vis = self.compute_visibility_loss(vis_points)
        
        # 4. Combine with RGB Loss (standard NeRF training)
        # [G] Implementation detail: Render image from batch and compute MSE/LPIPS
        loss_rgb = self.nerf.render_and_compute_rgb_loss(batch)
        
        total_loss = loss_rgb + 0.1 * loss_dsd + 0.1 * loss_vis
        
        # 5. Backpropagate
        total_loss.backward()
        
        # 6. Update Sampling Grid (EMA)
        # [G] Implementation detail: Update grid values based on current densities
        self.update_sampling_grid(densities, center)
        
        # 7. Optimizer step
        self.optimizer.step()
        self.optimizer.zero_grad()

    def update_sampling_grid(self, densities, center):
        """
        Update the importance sampling grid with EMA decay.
        """
        # Map center and densities to grid indices
        # [G] Implementation detail: Convert 3D center and density values to grid coordinates
        grid_idx = coord_to_grid_index(center, self.nerf.scene_bounds, 20)
        # Update EMA
        self.sampling_grid[grid_idx] = self.grid_ema * self.sampling_grid[grid_idx] + (1 - self.grid_ema) * densities.mean()
```

### Parameters
*   **Diffusion Timestep $t$:** Determines noise removal amount; interpreted as a learning rate. Typical value: $t \in [10, 50]$ [S1].
*   **Density Threshold $\tau$:** Decides at what density a voxel is considered empty or occupied. Typical value: $\tau = 0.01$ [S1].
*   **Density Weight $w$:** Determines how much to increase densities in occupied space. Typical value: $w = 0.01$ [S1].
*   **Importance Sampling Grid Resolution:** $20 \times 20 \times 20$ [S1].
*   **Cube Size for Sampling:** Randomly chosen in range 1-10% of scene size [S1].
*   **Visibility Depth Threshold:** Two times the largest distance between any two camera origins in training and evaluation splits. In Nerfstudio, this corresponds to a value of 2 due to camera poses scaled to fit within bounds $(-1,-1,-1)$ and $(1,1,1)$ [S1].

### Pitfalls
*   **Differentiable Mapping Issues:** Using a differentiable function (e.g., sigmoid) to convert densities to the diffusion range and backpropagating through it causes issues due to steep or zero gradients [S1]. DSDS avoids this by using discrete binarization and a single forward pass through the diffusion model [S1].
*   **Accumulated Weights:** Using accumulated weights for sampling is more expensive (requires shooting rays) and limits regularization to visible regions from a specific view [S1].
*   **Initial Densities:** The local 3D diffusion prior requires decent starting densities since it operates locally and needs contextual information to ground its denoising steps [S1].

## Operating characteristics
*   **Time Complexity:** The DSDS loss requires one forward pass through the diffusion model per sampled cube. The diffusion model is small (7.2M parameters), making it relatively fast compared to large 2D image diffusion models [S1].
*   **Memory Complexity:** The diffusion model occupies 28 MB. The importance sampling grid is small ($20^3$ floats) [S1].
*   **Performance:** Nerfbusters achieves state-of-the-art performance for casual captures compared to other geometry regularizers on the proposed Nerfbusters dataset [S1].
*   **Strengths:** Removes floaters and cloudy artifacts; improves geometry in regions with sparse supervision; category-agnostic; handles occlusions explicitly [S1].
*   **Limitations:** Requires a pre-trained diffusion model; relies on local patches, so it may struggle if initial densities are very poor [S1].

## Background
Nerfbusters was proposed by Weber et al. in 2023 [S1]. It addresses the issue of artifacts in NeRFs trained on casually captured images, where existing evaluation protocols often fail to capture these effects by only assessing image quality at every 8th frame of the training capture [S1]. The method introduces a new dataset and evaluation procedure involving two camera trajectories (training and evaluation) to better assess novel-view synthesis [S1]. It is implemented in the open-source Nerfstudio repository [S1].

## Sources
[S1] Nerfbusters: Removing Ghostly Artifacts from Casually Captured NeRFs, https://doi.org/10.1109/iccv51070.2023.01661, fetched 2026-10-27
