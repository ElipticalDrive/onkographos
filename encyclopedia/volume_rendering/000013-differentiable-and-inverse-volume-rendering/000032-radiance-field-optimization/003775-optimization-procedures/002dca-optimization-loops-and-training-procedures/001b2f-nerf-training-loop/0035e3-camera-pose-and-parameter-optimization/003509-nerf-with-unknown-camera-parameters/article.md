# NeRF with Unknown Camera Parameters

Also called: Joint optimisation of NeRF and camera parameters, NeRF with unknown cameras, NeRF-- (NeRF without known camera parameters)

## Overview
This method jointly optimises camera parameters alongside a Neural Radiance Field (NeRF) model on a sparse set of forward-facing views, eliminating the need for pre-estimated camera poses [S1, unofficial]. It sets itself apart from baseline NeRF by treating camera intrinsics and extrinsics as learnable variables within the same differentiable pipeline, allowing the model to self-calibrate from unposed images [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ unposed images $\{I_i\}_{i=1}^N$ with known width $W$ and height $H$.
*   **Outputs:** A trained NeRF model $F_\Theta$ and a set of optimised camera parameters $\{\pi_i\}_{i=1}^N$, where $\pi_i = (\hat{f}, \hat{\phi}_i, \hat{t}_i)$ representing the focal length, rotation (axis-angle), and translation for each image.

**Data Structures:**
*   **NeRF Model $F_\Theta$:** A multi-layer perceptron (MLP). The hidden layer dimension is reduced to 128 for computational efficiency [S1, unofficial].
*   **Camera Parameters $\pi_i$:**
    *   $\hat{f}$: Scalar focal length.
    *   $\hat{\phi}_i \in \mathbb{R}^3$: Axis-angle representation of rotation. The direction is the rotation axis, and the magnitude is the angle.
    *   $\hat{t}_i \in \mathbb{R}^3$: Translation vector (camera position).
    *   Rotation matrix $R_i$ is derived from $\hat{\phi}_i$ using Rodrigues' formula [S1, unofficial].

**Numbered Steps:**
1.  **Initialisation:**
    *   Initialise NeRF weights $\Theta$ using Kaiming initialisation [S1, unofficial].
    *   Initialise all camera positions $\hat{t}_i$ at the origin $(0,0,0)$.
    *   Initialise all camera rotations to look in the $-z$ direction.
    *   Initialise focal length $\hat{f}$ to the image width $W$ [S1, unofficial].
2.  **Ray Generation:** For each training iteration, randomly select $M$ pixels from each input image $I_i$. For each pixel $p=(u,v)$, compute the ray direction $\hat{d}_{i,p}$ and origin $\hat{o}_i$:
    $$ \hat{o}_i = \hat{t}_i $$
    $$ \hat{d}_{i,p} = R_i \begin{pmatrix} (u - W/2)/\hat{f} \\ -(v - H/2)/\hat{f} \\ -1 \end{pmatrix} $$
    where $R_i$ is the rotation matrix computed from $\hat{\phi}_i$ via Rodrigues' formula [S1, unofficial].
3.  **Sampling:** Sample $K$ 3D points $\{x_j\}$ along the ray $\hat{r}_{i,p}(h) = \hat{o}_i + h\hat{d}_{i,p}$ [S1, unofficial].
4.  **NeRF Evaluation:** Evaluate the NeRF model $F_\Theta$ at each sampled point $x_j$ to obtain radiance colour $c_j$ and density $\sigma_j$ [S1, unofficial].
5.  **Volume Rendering:** Aggregate the predicted radiance and densities along the ray using the standard differentiable volume rendering integral to produce the predicted pixel colour $\hat{C}_i(p)$ [S1, unofficial].
6.  **Loss Computation:** Compute the reconstruction loss $\mathcal{L}$ between the rendered colours $\hat{C}_i(p)$ and the ground-truth colours $C_i(p)$ for the sampled pixels [S1, unofficial].
7.  **Optimisation:** Perform gradient descent to minimise $\mathcal{L}$ with respect to both NeRF weights $\Theta$ and camera parameters $\pi_i$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def rodrigues_rotation_matrix(axis_angle):
    """
    Computes rotation matrix from axis-angle representation.
    axis_angle: Tensor of shape (3,)
    Returns: Tensor of shape (3, 3)
    """
    axis = axis_angle / (torch.norm(axis_angle) + 1e-8)
    angle = torch.norm(axis_angle)
    axis_x, axis_y, axis_z = axis[0], axis[1], axis[2]
    
    cos_a = torch.cos(angle)
    sin_a = torch.sin(angle)
    one_minus_cos = 1 - cos_a
    
    R = torch.zeros(3, 3)
    R[0, 0] = cos_a + axis_x**2 * one_minus_cos
    R[0, 1] = axis_x * axis_y * one_minus_cos - axis_z * sin_a
    R[0, 2] = axis_x * axis_z * one_minus_cos + axis_y * sin_a
    R[1, 0] = axis_y * axis_x * one_minus_cos + axis_z * sin_a
    R[1, 1] = cos_a + axis_y**2 * one_minus_cos
    R[1, 2] = axis_y * axis_z * one_minus_cos - axis_x * sin_a
    R[2, 0] = axis_z * axis_x * one_minus_cos - axis_y * sin_a
    R[2, 1] = axis_z * axis_y * one_minus_cos + axis_x * sin_a
    R[2, 2] = cos_a + axis_z**2 * one_minus_cos
    return R

def render_ray(ray_o, ray_d, points, nerf_model):
    """
    Renders a single ray using volume rendering.
    ray_o: Tensor (3,)
    ray_d: Tensor (3,)
    points: Tensor (K, 3) sampled along the ray
    nerf_model: NeRF MLP
    Returns: Tensor (3,) predicted color
    """
    # Evaluate NeRF
    # points shape: (K, 3)
    # nerf_model returns (K, 3) for color and (K, 1) for density
    # Note: Standard NeRF implementation details are assumed from parent article
    # Here we assume a simplified forward pass returning sigma and color
    sigma, color = nerf_model(points) 
    
    # Compute distances between samples
    dists = torch.cat([torch.norm(points[1:] - points[:-1], dim=1), torch.tensor([1e10])])
    
    # Alpha compositing
    alpha = 1 - torch.exp(-sigma * dists)
    T = torch.cumprod(1 - alpha + 1e-10, dim=0)
    T = torch.cat([torch.tensor([1.0]), T[:-1]])
    
    weights = alpha * T
    rendered_color = torch.sum(weights.unsqueeze(1) * color, dim=0)
    return rendered_color

def train_step(images, nerf_model, camera_params, optimizer_nerf, optimizer_cam, W, H, M, K):
    """
    One training step.
    images: List of tensors (N, 3, H, W)
    camera_params: Dict with 'f', 'phi', 't'
    """
    optimizer_nerf.zero_grad()
    optimizer_cam.zero_grad()
    
    loss = 0.0
    N = len(images)
    
    for i in range(N):
        # Randomly select M pixels
        u_coords = torch.randint(0, W, (M,))
        v_coords = torch.randint(0, H, (M,))
        
        # Get current camera params for image i
        f = camera_params['f'][i]
        phi = camera_params['phi'][i]
        t = camera_params['t'][i]
        
        R = rodrigues_rotation_matrix(phi)
        
        for j in range(M):
            u = u_coords[j].float()
            v = v_coords[j].float()
            
            # Ray direction calculation
            # d = R * [ (u - W/2)/f, -(v - H/2)/f, -1 ]^T
            ray_dir_local = torch.tensor([(u - W/2)/f, -(v - H/2)/f, -1.0])
            ray_d = torch.matmul(R, ray_dir_local)
            ray_o = t
            
            # Sample K points along the ray
            # Assuming standard sampling strategy (e.g., uniform or stratified)
            # Not given in sources, so we assume a helper function or standard NeRF sampling
            # [G] Standard NeRF sampling between near and far planes. 
            # Since near/far are not specified in NeRF-- notes, we assume standard defaults 
            # or that the parent article defines the sampling range.
            # For completeness, we assume points are sampled along the ray direction.
            # This is a placeholder for the specific sampling distribution logic 
            # which is standard in NeRF but not detailed in the NeRF-- text.
            # However, to be self-contained, we must define the points.
            # [G] Assuming uniform sampling along a fixed length L for simplicity 
            # as specific near/far bounds are not provided in the source notes.
            # In practice, this would use the scene's bounding box or near/far planes.
            t_vals = torch.linspace(0.0, 1.0, K) # Placeholder for actual depth sampling
            points = ray_o.unsqueeze(0) + ray_d.unsqueeze(0) * t_vals.unsqueeze(1)
            
            # Render
            pred_color = render_ray(ray_o, ray_d, points, nerf_model)
            gt_color = images[i][:, v_coords[j], u_coords[j]]
            
            # Loss
            loss += torch.mean((pred_color - gt_color)**2)
            
    loss.backward()
    optimizer_nerf.step()
    optimizer_cam.step()
    return loss.item()
```

**Parameters:**
*   **Pixels sampled per image ($M$):** 1024 [S1, unofficial].
*   **Points sampled along each ray ($K$):** 128 [S1, unofficial].
*   **Hidden layer dimension:** 128 [S1, unofficial].
*   **Optimisers:** Three separate Adam optimisers for NeRF weights, camera poses ($\phi, t$), and focal lengths ($f$) [S1, unofficial].
*   **Initial Learning Rate:** 0.001 for all optimisers [S1, unofficial].
*   **Learning Rate Decay:**
    *   NeRF: Multiplied by 0.9954 every 10 epochs [S1, unofficial].
    *   Poses and Focal Length: Multiplied by 0.9 every 100 epochs [S1, unofficial].
*   **Training Duration:** 10,000 epochs [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Local Minima:** Joint optimisation may trap into local minima for certain scenes (e.g., Orchids, Room), resulting in inaccurate intrinsics and worse Novel View Synthesis (NVS) quality compared to baseline [S1, unofficial].
*   **Intrinsic Accuracy:** The difference between optimised camera focal lengths and ground truth (COLMAP) can be noticeable in difficult scenes [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The method uses the same complexity as standard NeRF volume rendering but adds the overhead of optimising $3N$ rotation parameters and $3N$ translation parameters plus $N$ focal lengths. The hidden layer dimension is reduced to 128 (from 256) to improve computational efficiency [S1, unofficial].
*   **Measured Performance:** On the Blender Forward Facing (BLEFF) dataset and RealEstate10K, NeRF-- achieves comparable NVS quality to baseline NeRF (using COLMAP poses), with $\Delta$SSIM and $\Delta$LPIPS $\approx 0.05$ [S1, unofficial].
*   **Strengths:** Eliminates the need for pre-estimated camera poses (e.g., from COLMAP), enabling training on unposed image sets [S1, unofficial].
*   **Limitations:** Susceptible to local minima, particularly for scenes with ambiguous geometry or insufficient viewpoint diversity, leading to inaccurate camera parameter estimation [S1, unofficial].
*   **When to use:** When camera poses are unknown or unreliable, and the scene is forward-facing.
*   **When not to use:** When high-precision camera parameters are required and ground truth or robust pre-estimation is available, as baseline NeRF may be more stable.

## Background
*   **Origin:** Introduced by Mildenhall et al. in "NeRF--: Neural Radiance Fields Without Known Camera Parameters" [S1, unofficial].
*   **History:** The method was developed to address the limitation of standard NeRF, which requires accurate camera poses. It was evaluated on a newly introduced synthetic dataset, Blender Forward Facing (BLEFF), containing 14 path-traced scenes with rotation and translation perturbations [S1, unofficial].
*   **Usage:** Used in scenarios where acquiring calibrated camera data is difficult, such as casual photo collections or real-world captured images with unstable motion [S1, unofficial].

## Sources
[S1] NeRF--: Neural Radiance Fields Without Known Camera Parameters, https://doi.org/10.48550/arxiv.2102.07064, fetched 2026-10-26 (unofficial)
