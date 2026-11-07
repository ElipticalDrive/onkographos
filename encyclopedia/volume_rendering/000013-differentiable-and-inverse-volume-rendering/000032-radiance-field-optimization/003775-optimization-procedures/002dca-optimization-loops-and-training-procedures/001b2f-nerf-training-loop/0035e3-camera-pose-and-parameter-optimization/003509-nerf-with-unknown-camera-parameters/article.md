# NeRF with Unknown Camera Parameters

Also called: Joint optimisation of NeRF and camera parameters, NeRF with unknown cameras, NeRF-- (NeRF without known camera parameters)

## Overview
NeRF with Unknown Camera Parameters (NeRF--) is a method that jointly optimizes camera parameters alongside a Neural Radiance Field (NeRF) model on a sparse set of forward-facing views, eliminating the need for pre-estimated camera poses [S1, unofficial]. It differs from baseline NeRF, which requires ground-truth or COLMAP-estimated poses, by treating camera intrinsics and extrinsics as differentiable trainable parameters within the same optimization loop [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images $\{I_i\}$ with unknown camera parameters.
*   **Outputs:** A trained NeRF model $F_\Theta$ and optimized camera parameters $\{\pi_i\}$ for each image.

**Data Structures:**
*   **NeRF Model ($F_\Theta$):** A multi-layer perceptron (MLP) mapping 3D coordinates and viewing directions to color and density. The implementation uses a hidden layer dimension of 128 [S1, unofficial].
*   **Camera Parameters ($\pi_i$):** For each image $i$, parameters are defined as $\pi_i = (\hat{f}, \hat{\phi}_i, \hat{t}_i)$, where $\hat{f}$ is the shared focal length, $\hat{\phi}_i \in \mathbb{R}^3$ is the axis-angle rotation vector, and $\hat{t}_i \in \mathbb{R}^3$ is the translation vector [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:**
    *   Initialize NeRF weights $\Theta$ using Kaiming initialization [S1, unofficial].
    *   Initialize all camera poses at the origin, looking in the $-z$ direction.
    *   Initialize the focal length $\hat{f}$ to the image width [S1, unofficial].
2.  **Ray Generation:** For a pixel $p=(u,v)$ in image $I_i$, compute the ray origin $\hat{o}_i$ and direction $\hat{d}_{i,p}$ using current estimates $\hat{\pi}_i$:
    $$ \hat{o}_i = \hat{t}_i $$
    $$ \hat{d}_{i,p} = \hat{R}_i \begin{pmatrix} (u - W/2)/\hat{f} \\ -(v - H/2)/\hat{f} \\ -1 \end{pmatrix} $$
    where $W, H$ are image dimensions, and $\hat{R}_i$ is the rotation matrix derived from $\hat{\phi}_i$ using Rodrigues' formula [S1, unofficial].
3.  **Volume Rendering:**
    *   Sample $K=128$ 3D points $\{x_j\}$ along the ray [S1, unofficial].
    *   Evaluate radiance colors $\{c_j\}$ and density values $\{\sigma_j\}$ from $F_\Theta$ [S1, unofficial].
    *   Aggregate these values using the standard volume rendering integral to predict pixel color $\hat{C}_i(p)$ [S1, unofficial].
4.  **Loss Computation:** Randomly select $M=1024$ pixels per image. Compute the photometric reconstruction loss:
    $$ \mathcal{L} = \| \hat{C}_i(p) - I_i(p) \|^2 $$
    [S1, unofficial].
5.  **Optimization:** Compute gradients $\nabla_\Theta \mathcal{L}$ and $\nabla_{\pi} \mathcal{L}$. Update $\Theta$ and $\pi_i$ using Adam optimizers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def rodrigues(axis_angle):
    """Convert axis-angle rotation vector to rotation matrix.
    axis_angle: (3,) tensor
    Returns: (3, 3) tensor
    """
    # Implementation of Rodrigues' formula
    # [G] Standard mathematical implementation omitted for brevity, 
    # but required for differentiability.
    pass

def render_ray(ray_o, ray_d, nerf_model, num_samples=128):
    """Render a single ray using volume rendering.
    ray_o: (3,) tensor, ray origin
    ray_d: (3,) tensor, ray direction
    nerf_model: NeRF MLP
    num_samples: int, number of points to sample
    Returns: (3,) tensor, predicted color
    """
    # [G] Standard NeRF sampling and alpha compositing logic
    # 1. Sample points along ray_d from ray_o
    # 2. Query nerf_model for sigma and color
    # 3. Compute transmittance and accumulated color
    pass

def train_step(nerf_model, camera_params, images, pixels, optimizer_nerf, optimizer_cam, focal_length):
    """One training step."""
    optimizer_nerf.zero_grad()
    optimizer_cam.zero_grad()
    
    # camera_params: list of (phi, t) tuples for each image
    # pixels: list of (u, v) coordinates for sampled pixels
    
    loss = 0.0
    for i, (img, ps) in enumerate(zip(images, pixels)):
        phi, t = camera_params[i]
        
        # 1. Ray Generation
        # Compute rotation matrix from axis-angle phi
        R = rodrigues(phi)
        
        # Compute ray origins and directions for all pixels in this image
        # u, v are normalized or pixel coordinates depending on implementation
        # Assuming u, v are pixel coordinates
        u, v = ps[:, 0], ps[:, 1]
        W, H = img.shape[1], img.shape[2]
        
        # Direction calculation
        # d = R * [ (u - W/2)/f, -(v - H/2)/f, -1 ]^T
        # [G] Ensure proper broadcasting and tensor operations
        dirs = torch.stack([
            (u - W/2) / focal_length,
            -(v - H/2) / focal_length,
            torch.ones_like(u) * -1
        ], dim=1) # (N, 3)
        
        # Apply rotation
        # R is (3,3), dirs is (N,3) -> result (N,3)
        ray_d = torch.matmul(dirs, R.t()) # Assuming R is row-major or adjust accordingly
        ray_o = t.unsqueeze(0).expand(ray_d.shape[0], -1) # (N, 3)
        
        # 2. Volume Rendering
        pred_colors = []
        for j in range(ray_d.shape[0]):
            c = render_ray(ray_o[j], ray_d[j], nerf_model)
            pred_colors.append(c)
        pred_colors = torch.stack(pred_colors, dim=0) # (N, 3)
        
        # 3. Loss
        gt_colors = img[:, ps[:, 1].long(), ps[:, 0].long()].permute(1, 0) # (N, 3)
        # [G] Assuming H, W indexing matches u, v
        pixel_loss = torch.mean((pred_colors - gt_colors) ** 2)
        loss += pixel_loss
        
    loss.backward()
    optimizer_nerf.step()
    optimizer_cam.step()
    
    return loss.item()
```

**Parameters:**
*   **Hidden Layer Dimension:** 128 (reduced from 256 for efficiency) [S1, unofficial].
*   **Pixels Sampled ($M$):** 1024 per image [S1, unofficial].
*   **Points Sampled ($K$):** 128 per ray [S1, unofficial].
*   **Optimizers:** Three separate Adam optimizers for NeRF weights, camera poses ($\phi, t$), and focal length ($f$) [S1, unofficial].
*   **Learning Rate:** Initial LR 0.001 for all [S1, unofficial].
*   **Decay:** NeRF LR decays by 0.9954 every 10 epochs; Camera LR decays by 0.9 every 100 epochs [S1, unofficial].
*   **Training Duration:** 10,000 epochs [S1, unofficial].

**Pitfalls:**
*   **Local Minima:** Joint optimization may trap into local minima, particularly for scenes like "Orchids" or "Room," resulting in inaccurate intrinsics and worse Novel View Synthesis (NVS) quality compared to baseline [S1, unofficial].
*   **Initialization Sensitivity:** Incorrect initialization of focal length or pose can lead to divergence. The method initializes cameras at the origin looking at $-z$ with focal length equal to image width [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Comparable to standard NeRF training, but requires additional gradient computation through the camera parameterization (Rodrigues' formula and ray generation) [S1, unofficial].
*   **Memory Complexity:** Similar to standard NeRF, with minor overhead for storing camera parameters [S1, unofficial].
*   **Performance:** Achieves comparable NVS quality to baseline NeRF (using COLMAP poses) with $\Delta$SSIM and $\Delta$LPIPS $\approx 0.05$ [S1, unofficial].
*   **Strengths:** Eliminates dependency on pre-estimated camera poses (e.g., from COLMAP), enabling application to datasets where poses are unavailable or unreliable [S1, unofficial].
*   **Limitations:** Susceptible to local minima; performance degrades on scenes with insufficient geometric constraints or ambiguous motion [S1, unofficial].

## Background
NeRF-- was introduced by Mildenhall et al. in 2021 [S1, unofficial]. It addresses the limitation of the original NeRF method, which required accurate camera poses as input. The method was evaluated on the Blender Forward Facing (BLEFF) dataset, a synthetic dataset designed for evaluating joint optimization systems [S1, unofficial].

## Sources
[S1] NeRF--: Neural Radiance Fields Without Known Camera Parameters, https://doi.org/10.48550/arxiv.2102.07064, fetched 2026-10-26 (unofficial)
