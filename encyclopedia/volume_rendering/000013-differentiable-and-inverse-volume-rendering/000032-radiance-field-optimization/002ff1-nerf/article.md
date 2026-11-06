# NeRF++


## Overview
NeRF++ (also known as NeRF--) is a framework for novel view synthesis that jointly optimizes camera parameters (intrinsics and 6DoF poses) and the scene representation (NeRF) from a sparse set of forward-facing RGB images, eliminating the need for pre-computed camera parameters via Structure-from-Motion (SfM) [S2, unofficial]. Unlike standard NeRF which assumes known camera poses, NeRF++ treats camera parameters as learnable variables optimized end-to-end alongside the neural radiance field [S2, unofficial]. It is specifically designed for forward-facing scenes and differs from concurrent methods that either require intrinsics as input or assume a pre-trained NeRF [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ RGB images $I = \{I_1, \dots, I_N\}$ captured from sparse viewpoints. No camera parameters are provided.
*   **Outputs:** A trained NeRF model $F_\Theta$ and optimized camera parameters $\Pi^* = \{\hat{f}, \{\hat{\phi}_i\}, \{\hat{t}_i\}\}_{i=1}^N$ [S2, unofficial].

**Data Structures and Layout:**
1.  **Camera Intrinsics:** Represented by a single learnable scalar focal length $\hat{f} \in \mathbb{R}$. The principal points are fixed at the image center: $c_x \approx W/2$ and $c_y \approx H/2$, where $W$ and $H$ are image width and height [S2, unofficial].
2.  **Camera Poses:** Represented as a camera-to-world transformation matrix $T_{wc} = [R|t] \in SE(3)$ for each image $i$.
    *   **Translation:** $\hat{t}_i \in \mathbb{R}^3$ is a learnable vector [S2, unofficial].
    *   **Rotation:** Represented by an axis-angle vector $\hat{\phi}_i \in \mathbb{R}^3$, where $\hat{\phi}_i = \hat{\alpha}_i \hat{\omega}_i$ ($\hat{\alpha}_i$ is rotation angle, $\hat{\omega}_i$ is normalized axis). The rotation matrix $\hat{R}_i$ is recovered using Rodrigues' formula [S2, unofficial]:
        $$
        \hat{R}_i = I + \frac{\sin(\hat{\alpha}_i)}{\hat{\alpha}_i} (\hat{\phi}_i^\wedge) + \frac{1-\cos(\hat{\alpha}_i)}{\hat{\alpha}_i^2} (\hat{\phi}_i^\wedge)^2
        $$
        where $(\cdot)^\wedge$ is the skew-symmetric operator converting a vector to a matrix [S2, unofficial].
3.  **NeRF Model:** A Multi-Layer Perceptron (MLP) $F_\Theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ mapping 3D position $\mathbf{x}$ and view direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [S1]. The implementation uses a smaller network than the original NeRF, with all hidden layer dimensions shrunk by half, retaining the same positional encoding and skip connections [S2, unofficial].

**Algorithm Steps:**
The optimization minimizes the photometric loss between rendered images and ground truth images:
$$
\Theta^*, \Pi^* = \arg \min_{\Theta, \Pi} \mathcal{L}(\hat{I}, \hat{\Pi}|I)
$$
where $\mathcal{L}$ is the Mean Squared Error (MSE) [S1][S2, unofficial]:
$$
\mathcal{L}_{\text{MSE}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| C(\mathbf{r}) - \hat{C}(\mathbf{r}) \right\|_2^2
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

def rodrigues(phi):
    """
    Converts axis-angle vector phi to rotation matrix R.
    phi: (3,) tensor
    """
    alpha = torch.norm(phi)
    if alpha < 1e-8:
        return torch.eye(3)
    omega = phi / alpha
    w = torch.tensor([0, -omega[2], omega[1],
                      omega[2], 0, -omega[0],
                      -omega[1], omega[0], 0])
    w_mat = w.view(3, 3)
    R = torch.eye(3) + torch.sin(alpha) * w_mat + (1 - torch.cos(alpha)) * torch.matmul(w_mat, w_mat)
    return R

def construct_ray(phi_i, t_i, f, pixel_coords, image_dims):
    """
    Constructs ray direction and origin from camera parameters and pixel coordinates.
    phi_i: (3,) axis-angle rotation
    t_i: (3,) translation
    f: scalar focal length
    pixel_coords: (2,) (u, v)
    image_dims: (W, H)
    """
    W, H = image_dims
    u, v = pixel_coords
    # Ray direction in camera frame: (u - W/2)/f, -(v - H/2)/f, -1
    # Note: The source uses -(v - H/2)/f. Standard pinhole often uses (v - H/2)/f depending on coordinate system.
    # We follow the source formula exactly.
    ray_cam = torch.tensor([(u - W/2)/f, -(v - H/2)/f, -1.0])
    
    # Rotation matrix from axis-angle
    R = rodrigues(phi_i)
    
    # Transform to world frame
    ray_dir = torch.matmul(R, ray_cam)
    ray_dir = ray_dir / torch.norm(ray_dir) # Normalize to unit vector
    
    # Ray origin is the camera center (translation vector)
    ray_orig = t_i
    
    return ray_orig, ray_dir

def render_ray(ray_orig, ray_dir, nerf_model, near_plane, far_plane, num_samples):
    """
    Renders a single ray using the volume rendering equation.
    """
    # Sample points along the ray
    t_vals = torch.linspace(near_plane, far_plane, num_samples)
    points = ray_orig.unsqueeze(0) + t_vals.unsqueeze(1) * ray_dir.unsqueeze(0) # (1, N, 3)
    
    # Query NeRF
    # NeRF expects (x, d) -> (sigma, color)
    # We need to pass the view direction for each point. 
    # In standard NeRF, the view direction is constant for all samples on a ray.
    colors, densities = nerf_model(points, ray_dir.unsqueeze(0).expand_as(points))
    
    # Volume Rendering
    # delta_t is the distance between samples
    deltas = t_vals[1:] - t_vals[:-1]
    deltas = torch.cat([deltas, torch.tensor([1e10])]) # Last delta is large to ensure T_N -> 0
    
    # Transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    # C = sum T_i * alpha_i * c_i
    
    alpha = 1.0 - torch.exp(-densities * deltas)
    transmittance = torch.cumprod(torch.exp(-densities * deltas), dim=1)
    transmittance = torch.cat([torch.ones_like(transmittance[:, :1]), transmittance[:, :-1]], dim=1)
    
    weights = alpha * transmittance
    color = torch.sum(weights.unsqueeze(-1) * colors, dim=1)
    
    return color

def train_nerfmm(images, nerf_model, optimizer, num_epochs, num_samples_per_ray, near_plane, far_plane):
    """
    Joint optimization of camera parameters and NeRF.
    """
    N = len(images)
    H, W = images[0].shape[1], images[0].shape[2]
    
    # Initialize learnable camera parameters
    # phi_i: (N, 3), t_i: (N, 3), f: (1,)
    phi = nn.Parameter(torch.zeros(N, 3))
    t = nn.Parameter(torch.zeros(N, 3))
    f = nn.Parameter(torch.tensor([1.0]))
    
    # Set requires_grad to True
    phi.requires_grad = True
    t.requires_grad = True
    f.requires_grad = True
    
    # NeRF model parameters are already in nerf_model
    
    for epoch in range(num_epochs):
        total_loss = 0
        for i in range(N):
            img = images[i]
            # Get pixel coordinates for all pixels in the image
            # Flatten to list of (u, v)
            u_coords, v_coords = torch.meshgrid(torch.arange(W), torch.arange(H), indexing='ij')
            u_coords = u_coords.flatten().float()
            v_coords = v_coords.flatten().float()
            
            # Render all pixels for image i
            rendered_pixels = []
            for u, v in zip(u_coords, v_coords):
                ray_orig, ray_dir = construct_ray(phi[i], t[i], f, torch.tensor([u, v]), (W, H))
                color = render_ray(ray_orig, ray_dir, nerf_model, near_plane, far_plane, num_samples_per_ray)
                rendered_pixels.append(color)
            
            rendered_img = torch.stack(rendered_pixels).view(H, W, 3)
            
            # Compute Loss
            loss = torch.mean((rendered_img - img) ** 2)
            
            # Backpropagation
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
            
        print(f"Epoch {epoch}, Loss: {total_loss/N}")

    return nerf_model, phi, t, f
```

**Parameters:**
*   **NeRF Architecture:** Hidden layer dimensions are halved compared to the original NeRF [S2, unofficial].
*   **Training Duration:** Typically 3,000 epochs for perturbation experiments, though full training may use 10,000 epochs [S2, unofficial].
*   **Assumptions:** Forward-facing scenes (LLFF/NeRF style); all images share the same intrinsic parameters [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Track-to-Object Motion:** If the camera tracks an object, the object's position in the image remains nearly constant. This provides insufficient photometric supervision for blurry early-stage renders, leading to inaccurate joint optimization [S2, unofficial].
*   **Large Rotations:** The method initializes poses from identity matrices and assumes forward-facing scenes. It fails with rotation perturbations larger than $\pm 20^\circ$ or 360-degree scenes [S2, unofficial].
*   **Breaking Point:** Both NeRF-- and COLMAP fail when translation noise $> \pm 20\%$ and rotation noise $> \pm 20^\circ$ [S2, unofficial].
*   **Pose Ambiguity:** In cases like the "Roundtable" scene, COLMAP may predict poses with large rotation errors (e.g., $179.10^\circ$) due to mirror image ambiguity, yet still produce high-quality NVS if the geometry is self-consistent [S2, unofficial].

**Optional Refinement:**
After initial training, the NeRF model can be dropped and re-initialized with random parameters while keeping the pre-trained camera parameters. Re-running joint optimization with these initialized camera poses can lead to sharper images and improved synthesis [S2, unofficial].

## Subtopics

### [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md)
A spatial parameterization scheme that allows NeRF to work on 360° captures of unbounded scenes by separately modeling foreground and background, addressing the challenge of modeling unbounded 3D background content.

### [DoubleNeRF](7391-doublenerf.md)
A modified NeRF framework that represents a dynamic scene as a superposition of two or more neural radiance fields, specifically separating a static background scene and a moving object with a static shape.

## Operating characteristics
*   **Time Complexity:** Training is iterative and requires backpropagation through the ray construction and rendering steps for every pixel and epoch. The joint optimization increases memory usage compared to fixed-camera NeRF due to the gradient storage for camera parameters [S5].
*   **Memory Complexity:** $O(kn)$ for intermediate states during reverse-mode AD, where $n$ is pixels and $k$ is samples, plus storage for $N$ camera pose parameters [S5].
*   **Performance:** Achieves novel view synthesis quality comparable to COLMAP-enabled NeRF on forward-facing datasets (LLFF, BLEFF) [S2, unofficial].
*   **Strengths:** Eliminates the need for pre-computed camera parameters (SfM); provides globally consistent reconstruction similar to bundle adjustment [S2, unofficial]. Outperforms COLMAP in large translation perturbations [S2, unofficial].
*   **Limitations:** Fails on 360-degree scenes or large rotation perturbations ($> \pm 20^\circ$) [S2, unofficial]. Underperforms COLMAP in large rotation perturbations because feature descriptors handle appearance changes better than photometric methods [S2, unofficial]. Vulnerable to degenerate cases in translation where feature-based methods fail [S2, unofficial].

## Background
NeRF++ (NeRF--) was introduced in 2021 by Wang et al. [S2, unofficial]. It addresses the limitation of the original NeRF, which requires accurate camera poses and intrinsics typically obtained via COLMAP. The method was developed to simplify the training pipeline by removing the pre-computation step. It was evaluated on the LLFF-NeRF, RealEstate10K, and BLEFF (Blender Forward-Facing) datasets [S2, unofficial]. The work highlights the trade-offs between photometric joint optimization and traditional feature-based SfM, particularly in degenerate motion cases [S2, unofficial].

## Sources
[S1] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
[S2] NeRF--: Neural Radiance Fields Without Known Camera Parameters, https://doi.org/10.48550/arxiv.2102.07064, fetched 2026-10-26 (unofficial)
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
