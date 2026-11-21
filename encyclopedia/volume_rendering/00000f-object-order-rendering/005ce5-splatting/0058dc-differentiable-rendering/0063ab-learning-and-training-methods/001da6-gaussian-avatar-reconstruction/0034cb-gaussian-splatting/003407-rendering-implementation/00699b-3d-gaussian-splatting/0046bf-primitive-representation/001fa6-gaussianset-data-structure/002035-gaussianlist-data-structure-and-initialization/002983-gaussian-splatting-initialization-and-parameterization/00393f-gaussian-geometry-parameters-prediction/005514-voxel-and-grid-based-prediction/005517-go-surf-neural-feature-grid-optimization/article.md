# GO-Surf Neural Feature Grid Optimization

Also called: GO-Surf

## Overview
GO-Surf is a differentiable volume rendering method that optimizes a multi-resolution neural feature grid to reconstruct high-fidelity 3D surfaces from RGB-D observations. It solves the problem of fast, dense surface reconstruction by combining explicit grid-based feature storage with implicit SDF decoding, significantly reducing optimization time compared to fully implicit MLP-based methods like NeuralRGB-D. It sets itself apart by using a multi-resolution grid to store geometry and color features, allowing for faster convergence and higher detail capture, albeit at the cost of higher memory usage.

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $M$ images with corresponding depth maps $D[u, v]$ and camera poses (translation $t \in \mathbb{R}^3$ and rotation parameterized by Lie Algebra $\nu \in \mathfrak{so}(3)$).
*   A 3D bounding box defining the scene volume.

**Outputs:**
*   Optimized multi-resolution feature grids (geometry and color).
*   Optimized geometry and color decoder MLPs.
*   Optimized camera poses.
*   Rendered color and depth images.

### Data Structures

**Multi-Resolution Feature Grids:**
The method uses a set of 3D grids with different voxel sizes. Each grid stores feature vectors.
*   **Geometry Grids:** 4 levels. Voxel sizes: 3cm, 6cm, 24cm, 96cm.
    *   Feature dimension per level ($h_g$): 4.
    *   Total geometry feature dimension: 16.
*   **Color Grids:** 4 levels. Voxel sizes: 3cm, 6cm, 24cm, 96cm.
    *   Feature dimension per level ($h_c$): 6.
*   **Storage:** Trilinear interpolation is used to query features from these grids. A custom CUDA kernel is required to support double back-propagation for computing second-order derivatives (Hessian) of the trilinear interpolation.

**Decoders:**
*   **Geometry Decoder:** Lightweight MLP with 2 hidden layers, 32 neurons each. Takes the concatenated geometry feature vector (16D) as input and outputs the SDF value $\phi(x)$.
*   **Color Decoder:** Lightweight MLP with 2 hidden layers, 32 neurons each. Takes the concatenated color feature vector (6D) and viewing direction $d$ as input and outputs the RGB color $c$.

**Sampling Points:**
*   Points $x_i$ along a ray are sampled using importance sampling.
*   Coarse samples: $N_c = 96$.
*   Fine samples: $N_f = 36$.
*   Total samples per ray: $N = N_c + 3 \times 12 + N_f = 96 + 36 + 36 = 168$? No, the text says "starting from $N_c=96$ coarse samples we iteratively add 12 samples each time [3 times]... and $N_f=36$ fine samples". So $N = 96 + 36 + 36 = 168$? Or is $N_f$ the total fine? The text says "Nc=96 coarse samples and Nf=36 fine samples". The importance sampling adds 12 samples *each time* for 3 steps. So intermediate samples = $3 \times 12 = 36$. Total $N = 96 + 36 + 36 = 168$.

### Parameters
*   **Loss Weights:** $\lambda_{rgb} = 10.0$, $\lambda_{depth} = 1.0$, $\lambda_{sdf} = 10.0$, $\lambda_{fs} = 1.0$, $\lambda_{eik} = 1.0$, $\lambda_{smooth} = 1.0$.
*   **Truncation Threshold ($t$):** 16cm.
*   **Free Space Exponential Penalty ($\alpha$):** 5.
*   **Smoothness Perturbation Length ($\delta_s$):** Between 1mm and 4mm.
*   **Sigmoid Smoothness Parameter ($s$):** Learnable parameter controlling the transition smoothness of the SDF to opacity.
*   **Optimizer:** ADAM.
    *   Learning rate for MLP decoders: $1 \times 10^{-3}$.
    *   Learning rate for feature grids: $1 \times 10^{-2}$.
    *   Learning rate for camera poses: $5 \times 10^{-4}$.
*   **Batch Size:** $M = 6144$ rays per batch.
*   **Optimization Iterations:** 10,000.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Custom Grid Sampler supporting double back-propagation
# This replaces torch.nn.functional.grid_sample for the feature grids
def custom_grid_sample(features, coords):
    # Forward pass: trilinear interpolation
    # Backward pass: standard gradient
    # Second backward pass: custom CUDA kernel for Hessian of trilinear interpolation
    pass

class GeometryDecoder(nn.Module):
    def __init__(self):
        super().__init__()
        # 2 hidden layers, 32 neurons
        self.fc1 = nn.Linear(16, 32)
        self.fc2 = nn.Linear(32, 32)
        self.fc3 = nn.Linear(32, 1) # Output SDF

    def forward(self, x):
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        return self.fc3(x).squeeze(-1)

class ColorDecoder(nn.Module):
    def __init__(self):
        super().__init__()
        # 2 hidden layers, 32 neurons
        self.fc1 = nn.Linear(6, 32)
        self.fc2 = nn.Linear(32, 32)
        self.fc3 = nn.Linear(32, 3) # Output RGB

    def forward(self, x, d):
        # x is color feature (6D), d is viewing direction
        # Concatenate if necessary, but text implies input is feature vector
        # Assuming feature vector is passed directly
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        return self.fc3(x)

class GOSurf:
    def __init__(self, scene_bounds, voxel_sizes=[0.03, 0.06, 0.24, 0.96]):
        self.scene_bounds = scene_bounds
        self.voxel_sizes = voxel_sizes
        
        # Initialize Geometry Feature Grids (4 levels)
        # Dimensions depend on scene bounds and voxel sizes
        self.geo_features = [nn.Parameter(torch.rand(dims, 4)) for dims in self.get_grid_dims(scene_bounds, voxel_sizes)]
        
        # Initialize Color Feature Grids (4 levels)
        self.color_features = [nn.Parameter(torch.rand(dims, 6)) for dims in self.get_grid_dims(scene_bounds, voxel_sizes)]
        
        # Initialize Decoders
        self.geo_decoder = GeometryDecoder()
        self.color_decoder = ColorDecoder()
        
        # Initialize Camera Poses (t, nu)
        self.camera_poses = [nn.Parameter(torch.zeros(3)) for _ in range(num_cameras)] # t
        self.camera_rotations = [nn.Parameter(torch.zeros(3)) for _ in range(num_cameras)] # nu in so(3)
        
        # Learnable Sigmoid parameter
        self.s = nn.Parameter(torch.tensor(1.0))
        
        # Hyperparameters
        self.t = 0.16 # 16cm
        self.alpha = 5
        self.delta_s = 0.002 # 2mm average
        self.Nc = 96
        self.Nf = 36
        self.M = 6144
        self.lambda_rgb = 10.0
        self.lambda_depth = 1.0
        self.lambda_sdf = 10.0
        self.lambda_fs = 1.0
        self.lambda_eik = 1.0
        self.lambda_smooth = 1.0

    def get_grid_dims(self, bounds, voxel_sizes):
        # Calculate grid dimensions based on bounds and voxel sizes
        # Not given in sources, assume standard calculation: ceil((max-min)/voxel_size)
        pass

    def initialize_surface(self):
        # Geometric Initialization: Sphere centered at volume origin
        # Radius = half the size of the smallest dimension
        min_dim = min(self.scene_bounds.max - self.scene_bounds.min)
        radius = min_dim / 2.0
        # Initialize SDF such that phi(x) = ||x|| - radius
        # This is done by initializing the grid and decoder to approximate this
        # [G] Specific initialization details for grid values to match sphere are not given,
        # but the decoder output should be close to ||x|| - radius for all x.
        pass

    def query_sdf(self, x):
        # x: (N, 3) coordinates
        # Get features from multi-resolution grid
        features = []
        for grid in self.geo_features:
            # Normalize coordinates to [-1, 1] for grid sampling
            coords = self.normalize_coords(x)
            feat = custom_grid_sample(grid, coords)
            features.append(feat)
        geo_feat = torch.cat(features, dim=-1) # (N, 16)
        sdf = self.geo_decoder(geo_feat)
        return sdf

    def query_color(self, x, d):
        features = []
        for grid in self.color_features:
            coords = self.normalize_coords(x)
            feat = custom_grid_sample(grid, coords)
            features.append(feat)
        color_feat = torch.cat(features, dim=-1) # (N, 6)
        color = self.color_decoder(color_feat, d)
        return color

    def normalize_coords(self, x):
        # Map x from scene bounds to [-1, 1]
        # Not given in sources, assume standard normalization
        pass

    def sample_points(self, rays_o, rays_d):
        # rays_o: (M, 3), rays_d: (M, 3)
        # Importance sampling
        # 1. Coarse sampling: Nc = 96 points uniformly distributed along ray within bounds
        # 2. Compute weights from coarse samples
        # 3. Add 12 samples per step for 3 steps based on weights
        # 4. Fine sampling: Nf = 36 points
        # Return all points x, depths d, and weights
        pass

    def render(self, rays_o, rays_d):
        # Sample points
        points, depths, weights = self.sample_points(rays_o, rays_d)
        
        # Query SDF and Color
        sdf = self.query_sdf(points)
        colors = self.query_color(points, rays_d)
        
        # Compute Opacity
        # alpha_i = max((sigma_s(phi(x_i)) - sigma_s(phi(x_{i+1}))) / sigma_s(phi(x_i)), 0)
        # sigma_s(x) = 1 / (1 + exp(-s * x))
        s_val = torch.sigmoid(-self.s * sdf) # sigma_s(phi)
        s_val_next = torch.sigmoid(-self.s * torch.roll(sdf, -1))
        
        # Handle last point
        s_val_next[-1] = 0 # Or some small value
        
        alpha = torch.clamp((s_val - s_val_next) / s_val, min=0)
        
        # Compute Transmittance
        # T_i = product_{j=1}^{i-1} (1 - alpha_j)
        T = torch.cumprod(1 - alpha, dim=0)
        T = torch.cat([torch.ones_like(T[:1]), T[:-1]], dim=0)
        
        # Weights
        w = T * alpha
        
        # Render Color and Depth
        rendered_color = torch.sum(w.unsqueeze(-1) * colors, dim=0)
        rendered_depth = torch.sum(w * depths, dim=0)
        
        return rendered_color, rendered_depth, points, weights

    def compute_losses(self, rays_o, rays_d, gt_color, gt_depth):
        # Render
        pred_color, pred_depth, points, weights = self.render(rays_o, rays_d)
        
        # RGB Loss
        l_rgb = torch.mean(torch.abs(gt_color - pred_color))
        
        # Depth Loss (only for valid depth pixels)
        valid_mask = gt_depth > 0
        l_depth = torch.mean(torch.abs(gt_depth[valid_mask] - pred_depth[valid_mask]))
        
        # SDF Supervision
        # Approximate SDF bound: b(x) = D[u,v] - d
        # For each point, find the corresponding depth measurement
        # This requires mapping points back to image coordinates
        # [G] Implementation detail: Project points to image plane to get D[u,v]
        # b = gt_depth - points_depth_along_ray
        
        # SDF Loss (near surface)
        # |D[u,v] - d| <= t
        near_surface_mask = torch.abs(gt_depth - points[:, 2]) <= self.t # Assuming z is depth
        # l_sdf(x) = |phi(x) - b(x)|
        b = gt_depth - points[:, 2] # Simplified, need actual depth value at ray intersection
        # [G] b(x) is the distance to the observed depth surface along the ray
        # If point is on ray, b(x) = D[u,v] - d_ray_point
        # Let's assume we have b_values for each point
        l_sdf = torch.mean(torch.abs(points_sdf[near_surface_mask] - b_values[near_surface_mask]))
        
        # Free Space Loss (far from surface)
        # |D[u,v] - d| > t
        far_surface_mask = ~near_surface_mask
        # l_fs(x) = max(0, exp(-alpha * phi(x)) - 1, phi(x) - b(x))
        # Applied when |D[u,v] - d| > t
        # [G] b(x) is still the bound
        term1 = torch.exp(-self.alpha * points_sdf[far_surface_mask]) - 1
        term2 = points_sdf[far_surface_mask] - b_values[far_surface_mask]
        l_fs = torch.mean(torch.clamp(torch.stack([torch.zeros_like(term1), term1, term2], dim=-1), min=0).sum(dim=-1))
        
        # Eikonal Regularization
        # l_eik(x) = (1 - ||grad phi(x)||)^2
        # Applied on S_fs (far surface points)
        # Requires gradient of SDF w.r.t x
        # [G] Use torch.autograd.grad to compute gradient
        grad_phi = torch.autograd.grad(points_sdf[far_surface_mask], points[far_surface_mask], 
                                       grad_outputs=torch.ones_like(points_sdf[far_surface_mask]), 
                                       create_graph=True)[0]
        l_eik = torch.mean((1 - torch.norm(grad_phi, dim=-1))**2)
        
        # Smoothness Regularization
        # l_smooth(x) = ||grad phi(x) - grad phi(x + eps)||^2
        # Applied on near-surface points
        # [G] Sample random perturbation eps
        eps = torch.randn_like(points[near_surface_mask]) * self.delta_s
        points_perturbed = points[near_surface_mask] + eps
        grad_phi_perturbed = torch.autograd.grad(self.query_sdf(points_perturbed), points_perturbed, 
                                                 grad_outputs=torch.ones_like(self.query_sdf(points_perturbed)), 
                                                 create_graph=True)[0]
        grad_phi_near = torch.autograd.grad(points_sdf[near_surface_mask], points[near_surface_mask], 
                                            grad_outputs=torch.ones_like(points_sdf[near_surface_mask]), 
                                            create_graph=True)[0]
        l_smooth = torch.mean((grad_phi_near - grad_phi_perturbed)**2)
        
        # Total Loss
        loss = (self.lambda_rgb * l_rgb + 
                self.lambda_depth * l_depth + 
                self.lambda_sdf * l_sdf + 
                self.lambda_fs * l_fs + 
                self.lambda_eik * l_eik + 
                self.lambda_smooth * l_smooth)
        
        return loss

    def optimize(self, images, depths, poses):
        # Initialize surface
        self.initialize_surface()
        
        # Optimizer setup
        optimizer = torch.optim.Adam([
            {'params': self.geo_decoder.parameters(), 'lr': 1e-3},
            {'params': self.color_decoder.parameters(), 'lr': 1e-3},
            {'params': self.geo_features, 'lr': 1e-2},
            {'params': self.color_features, 'lr': 1e-2},
            {'params': self.camera_poses, 'lr': 5e-4},
            {'params': self.camera_rotations, 'lr': 5e-4},
            {'params': [self.s], 'lr': 1e-3}
        ])
        
        for i in range(10000):
            # Sample batch of rays
            rays_o, rays_d, gt_color, gt_depth = self.sample_batch_rays(images, depths, poses)
            
            # Compute loss
            loss = self.compute_losses(rays_o, rays_d, gt_color, gt_depth)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            # Update camera poses
            # [G] Camera poses are optimized jointly
            pass
```

### Edge Cases and Pitfalls
*   **Second-Order Derivatives:** Computing the Eikonal and smoothness losses requires the Hessian of the SDF. Standard PyTorch autograd does not support double back-propagation through the standard grid sampler for trilinear interpolation. A custom CUDA kernel must be implemented for the second backward pass.
*   **Initialization:** Poor initialization can lead to local minima. Initializing the SDF to a sphere centered at the volume origin with radius half the smallest dimension helps convergence.
*   **Depth Noise:** RGB-D sensors often have noisy depth or missing values (e.g., glass). The SDF regularization and RGB loss help mitigate these artifacts.

## Operating characteristics

**Time Complexity:**
*   Optimization takes 15 to 45 minutes depending on scene size.
*   Significantly faster than NeuralRGB-D (15-25 hours).

**Memory Complexity:**
*   Model size scales with scene resolution.
*   Example: 268 MB for 8.8x9.1x3.4m scene (66.5M params).
*   41 MB for synthetic scene (10.1M params).
*   Larger memory footprint than implicit MLP methods.

**Strengths:**
*   Fast optimization.
*   High-fidelity surface reconstruction.
*   Good hole-filling ability.
*   Captures thin structures.

**Limitations:**
*   High memory usage.
*   Requires custom CUDA kernel for training.

**When to Use:**
*   When fast reconstruction is required.
*   When high-fidelity surface is needed.
*   When memory is not a constraint.

**When Not to Use:**
*   When memory is very limited.
*   When implicit representation is preferred for scalability.

## Background

GO-Surf was introduced in the paper "GO-Surf: Neural Feature Grid Optimization for Fast, High-Fidelity RGB-D Surface Reconstruction" [S1]. It builds on the idea of using neural feature grids for scene representation, combining the speed of explicit grids with the flexibility of implicit decoding. It is an improvement over NeuralRGB-D, which uses a single MLP for the entire scene, resulting in slower optimization. GO-Surf uses a multi-resolution grid to store features, allowing for faster convergence and better detail capture.

## Sources

[S1] GO-Surf: Neural Feature Grid Optimization for Fast, High-Fidelity RGB-D Surface Reconstruction, https://doi.org/10.1109/3dv57658.2022.00055, fetched 2026-10-29
