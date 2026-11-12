# Nerfbusters

Also called: Density Score Distillation Sampling, Nerfbusters method

## Overview
Nerfbusters is a 3D diffusion-based regularization method that improves scene geometry and removes ghostly artifacts (floaters) from Neural Radiance Fields (NeRFs) trained on casually captured images [S1]. It distinguishes itself from global 3D priors and 2.5D RGB-D diffusion priors by learning a category-agnostic local 3D geometric prior via a Denoising Diffusion Probabilistic Model (DDPM) and applying it through a novel Density Score Distillation Sampling (DSDS) loss [S1]. Unlike SDS, DSDS operates directly on discretized density values without adding noise to the input cube, and unlike visibility-based methods, it regularizes occluded regions by sampling local 3D patches independently of camera viewpoints [S1].

## How it works
The method consists of two phases: training a local 3D diffusion prior on synthetic data, and applying this prior to a NeRF during optimization via the DSDS loss and a visibility loss [S1].

### Inputs and Outputs
**Inputs:**
*   Training images and camera poses.
*   Pre-trained 3D diffusion model $\epsilon_\theta$ (trained on ShapeNet cubes).
*   NeRF parameters (typically built on the Nerfacto architecture from Nerfstudio) [S1].

**Outputs:**
*   Optimized NeRF model with reduced floaters and crisper geometry [S1].

### Data Structures
1.  **Diffusion Model Input/Output:** A voxelized 3D cube of occupancy $x$ with resolution $32 \times 32 \times 32$ [S1].
2.  **Diffusion Model Architecture:** A small 3D U-Net with 7.2 million parameters (28 MB) [S1].
3.  **Importance Sampling Grid:** A low-resolution grid of densities clamped to $[0, 1]$, updated with Exponential Moving Average (EMA) decay. Resolution: $20 \times 20 \times 20$ [S1].
4.  **Visibility Mask:** A binary mask indicating regions not seen by any training view or where predicted depth exceeds a threshold [S1].

### Parameters
*   **Diffusion Timestep $t$:** Determines the amount of noise removed; interpreted as a learning rate. Typical value: $t \in [10, 50]$ [S1].
*   **Density Threshold $\tau$:** Threshold for binarizing densities. Typical value: $\tau = 0.01$ [S1].
*   **Density Weight $w$:** Target density for occupied voxels. Typical value: $w = 0.01$ [S1].
*   **Visibility Depth Threshold:** Two times the largest distance between any two camera origins in training/evaluation splits. In Nerfstudio (where bounds are $(-1,-1,-1)$ to $(1,1,1)$), this value is 2 [S1].
*   **Cube Size:** Randomly chosen in the range of 1-10% of the scene size [S1].

### Steps

1.  **Train Diffusion Prior (Offline):**
    *   Extract $N$ local cube-bounded patches from ShapeNet meshes. Cube sizes vary between 1-10% of the object's bounding volume [S1].
    *   Voxelize patches at resolution $32 \times 32 \times 32$ [S1].
    *   Augment cubes with random rotations and random dilation during online training [S1].
    *   Train a DDPM $\epsilon_\theta$ using the loss:
        $$ L_{Diff} = \|\epsilon - \epsilon_\theta(\sqrt{\bar{\alpha}_t} x_0 + \sqrt{1-\bar{\alpha}_t}\epsilon, t)\|^2_2 $$
        where $t \sim U(0, 1000)$, $\epsilon \sim \mathcal{N}(0, I)$, and $\bar{\alpha}_t$ follows a linear schedule [S1].

2.  **Initialize NeRF and Sampling Grid:**
    *   Initialize NeRF (e.g., Nerfacto) [S1].
    *   Initialize importance sampling grid of densities (resolution $20^3$) with EMA decay [S1].

3.  **NeRF Training Loop (Per Step):**
    *   **Ray Marching:** Render training images using standard differentiable volume rendering [S1].
    *   **Update Sampling Grid:** Update the $20^3$ grid with current NeRF densities (clamped to $[0,1]$) using EMA decay [S1].
    *   **Cube Sampling:**
        *   Select a center point in the scene based on probabilities defined by the sampling grid [S1].
        *   Define a cube centered at this point with size randomly chosen from 1-10% of the scene size [S1].
        *   Query NeRF densities $\sigma$ for all voxels in this cube [S1].
    *   **Binarization:**
        *   Discretize densities $\sigma$ into binary occupancy $x_t$:
            $$ x_t = \begin{cases} 1 & \text{if } \sigma_i > \tau \\ -1 & \text{otherwise} \end{cases} $$
        *   Here $\tau$ is the density threshold (typically 0.01) [S1].
    *   **Denoising:**
        *   Pass $x_t$ through the pre-trained diffusion model $\epsilon_\theta$ at timestep $t$ to obtain the denoised cube $x_0$ [S1].
        *   Note: Unlike SDS, no additional noise is added to $x_t$ before passing to the model [S1].
    *   **Compute Mask:**
        *   Compute mask $m$ from denoised predictions:
            $$ m_i = \mathbb{1}\{x_{0,i} < 0\} $$
        *   $m_i=1$ indicates the diffusion model predicts the voxel as empty [S1].
    *   **Compute DSDS Loss:**
        $$ L_{DSDS} = \sum_i m_i \sigma_i + (1 - m_i) \max(w - \sigma_i, 0) $$
        *   Penalize density $\sigma_i$ if predicted empty ($m_i=1$) [S1].
        *   Increase density if predicted occupied ($m_i=0$) and $\sigma_i < w$ [S1].
    *   **Compute Visibility Loss:**
        *   Identify 3D locations $q_i$ not visible from any training view using frustum checks [S1].
        *   Compute visibility mask $V(q_i) = \mathbb{1}\{\sum_{j=1}^V v_{ij} < 1\}$, where $v_{ij}$ is the visibility of location $i$ in view $j$ [S1].
        *   Compute loss:
            $$ L_{vis} = \sum_i V(q_i) \sigma_i $$
        *   This penalizes densities in regions outside the convex hull of training views [S1].
    *   **Optimization:**
        *   Update NeRF parameters using the combined loss: $L_{total} = L_{render} + \lambda_{DSDS} L_{DSDS} + \lambda_{vis} L_{vis}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

# Hyperparameters
T_MIN = 10
T_MAX = 50
TAU = 0.01
W = 0.01
GRID_RES = 20
CUBE_RES = 32
EMA_DECAY = 0.99

# Pre-trained diffusion model
diffusion_model = load_diffusion_model() # 3D U-Net, 32x32x32 input

# NeRF model (e.g., Nerfacto)
nerf = NerfactoModel()

# Importance sampling grid
sampling_grid = np.zeros((GRID_RES, GRID_RES, GRID_RES))

def get_cube_from_nerf(nerf, center, size, grid_res):
    """
    Sample a 3D cube of densities from the NeRF.
    center: (x, y, z) in normalized scene coordinates
    size: float, relative size of the cube
    grid_res: resolution of the cube (32)
    """
    # Define voxel coordinates relative to center
    coords = np.linspace(-0.5, 0.5, grid_res)
    X, Y, Z = np.meshgrid(coords, coords, coords)
    
    # Scale by size and shift by center
    points = np.stack([X * size + center[0], 
                       Y * size + center[1], 
                       Z * size + center[2]], axis=-1)
    
    # Query NeRF for densities
    # Assuming nerf.query_density returns a tensor of shape (N, 1)
    densities = nerf.query_density(points.reshape(-1, 3))
    return densities.reshape(grid_res, grid_res, grid_res)

def binarize(densities, tau):
    """
    Discretize densities into binary occupancy.
    1 if density > tau, else -1.
    """
    return np.where(densities > tau, 1, -1)

def denoise(cube, t, diffusion_model):
    """
    Pass cube through diffusion model at timestep t.
    Returns denoised cube x0.
    """
    # Diffusion model expects input in range [-1, 1] or similar
    # The model predicts noise epsilon, we invert the forward process
    # Simplified: assume diffusion_model.denoise(x_t, t) returns x_0
    return diffusion_model.denoise(cube, t)

def compute_dsd_loss(densities, mask, w):
    """
    Compute DSDS loss.
    mask: 1 if diffusion predicts empty, 0 if occupied.
    """
    # Term 1: Penalize density where mask is 1 (empty)
    term1 = mask * densities
    
    # Term 2: Increase density where mask is 0 (occupied) if density < w
    # max(w - density, 0)
    term2 = (1 - mask) * np.maximum(w - densities, 0)
    
    return np.sum(term1 + term2)

def compute_visibility_loss(nerf, training_views, points):
    """
    Compute visibility loss for points not seen by any training view.
    """
    # Approximate visibility using frustum check
    # v_ij = 1 if point i is in frustum of view j
    # V(q_i) = 1 if sum(v_ij) < 1
    vis_mask = np.zeros(len(points))
    for view in training_views:
        # Check if point is within view's frustum
        in_frustum = check_frustum(view, points)
        vis_mask += in_frustum
    
    # Points not visible from ANY view
    not_visible = vis_mask < 1
    
    # Get densities at these points
    densities = nerf.query_density(points)
    
    return np.sum(densities[not_visible])

def train_step(nerf, diffusion_model, training_views, sampling_grid, ema_decay):
    # 1. Sample cube center based on sampling grid probabilities
    # Flatten grid to 1D array for sampling
    grid_flat = sampling_grid.flatten()
    grid_flat = grid_flat / np.sum(grid_flat) # Normalize to probability
    
    # Sample index
    idx = np.random.choice(len(grid_flat), p=grid_flat)
    center = index_to_3d_coord(idx, GRID_RES)
    
    # Random cube size
    size = np.random.uniform(0.01, 0.10) # 1-10% of scene
    
    # 2. Query densities
    densities = get_cube_from_nerf(nerf, center, size, CUBE_RES)
    
    # 3. Binarize
    x_t = binarize(densities, TAU)
    
    # 4. Denoise
    t = np.random.randint(T_MIN, T_MAX + 1)
    x_0 = denoise(x_t, t, diffusion_model)
    
    # 5. Compute mask
    # m = 1 if x_0 < 0
    mask = (x_0 < 0).astype(float)
    
    # 6. Compute DSDS Loss
    l_dsd = compute_dsd_loss(densities, mask, W)
    
    # 7. Compute Visibility Loss (optional but recommended)
    # Sample points for visibility check (e.g., from the cube or global)
    # For simplicity, assume we check points in the cube
    points_in_cube = ... # Get 3D coordinates of cube voxels
    l_vis = compute_visibility_loss(nerf, training_views, points_in_cube)
    
    # 8. Update NeRF
    total_loss = l_render + l_dsd + l_vis
    nerf.update(total_loss)
    
    # 9. Update Sampling Grid with EMA
    # Update grid with current densities (clamped to [0,1])
    current_densities = densities
    current_densities = np.clip(current_densities, 0, 1)
    # Map cube densities back to grid? Or update grid based on cube center?
    # The paper says "store a low-resolution grid... updated with EMA decay"
    # This implies the grid is a global structure.
    # For implementation, one might update the grid cells corresponding to the cube center
    # or simply update the grid with the new densities if the grid represents the whole scene.
    # Given the grid is 20x20x20 and cube is 32x32x32, the grid is coarser.
    # We assume the grid is updated by averaging or EMA with the new sampled densities
    # mapped to the grid resolution.
    sampling_grid = ema_decay * sampling_grid + (1 - ema_decay) * current_densities

```

### Edge Cases and Pitfalls
*   **Gradient Issues:** Using a differentiable function (e.g., sigmoid) to map densities to the diffusion range and backpropagating through it causes issues due to steep or zero gradients [S1]. DSDS avoids this by using discrete binarization and a single forward pass through the diffusion model [S1].
*   **Visibility Approximation:** The visibility loss uses frustum checks which overestimate visibility by not handling occlusions [S1].
*   **Sampling Bias:** Using accumulated weights for sampling is more expensive and limits regularization to visible regions from a specific view [S1]. DSDS uses densities to sample occluded regions as well [S1].

## Operating characteristics
*   **Time Complexity:** The DSDS loss requires one forward pass through the diffusion model per sampled cube. The diffusion model is small (7.2M parameters), making it efficient [S1]. Importance sampling adds negligible cost as it uses a small $20^3$ grid [S1].
*   **Memory Complexity:** The diffusion model requires 28 MB [S1]. The importance sampling grid requires minimal memory ($20^3$ floats) [S1].
*   **Performance:** Nerfbusters achieves state-of-the-art performance for casual captures compared to other geometry regularizers on the proposed Nerfbusters dataset [S1].
*   **Strengths:** Removes floaters, improves geometry in sparse supervision regions, category-agnostic, works on arbitrary scenes [S1].
*   **Limitations:** Requires a pre-trained diffusion model. The visibility loss approximation may not perfectly handle occlusions [S1].

## Background
Nerfbusters was proposed by Weber et al. in 2023 [S1]. It addresses the issue of floaters and flawed geometry in NeRFs trained on casually captured images, which are often not captured by standard evaluation protocols that only assess image quality at every 8th frame of the training trajectory [S1]. The method builds on the Nerfacto model from the Nerfstudio repository [S1]. It differs from global 3D priors by using local geometry, which is simpler and category-agnostic [S1]. It also differs from 2.5D RGB-D diffusion priors by explicitly modeling occlusions and being 3D-consistent by definition [S1].

## Sources
[S1] Nerfbusters: Removing Ghostly Artifacts from Casually Captured NeRFs, https://doi.org/10.1109/iccv51070.2023.01661, fetched 2026-10-27
