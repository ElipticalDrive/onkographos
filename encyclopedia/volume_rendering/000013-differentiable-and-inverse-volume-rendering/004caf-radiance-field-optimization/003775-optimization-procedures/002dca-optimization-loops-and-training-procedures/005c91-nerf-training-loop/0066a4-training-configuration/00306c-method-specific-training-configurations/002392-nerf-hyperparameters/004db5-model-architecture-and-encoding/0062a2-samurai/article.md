# SAMURAI

Also called: Joint optimization of shape, BRDF, cameras, and illuminations, Shape and Material from Unconstrained Real-World Arbitrary Image Collections

## Overview
SAMURAI is a joint optimization framework that estimates 3D shape, BRDF material properties, per-image illumination, camera poses, and camera intrinsics from unconstrained in-the-wild image collections with varying backgrounds and illuminations [S1]. It addresses the limitations of prior methods like NeRF and Neural-PIL by handling imperfect segmentation masks, arbitrary camera distances, and multiple illumination conditions without requiring correspondences or GAN-style training [S1]. Unlike [SUDS](14964-suds.md) which focuses on differentiable rendering for inverse problems, SAMURAI specifically targets the decomposition of radiance into shape, material, and illumination from single-view-like unconstrained data.

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   A collection of $q$ object images $C_j \in \mathbb{R}^{s_j \times 3}$ with varying resolutions, backgrounds, and illuminations [S1].
    *   Rough camera pose quadrants annotated with 3 binary questions: Left vs. Right, Above vs. Below, Front vs. Back [S1].
    *   Automatically estimated foreground segmentation masks $M_j \in \{0, 1\}^{s \times 1}$ (e.g., via U2-Net), which may be noisy [S1].
*   **Outputs:**
    *   A 3D neural volume representing shape and material.
    *   Explicit meshes with BRDF textures extracted from the volume [S1].
    *   Per-image camera poses and intrinsics [S1].
    *   Per-image latent illumination vectors [S1].

**Data Structures and Network Architecture**
The method uses a single fine/decomposition network (no coarse network) implemented as an MLP with 8 layers of 128 features [S1].
*   **Input Encoding:** Input coordinates $\mathbf{x}$ and view directions $\mathbf{d}$ are processed using hierarchical positional encoding $\gamma(\cdot)$ [S1]. To prevent axis-aligned artifacts, Gaussian distributed noise is added to the Fourier embedding frequencies as offsets to the logarithmically spaced frequencies [S1].
*   **Output Heads:**
    *   **Density:** A layer with 1 feature output predicts volume density $\sigma \in \mathbb{R}$ [S1].
    *   **BRDF/Normal:** The main network features are compressed to 16 features and expanded to predict BRDF parameters $\mathbf{b} \in \mathbb{R}^5$ (base color $\mathbf{bc} \in \mathbb{R}^3$, metallic $\mathbf{bm} \in \mathbb{R}^1$, roughness $\mathbf{br} \in \mathbb{R}$) and unit-length surface normal $\mathbf{n} \in \mathbb{R}^3$ [S1].
    *   **Radiance:** An MLP with a hidden layer predicts view-dependent radiance $\tilde{\mathbf{c}}$ [S1].
*   **Illumination:** Per-image latent illumination vectors $\mathbf{z}_{jl} \in \mathbb{R}^{128}$ are estimated [S1].
*   **Camera Parameterization:** Uses a 'look-at' parameterization stored as offsets to initial parameters: 3D look-at vector $\mathbf{r}_j \in \mathbb{R}^3$, scalar up rotation $u_j \in [-\pi, \pi]$, 3D camera position $\mathbf{t}_j \in \mathbb{R}^3$, and focal length $f_j \in \mathbb{R}$ [S1]. Vertical focal length offsets are compressed as $\hat{f}_j = f_j / \sqrt{h}$, where $h$ is image height [S1].

**Ray Sampling and Bounds**
*   **Sampling:** 128 points are sampled along the ray in fixed steps [S1].
*   **Bounds:** Unlike fixed near/far planes, bounds are defined by a world-space sphere with radius $v$ (diameter 1). For a camera origin $\mathbf{o}$, the near bound is $|\mathbf{o}| - v$ and the far bound is $|\mathbf{o}| + v$ [S1]. This allows arbitrary camera distances [S1].

**Optimization Steps**
1.  **Initialization:**
    *   Initialize camera poses based on quadrant annotations and an initial Field of View of 53.13 degrees [S1].
    *   Initialize Fourier frequencies with logarithmic spacing plus Gaussian noise offsets [S1].
2.  **Camera Multiplex Optimization:**
    *   For each image, optimize 4 randomly jittered poses around the quadrant center [S1].
    *   Compute mask reconstruction loss $L_{Mask_{ij}}$ for each camera $i$ in the multiplex for image $j$ [S1].
    *   Calculate multiplex weights $\mathbf{S}_j \in \mathbb{R}^4$:
        $$ \mathbf{S}_j = \text{softmax}(-\lambda_s L_{Mask_{ij}}) $$
        where $\lambda_s$ is a scalar gradually increased during optimization [S1].
    *   Update weights $\mathbf{S}_j$ using a memory bank and momentum across batches [S1].
    *   Dynamically re-weight network losses:
        $$ L_{Network_j} = \sum_i S_{ji} L_{Network_{ij}} $$
        [S1].
3.  **Posterior Scaling:**
    *   Maintain a circular buffer of ~1000 elements containing recent mask and rendered image losses [S1].
    *   Calculate mean $\mu_l$ and standard deviation $\sigma_l$ of losses [S1].
    *   Compute loss scalar $sp_j$:
        $$ sp_j = \max\left(\tanh\left(\frac{\mu_l - (L_{Mask_j} + L_{Image_j})}{\sigma_l}\right) + 1, 1\right) $$
        [S1].
    *   Apply scaling: $L_{Network_j} = sp_j L_{Network_j}$ [S1].
4.  **Network and Camera Update:**
    *   Optimize networks using Adam (LR $10^{-4}$, decay order of magnitude every 300k steps) [S1].
    *   Optimize cameras using Adam (LR $3 \times 10^{-3}$, decay order of magnitude every 70k steps) [S1].
    *   Camera optimization uses the same loss structure but does *not* apply posterior scaling, allowing badly initialized poses to recover [S1].
5.  **Scheduling:**
    *   Use fading variables $\lambda_b$ (fade direct color to BRDF), $\lambda_a$ (scale specific losses), and $\lambda_c$ (increase resolution, reduce active multiplex cameras) [S1].
    *   Perform Fourier frequency annealing (BARF) in early stages [S1].
    *   Delay focal length optimization to later stages [S1].

**Loss Functions**
The total network loss is:
$$ L_{Network} = \lambda_b L_{Image}(C_s, \tilde{\mathbf{c}}) + (1 - \lambda_b) L_{Image}(C_s, \hat{\mathbf{c}}) + L_{Mask} + \lambda_a L_{Init} + \lambda_{ndir} L_{ndir} + \lambda_{Smooth} L_{Smooth} + \lambda_{Dec\_Smooth} L_{Dec\_Smooth} + \lambda_{Dec\_Sparsity} L_{Dec\_Sparsity} $$
[S1].

*   **Image Loss:** Charbonnier loss $L_{Image}(g, p) = \sqrt{(g - p)^2 + 0.001^2}$ between input color $C_s$ and predicted color [S1].
*   **Mask Loss:** $L_{Mask} = L_{BCE} + L_{Background}$, where $L_{BCE}$ is binary cross-entropy and $L_{Background}$ forces background samples to 0 (from NeRD) [S1].
*   **Initialization Loss:** $L_{Init} = L_{MSE}(C_s, \mathbf{bc})$ to prevent illumination local minima [S1].
*   **Normal Direction Loss:** $L_{ndir}$ constrains normals to face the camera [S1].
*   **BRDF Sparsity:** $L_{Dec\_Sparsity} = \frac{1}{N} \sum_i |e_i|$, where $e_i$ is the BRDF embedding [S1].
*   **BRDF Smoothness:** $L_{Dec\_Smooth} = \frac{1}{N} \sum_i |f(\theta; e_i) - f(\theta; e_i + \epsilon)|$, where $\epsilon \sim \mathcal{N}(0, 0.01^2)$ [S1].
*   **Camera Constraints:**
    *   $L_{lookat}$: L1 loss on look-at vector $\mathbf{r}_j$ [S1].
    *   $L_{Bounds} = \max((v - |\mathbf{t}_j|)^2, 0)$ to prevent cameras from going too far into the volume bound [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim

class SAMURAI:
    def __init__(self, images, masks, quadrants, num_images, num_rays_per_image=128):
        self.num_images = num_images
        self.num_rays_per_image = num_rays_per_image
        self.images = images
        self.masks = masks
        self.quadrants = quadrants
        
        # Initialize Network (8 layers, 128 features)
        self.network = FineDecompositionNetwork()
        
        # Initialize Camera Parameters (Look-at parameterization)
        # r_j: look-at vector, t_j: position, u_j: up rotation, f_j: focal length
        self.camera_params = self.initialize_cameras(quadrants, fov=53.13)
        
        # Multiplex: 4 jittered poses per image
        self.num_multiplex = 4
        self.camera_multiplex = self.initialize_multiplex(self.camera_params)
        
        # Multiplex weights and memory bank
        self.S = torch.ones(num_images, self.num_multiplex) / self.num_multiplex
        self.loss_buffer = []
        
        # Optimization parameters
        self.optimizer_net = optim.Adam(self.network.parameters(), lr=1e-4)
        self.optimizer_cam = optim.Adam(self.camera_params.parameters(), lr=3e-3)
        
        # Scheduling variables
        self.lambda_s = 0.0
        self.lambda_b = 1.0
        self.lambda_a = 1.0
        self.lambda_c = 1.0
        
        # Volume bound radius
        self.v = 0.5  # Diameter 1

    def initialize_cameras(self, quadrants, fov):
        # Initialize based on quadrants (Left/Right, Above/Below, Front/Back)
        # and initial FOV. Returns parameters stored as offsets.
        pass # Implementation details for quadrant-to-pose mapping [S1]

    def initialize_multiplex(self, base_params):
        # Create 4 jittered poses around base params for each image
        multiplex = []
        for i in range(self.num_images):
            poses = []
            for _ in range(self.num_multiplex):
                pose = base_params[i] + torch.randn_like(base_params[i]) * 0.1
                poses.append(pose)
            multiplex.append(poses)
        return multiplex

    def forward(self, ray_origins, ray_directions, camera_params):
        # Sample 128 points along ray
        # Bounds: near = |o| - v, far = |o| + v
        t_vals = torch.linspace(0, 1, 128)
        points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(2)
        
        # Encode inputs
        x_encoded = positional_encoding(points)
        d_encoded = positional_encoding(ray_directions)
        
        # Network forward pass
        density, brdf_params, normal, radiance = self.network(x_encoded, d_encoded)
        
        # Volume rendering
        color, alpha = volume_render(points, density, radiance, ray_directions)
        return color, alpha, density, brdf_params, normal

    def compute_losses(self, predicted_color, predicted_alpha, brdf_params, normal, 
                       image, mask, camera_params, multiplex_idx):
        # Image Loss (Charbonnier)
        L_image = torch.sqrt((predicted_color - image)**2 + 0.001**2)
        
        # Mask Loss (BCE + Background)
        L_bce = binary_cross_entropy(predicted_alpha, mask)
        L_bg = background_loss(predicted_alpha, mask)
        L_mask = L_bce + L_bg
        
        # BRDF Losses
        L_init = mse(brdf_params.base_color, image.mean(dim=(0,1)))
        L_dec_sparsity = torch.mean(torch.abs(brdf_params.embedding))
        L_dec_smooth = smoothness_loss(brdf_params, epsilon=0.01)
        
        # Normal Direction Loss
        L_ndir = normal_direction_loss(normal, ray_directions)
        
        # Camera Losses
        L_lookat = l1_loss(camera_params.look_at, target_look_at)
        L_bounds = max((self.v - torch.norm(camera_params.position))**2, 0)
        
        # Total Network Loss
        L_net = (self.lambda_b * L_image + 
                 (1 - self.lambda_b) * L_image_direct + # Direct color for stabilization
                 L_mask + 
                 self.lambda_a * L_init + 
                 L_ndir + 
                 L_dec_smooth + 
                 L_dec_sparsity)
        
        return L_net, L_mask

    def step(self, batch_images, batch_masks, batch_quadrants, step_count):
        # 1. Update Scheduling
        self.lambda_s = min(self.lambda_s + 0.001, 10.0) # Gradually increase
        # Update lambda_b, lambda_a, lambda_c based on step_count [S1]
        
        # 2. Camera Multiplex Weight Update
        L_mask_multiplex = []
        for j in range(self.num_images):
            for i in range(self.num_multiplex):
                cam_params = self.camera_multiplex[j][i]
                # Render rays for this camera
                color, alpha, _, _, _ = self.forward(...)
                L_mask, _ = self.compute_losses(...)
                L_mask_multiplex.append(L_mask)
        
        # Calculate weights S_j
        L_mask_tensor = torch.stack(L_mask_multiplex).view(self.num_images, self.num_multiplex)
        self.S = torch.softmax(-self.lambda_s * L_mask_tensor, dim=1)
        
        # Update memory bank and momentum for S (simplified)
        self.update_multiplex_memory()
        
        # 3. Posterior Scaling
        # Calculate recent loss stats from buffer
        mean_loss, std_loss = self.get_loss_stats()
        sp = torch.clamp(torch.tanh((mean_loss - L_mask_tensor.mean(dim=1)) / std_loss) + 1, min=1.0)
        
        # 4. Network Optimization
        self.optimizer_net.zero_grad()
        total_loss = 0
        for j in range(self.num_images):
            for i in range(self.num_multiplex):
                # Only optimize active cameras (controlled by lambda_c)
                if torch.rand(1) < self.lambda_c:
                    cam_params = self.camera_multiplex[j][i]
                    # Forward pass
                    color, alpha, density, brdf, normal = self.forward(...)
                    
                    # Compute losses
                    L_net, L_mask = self.compute_losses(...)
                    
                    # Apply multiplex weight and posterior scaling
                    L_net = L_net * self.S[j, i] * sp[j]
                    
                    total_loss += L_net
            
        total_loss.backward()
        self.optimizer_net.step()
        
        # 5. Camera Optimization
        self.optimizer_cam.zero_grad()
        # Do NOT apply posterior scaling for camera optimization [S1]
        total_cam_loss = 0
        for j in range(self.num_images):
            for i in range(self.num_multiplex):
                cam_params = self.camera_multiplex[j][i]
                # Forward pass with current cam params
                color, alpha, _, _, _ = self.forward(...)
                L_net, _ = self.compute_losses(...)
                
                # Add camera specific losses
                L_cam = L_net + L_lookat + L_bounds
                
                total_cam_loss += L_cam.mean() # Mean loss for recovery [S1]
        
        total_cam_loss.backward()
        self.optimizer_cam.step()

    def update_multiplex_memory(self):
        # Update memory bank with recent losses and momentum for S
        pass

    def extract_mesh(self):
        # Extract explicit mesh with BRDF texture from the learned neural volume
        pass
```

## Operating characteristics
*   **Time Complexity:** Rendering samples 128 points per ray. Optimization involves multiple cameras per image (multiplex), increasing memory and compute per step compared to single-camera NeRF. Adam optimizer is used for both network and cameras [S1].
*   **Memory Complexity:** Stores 4 camera poses per image in the multiplex. Requires a circular buffer of ~1000 elements for posterior scaling statistics [S1].
*   **Performance:** Handles in-the-wild images with varying illumination and background. Outperforms BARF on coarse poses and NeRD/Neural-PIL on unconstrained datasets [S1].
*   **Strengths:** Robust to noisy masks, arbitrary camera distances, and multiple illuminations. Does not require correspondences or GAN training [S1].
*   **Limitations:** Requires manual quadrant annotation (3-5 minutes per 80 images). Optimization is complex due to multiplex and posterior scaling [S1].

## Background
SAMURAI was proposed to tackle the challenge of estimating 3D shape, BRDF, and per-image camera pose and illumination from unconstrained in-the-wild online image collections [S1]. It addresses limitations of NeRF (fixed near/far planes, no BRDF) and Neural-PIL (requires perfect camera poses and segmentation) [S1]. It is distinct from SCNeRF (no correspondences), BARF (handles extremely coarse poses), NeRF-- (no double training), and GNeRF (no GAN) [S1].

## Sources
[S1] SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections, https://doi.org/10.52202/068431-1914, fetched 2026-10-29
