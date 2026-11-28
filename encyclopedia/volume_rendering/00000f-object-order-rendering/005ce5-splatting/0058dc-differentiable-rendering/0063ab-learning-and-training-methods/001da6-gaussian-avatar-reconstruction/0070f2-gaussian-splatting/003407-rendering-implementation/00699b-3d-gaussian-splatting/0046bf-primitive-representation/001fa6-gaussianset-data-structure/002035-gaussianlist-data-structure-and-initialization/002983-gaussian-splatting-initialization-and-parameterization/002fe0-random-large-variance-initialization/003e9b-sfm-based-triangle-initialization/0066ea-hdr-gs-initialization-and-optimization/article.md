# HDR-GS Initialization and Optimization

Also called: HDR-GS loss, HDR-GS training

## Overview
HDR-GS initialization and optimization is a method for initializing 3D Gaussian primitives and training a Gaussian Splatting model for High Dynamic Range (HDR) novel view synthesis [S1]. It addresses the limitation of standard 3D Gaussian Splatting (3DGS) which relies on Low Dynamic Range (LDR) images and often lacks accurate Structure-from-Motion (SfM) point clouds for initialization in HDR datasets [S1]. Unlike standard 3DGS which initializes from SfM points of LDR inputs, HDR-GS recalibrates camera parameters using SfM on LDR views captured at a single exposure time to generate initial 3D Gaussian positions, then optimizes the scene using a combined pixel-wise loss and a µ-law domain constraint loss [S1]. This approach establishes a robust data foundation for 3DGS-based algorithms in 3D HDR imaging [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A multi-view HDR dataset containing $N_v$ viewpoints [S1].
*   For each viewpoint $j$, LDR images $\hat{I}_{lj}(t_s)$ captured at a specific exposure time $t_s$ [S1].
*   Ground truth HDR images $I_{hj}$ (available for synthetic scenes, not for real scenes) [S1].
*   Rendered LDR images $\hat{I}_{lj}(\Delta t_j)$ and HDR images $\hat{I}_{hj}$ generated from the current Gaussian set $G$ [S1].

**Outputs:**
*   Recalibrated camera intrinsics $M_{int}$ and extrinsics $\{M_j^{ext}\}_{j=1}^{N_v}$ [S1].
*   Initial number of points $N_p$ and their 3D positions $\{\mu_i\}_{i=1}^{N_p}$ [S1].
*   Optimized Gaussian parameters (position, features, opacity, scaling, rotation) and tone mapper network weights [S1].

### Data Structures
The method initializes the Gaussian set $G$ as described in [GaussianList Data Structure and Initialization](9768-gaussian-list-data-structure-and-initialization.md).
*   **Positions:** Initialized from SfM output $\{\mu_i\}$ [S1].
*   **Other Parameters:** Randomly initialized [S1].

### Algorithm Steps

1.  **SfM Initialization:**
    Select LDR images $\{\hat{I}_j(t_s)\}_{j=1}^{N_v}$ from the multi-view HDR dataset where all images share the same exposure time $t_s$ [S1]. This constraint is critical because SfM relies on multi-view feature detection and matching, and exposure changes degrade accuracy [S1].
    Apply the SfM mapping function $F_{SfM}$ to obtain camera parameters and initial point cloud:
    $$ M_{int}, \{M_j^{ext}\}_{j=1}^{N_v}, N_p, \{\mu_i\}_{i=1}^{N_p} = F_{SfM}(\{\hat{I}_j(t_s)\}_{j=1}^{N_v}) $$
    where $N_v$ is the number of viewpoints, $M_{int}$ is the intrinsic matrix (constant across viewpoints), and $\{\mu_i\}$ are the initial 3D Gaussian positions [S1].

2.  **Gaussian Initialization:**
    Initialize the Gaussian set $G$ with $N_p$ points at positions $\{\mu_i\}$ [S1]. Randomly initialize all other learnable parameters of $G$ (features, opacity, scaling, rotation) [S1].

3.  **Training Loop:**
    For each training iteration (total $3 \times 10^4$ iterations):
    a. **Rendering:** Render LDR images $\hat{I}_{lj}(\Delta t_j)$ and HDR images $\hat{I}_{hj}$ for a batch of $B$ viewpoints using the current Gaussian set $G$ and camera poses $\{M_j^{ext}\}$ [S1].
    b. **Loss Calculation:** Compute the total loss $L = L_p + \gamma \cdot L_c$ [S1].
    c. **Optimization:** Update parameters using the Adam optimizer [S1].

### Formulas

**Pixel-wise Loss ($L_p$):**
Combines L1 loss and D-SSIM loss on LDR images to ensure photometric consistency [S1]:
$$ L_p = \sum_{j=1}^B L_1(I_{lj}(\Delta t_j), \hat{I}_{lj}(\Delta t_j)) + \lambda \cdot L_{D-SSIM}(I_{lj}(\Delta t_j), \hat{I}_{lj}(\Delta t_j)) $$
where $B$ is the batch size, $I_{lj}$ is the ground truth LDR image, $\hat{I}_{lj}$ is the rendered LDR image, and $\lambda$ is a hyperparameter [S1].

**µ-law Domain Constraint Loss ($L_c$):**
Enforces a constraint on the rendered HDR image in the µ-law LDR domain, primarily for synthetic scenes where ground truth HDR is available [S1]:
$$ L_c = \sum_{j=1}^B \left( \frac{\log(1 + \mu \cdot norm(I_{hj}))}{\log(1 + \mu)} - \frac{\log(1 + \mu \cdot norm(\hat{I}_{hj}))}{\log(1 + \mu)} \right)^2 $$
where $I_{hj}$ is the ground truth HDR image, $\hat{I}_{hj}$ is the rendered HDR image, $\mu$ is the compression amount, and $norm(\cdot)$ is min-max normalization [S1].

**Total Loss:**
$$ L = L_p + \gamma \cdot L_c $$
where $\gamma$ controls the relative importance of $L_c$ [S1].

### Parameters and Hyperparameters

*   **$\mu$ (Compression):** The amount of compression in the µ-law normalization [S1].
*   **$\gamma$ (Constraint Weight):**
    *   Synthetic datasets: $\gamma = 0.6$ [S1].
    *   Real datasets: $\gamma = 0$ (since ground truth HDR images are unavailable) [S1].
*   **$\lambda$ (D-SSIM Weight):** Hyperparameter for the D-SSIM term in $L_p$. Value not explicitly given in the source text, but cited as part of the loss formulation [S1].
*   **Optimizer:** Adam [S1].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 1 \times 10^{-15}$ [S1]
*   **Learning Rates:**
    *   Point cloud position: Initial $1.6 \times 10^{-4}$, exponentially decaying to $1.6 \times 10^{-6}$ [S1].
    *   Point feature: $2.5 \times 10^{-3}$ [S1].
    *   Opacity: $5 \times 10^{-2}$ [S1].
    *   Scaling: $5 \times 10^{-3}$ [S1].
    *   Rotation: $1 \times 10^{-3}$ [S1].
    *   Tone mapper network: Initial $5 \times 10^{-4}$, exponentially decaying to $5 \times 10^{-5}$ [S1].
*   **Training Iterations:** $3 \times 10^4$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.optim as optim
from torch.nn.functional import l1_loss

class GaussianSet:
    def __init__(self, positions, feature_dim):
        self.positions = positions
        self.features = torch.randn(positions.shape[0], feature_dim)
        self.opacities = torch.rand(positions.shape[0])
        self.scales = torch.rand(positions.shape[0], 3)
        self.rotations = torch.rand(positions.shape[0], 4)

class ToneMapperNetwork(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.net = torch.nn.Linear(3, 3)
    def forward(self, x):
        return self.net(x)

def min_max_normalize(img):
    min_val = img.min()
    max_val = img.max()
    return (img - min_val) / (max_val - min_val + 1e-8)

def hdr_gs_initialization_and_optimization(hdr_dataset, sfm_algorithm, mu, gamma, lambda_dssim, batch_size, feature_dim=256):
    # Step 1: SfM Initialization
    ts = hdr_dataset.target_exposure
    ldr_images_for_sfm = [view.ldr_image for view in hdr_dataset if view.exposure == ts]
    M_int, M_ext_list, N_p, mu_positions = sfm_algorithm(ldr_images_for_sfm)

    # Step 2: Initialize Gaussian Set G
    G = GaussianSet(mu_positions, feature_dim)
    tone_mapper = ToneMapperNetwork()

    # Step 3: Setup Optimizer
    optimizer = optim.Adam([
        {'params': G.positions, 'lr': 1.6e-4},
        {'params': G.features, 'lr': 2.5e-3},
        {'params': G.opacities, 'lr': 5e-2},
        {'params': G.scales, 'lr': 5e-3},
        {'params': G.rotations, 'lr': 1e-3},
        {'params': tone_mapper.parameters(), 'lr': 5e-4}
    ], betas=(0.9, 0.999), eps=1e-15)

    # Exponential decay factors to reach target LRs over 30000 iterations
    decay_pos = (1.6e-6 / 1.6e-4) ** (1 / 30000)
    decay_tm = (5e-5 / 5e-4) ** (1 / 30000)
    scheduler_pos = optim.lr_scheduler.ExponentialLR(optimizer.param_groups[0], gamma=decay_pos)
    scheduler_tm = optim.lr_scheduler.ExponentialLR(optimizer.param_groups[5], gamma=decay_tm)

    # Step 4: Training Loop
    num_iterations = 30000

    for iteration in range(num_iterations):
        optimizer.zero_grad()
        batch_views = [hdr_dataset[i] for i in torch.randperm(len(hdr_dataset))[:batch_size]]
        loss_p = 0.0
        loss_c = 0.0

        for view in batch_views:
            # Standard 3DGS rasterization for LDR rendering [G]
            rendered_ldr = torch.zeros_like(view.ldr_image)
            # Standard 3DGS rasterization for HDR rendering [G]
            rendered_hdr = torch.zeros_like(view.hdr_image)
            gt_ldr = view.ldr_image
            gt_hdr = view.hdr_image

            l1_term = l1_loss(rendered_ldr, gt_ldr)
            # D-SSIM computation not provided in source [G]
            dssim_term = torch.zeros_like(l1_term)
            loss_p += l1_term + lambda_dssim * dssim_term

            if gt_hdr is not None and gamma > 0:
                norm_gt_hdr = min_max_normalize(gt_hdr)
                norm_rendered_hdr = min_max_normalize(rendered_hdr)
                mu_law_gt = torch.log(1 + mu * norm_gt_hdr) / torch.log(1 + mu)
                mu_law_rendered = torch.log(1 + mu * norm_rendered_hdr) / torch.log(1 + mu)
                loss_c += torch.mean((mu_law_gt - mu_law_rendered) ** 2)

        total_loss = loss_p + gamma * loss_c
        total_loss.backward()
        optimizer.step()
        scheduler_pos.step()
        scheduler_tm.step()

    return G, tone_mapper
```

### Edge Cases and Pitfalls
*   **SfM Input Constraint:** Only HDR views under the same exposure time are used as SfM inputs. Using images with different exposure times degrades the accuracy of SfM due to feature detection/matching failures [S1].
*   **Real vs. Synthetic Scenes:** The µ-law constraint loss $L_c$ is not used for real scenes because ground truth HDR images are not provided. For real datasets, $\gamma$ is set to 0, whereas for synthetic datasets it is set to 0.6 [S1].

## Operating characteristics

*   **Time Complexity:** Not explicitly given. Training takes $3 \times 10^4$ iterations [S1].
*   **Memory Complexity:** Not explicitly given.
*   **Measured Performance:**
    *   Hardware: Single RTX A5000 GPU [S1].
    *   Speed: 126 fps (inference speed on synthetic datasets) [S1].
    *   PSNR: 41.10 (synthetic), 36.33 (real) [S1].
    *   SSIM: 0.982 (synthetic), 0.977 (real) [S1].
    *   LPIPS: 0.011 (synthetic), 0.016 (real) [S1].
*   **Strengths:**
    *   Provides a robust initialization for 3DGS in HDR datasets where SfM point clouds are not provided [S1].
    *   Achieves high-quality novel view synthesis with fast rendering speeds [S1].
*   **Limitations:**
    *   Relies on the accuracy of SfM, which is sensitive to exposure changes [S1].
    *   The µ-law constraint loss is only applicable to synthetic scenes with ground truth HDR [S1].

## Background

HDR-GS was introduced in the paper "HDR-GS: Efficient High Dynamic Range Novel View Synthesis at 1000x Speed via Gaussian Splatting" [S1]. The method addresses the limitations of voxel-based methods and the original 3DGS, which do not natively support HDR imaging and often lack SfM point clouds for initialization in HDR datasets [S1]. It builds on the 3DGS framework [15] and incorporates techniques from HDR-NeRF [14] for handling HDR data [S1]. The method was implemented using PyTorch [75] and tested on datasets collected by [14], including real and synthetic scenes from Blender [74] [S1].

## Sources

[S1] HDR-GS: Efficient High Dynamic Range Novel View Synthesis at 1000x Speed via Gaussian Splatting, https://arxiv.org/abs/2405.15125v4, fetched 2026-10-26
