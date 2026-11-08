# D-NeuS

Also called: Details recovering Neural implicit Surface reconstruction

## Overview
D-NeuS is a volume rendering-based neural implicit surface reconstruction method that recovers fine-level geometric details by extending NeuS with two additional loss functions: a geometry bias loss and a multi-view feature consistency loss [S1, unofficial]. It addresses the ambiguity in geometry and appearance representation inherent in volume rendering by ensuring consistency between the rendered surface points and the underlying signed distance field (SDF) zero-crossings [S1, unofficial]. Unlike NeuS, which may suffer from geometry bias due to the transformation from SDF to density, D-NeuS explicitly penalizes this bias and enforces multi-view feature consistency on surface points derived via linear interpolation of SDF zero-crossings, rather than using patch-based photometric consistency on all sampled points [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view RGB images with known camera poses.
*   **Outputs:** A reconstructed 3D mesh extracted from the learned SDF field, and novel view synthesis images.

**Data Structures:**
1.  **Geometry Network ($f$):** An MLP mapping $\mathbb{R}^3 \to \mathbb{R}$.
    *   **Layout:** 8 hidden layers, each with 256 units.
    *   **Skip Connections:** Input is concatenated to the output of the 4th hidden layer.
    *   **Initialization:** Weights initialized to approximate the SDF of a unit sphere [S1, unofficial].
2.  **Radiance Network ($g$):** An MLP mapping spatial position, view direction, normal, and feature vector to color $\mathbb{R}^3$.
    *   **Layout:** 4 hidden layers, each with 256 units.
    *   **Input:** Spatial point $\mathbf{x}$, view direction $\mathbf{v}$, normal $\mathbf{n} = \nabla f(\mathbf{x})$, and feature vector $\mathbf{z}$ from $f$ [S1, unofficial].
3.  **Position Encoding:**
    *   Spatial coordinates $\mathbf{x}$: 6 frequencies.
    *   View direction $\mathbf{v}$: 4 frequencies [S1, unofficial].
4.  **Feature Consistency:**
    *   Feature channels: $N_c = 32$.
    *   Neighboring views: $N_v = 2$ [S1, unofficial].

**Parameters:**
*   **Sigmoid Sharpness ($s$):** A trainable parameter in the density transformation. Initialized to $1/0.3 \approx 3.33$ (since standard deviation $\sigma = s^{-1} = 0.3$) [S1, unofficial].
*   **Optimizer:** Adam.
    *   Learning Rate: $5 \times 10^{-4}$.
    *   Warm-up: 5,000 iterations.
    *   Decay: Cosine decay to minimum $2.5 \times 10^{-5}$ [S1, unofficial].
*   **Loss Weights:**
    *   Eikonal weight ($\alpha$): Fixed at 0.1 throughout training [S1, unofficial].
    *   Geometry Bias weight ($\beta$):
        *   Stage 1 (0–50k iters): 0.01.
        *   Stage 2 (50k–150k iters): 0.1.
        *   Stage 3 (150k–300k iters): 0.01.
    *   Feature Consistency weight ($\gamma$):
        *   Stage 1: 0.
        *   Stage 2: 0.5.
        *   Stage 3: 0.05 [S1, unofficial].

**Numbered Steps:**

1.  **Ray Sampling:** For each iteration, sample 512 rays. For each ray, generate 64 coarse points and 64 fine points within the scene, plus 32 additional points outside the unit sphere [S1, unofficial].
2.  **SDF Evaluation:** Query the geometry network $f(\mathbf{x})$ for all sampled points to obtain SDF values [S1, unofficial].
3.  **Density Transformation:** Compute the opaque density $\sigma(t)$ for each point $t$ along the ray using the derivative of the sigmoid function applied to the SDF:
    $$ \sigma(t) = \max \left( -\frac{d\Phi_s}{dt}(f(\mathbf{x}(t))), 0 \right) $$
    where $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1, unofficial].
4.  **Volume Rendering:** Perform standard volume rendering (alpha compositing) using the computed densities to render the color image [S1, unofficial].
5.  **Surface Point Interpolation:** Identify the surface point along each ray by finding the first zero-crossing of the SDF values from the sampled points. Use locally differentiable linear interpolation between the two adjacent sampled points where the sign of the SDF changes [S1, unofficial].
6.  **Geometry Bias Loss Calculation:**
    *   Generate additional distance maps during rendering.
    *   Back-project these distances to 3D points.
    *   Compute the absolute SDF values predicted by $f$ at these back-projected points.
    *   Minimize the difference between the back-projected distance and the absolute SDF value to encourage consistency between the volume-rendered surface and the implicit surface [S1, unofficial].
7.  **Multi-View Feature Consistency Loss Calculation:**
    *   Extract feature vectors from the geometry network $f$ at the interpolated surface points.
    *   Compare features of the reference view with features from $N_v=2$ neighboring source views at corresponding surface points.
    *   Use CNNs for robust feature representation [S1, unofficial].
8.  **Eikonal Loss:** Apply the Eikonal constraint $\|\nabla f(\mathbf{x})\| = 1$ with weight $\alpha = 0.1$ [S1, unofficial].
9.  **Optimization:** Update network weights using the combined loss function over 300,000 iterations [S1, unofficial].
10. **Mesh Extraction:** After training, extract the mesh using Marching Cubes on the SDF field $f$ within a bounding box of size $512^3$ voxels [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GeometryNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # 8 hidden layers, size 256
        self.layers = nn.ModuleList([nn.Linear(256, 256) for _ in range(8)])
        # Skip connection from input to 4th layer output
        self.skip_layer_idx = 3 # 0-indexed, so 4th layer is index 3
        
    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers):
            h = F.relu(layer(h))
            if i == self.skip_layer_idx:
                h = torch.cat([h, x], dim=-1)
        return h

class RadianceNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # 4 hidden layers, size 256
        self.layers = nn.ModuleList([nn.Linear(256, 256) for _ in range(4)])
        
    def forward(self, x, v, n, z):
        # Concatenate inputs: x, v, n, z
        # Note: Exact input dimensionality depends on position encoding
        # Assuming inputs are already encoded
        h = torch.cat([x, v, n, z], dim=-1)
        for layer in self.layers:
            h = F.relu(layer(h))
        return h

def sigmoid_derivative(x, s):
    """
    Computes -d/dx Phi_s(x) where Phi_s(x) = (1 + exp(-sx))^-1
    d/dx Phi_s(x) = s * exp(-sx) * (1 + exp(-sx))^-2
    So -d/dx Phi_s(x) = -s * exp(-sx) * (1 + exp(-sx))^-2
    """
    exp_neg_sx = torch.exp(-s * x)
    sigmoid_val = 1.0 / (1.0 + exp_neg_sx)
    return -s * exp_neg_sx * sigmoid_val * sigmoid_val

def compute_density(sdf_values, s):
    """
    Computes density sigma(t) = max(-dPhi_s/dt(f(x(t))), 0)
    """
    # s is a scalar or tensor of shape matching sdf_values
    # Note: In practice, s is a trainable parameter
    deriv = sigmoid_derivative(sdf_values, s)
    return torch.clamp(deriv, min=0.0)

def find_zero_crossing(sdf_values, t_values):
    """
    Finds the first zero crossing via linear interpolation.
    sdf_values: tensor of SDF values along the ray.
    t_values: tensor of distances along the ray.
    Returns: interpolated t_surface, interpolated sdf (should be ~0)
    """
    # Find indices where sign changes
    signs = torch.sign(sdf_values)
    # Check for sign change between i and i+1
    # We look for the first occurrence
    # Note: This is a simplified implementation. 
    # In practice, one might need to handle exact zeros or multiple crossings.
    
    # Create mask for sign change
    # signs[i] != signs[i+1]
    # Handle edge cases where sign is 0
    sign_changes = (signs[:-1] * signs[1:] < 0) | (signs[:-1] == 0) | (signs[1:] == 0)
    
    if not torch.any(sign_changes):
        return None, None
        
    # Get first index
    idx = torch.argmax(sign_changes.int())
    
    t1 = t_values[idx]
    t2 = t_values[idx+1]
    s1 = sdf_values[idx]
    s2 = sdf_values[idx+1]
    
    # Linear interpolation
    # t_surface = t1 - s1 * (t2 - t1) / (s2 - s1)
    denom = s2 - s1
    # Avoid division by zero
    denom = torch.where(denom == 0, torch.ones_like(denom) * 1e-8, denom)
    t_surface = t1 - s1 * (t2 - t1) / denom
    
    return t_surface, 0.0 # SDF at surface is 0

def volume_render(rays_o, rays_d, geometry_net, radiance_net, s, num_coarse=64, num_fine=64, num_outside=32):
    """
    Performs volume rendering for a batch of rays.
    Returns: rendered color, depth, and intermediate values for loss computation.
    """
    # Sample points
    # Coarse sampling
    t_coarse = torch.linspace(0, 1, num_coarse).to(rays_o.device) # Placeholder for actual sampling strategy
    # Note: NeuS uses hierarchical sampling. This is a simplified representation.
    # Actual implementation would involve sampling from the CDF of the coarse density.
    
    # For the purpose of this article, we assume standard hierarchical sampling as in NeuS
    # is handled by the framework. We focus on the D-NeuS specific additions.
    
    # 1. Evaluate SDF at coarse points
    # x_coarse = rays_o + rays_d * t_coarse
    # sdf_coarse = geometry_net(x_coarse)
    
    # 2. Compute Density
    # sigma_coarse = compute_density(sdf_coarse, s)
    
    # 3. Render Coarse Color
    # color_coarse, alpha_coarse, weights_coarse = standard_volume_render(...)
    
    # 4. Fine Sampling (based on coarse weights)
    # t_fine = ...
    # sdf_fine = geometry_net(x_fine)
    # sigma_fine = compute_density(sdf_fine, s)
    
    # 5. Render Fine Color
    # color_fine, alpha_fine, weights_fine = standard_volume_render(...)
    
    # 6. Find Surface Points (Zero Crossing)
    # Combine coarse and fine SDF values
    # sdf_all = torch.cat([sdf_coarse, sdf_fine])
    # t_all = torch.cat([t_coarse, t_fine])
    # t_surface, _ = find_zero_crossing(sdf_all, t_all)
    
    # 7. Get Surface Features
    # x_surface = rays_o + rays_d * t_surface
    # n_surface = torch.autograd.grad(geometry_net(x_surface).sum(), x_surface, create_graph=True)[0]
    # n_surface = F.normalize(n_surface, dim=-1)
    # z_surface = geometry_net(x_surface) # Feature vector
    
    # 8. Render Color at Surface (for feature consistency)
    # color_surface = radiance_net(x_surface, rays_d, n_surface, z_surface)
    
    return color_fine, t_surface, z_surface, sdf_all, t_all

def compute_geometry_bias_loss(back_projected_points, sdf_values_at_points):
    """
    Penalizes absolute SDF values at back-projected points.
    """
    # loss = mean(abs(sdf_values_at_points))
    return torch.mean(torch.abs(sdf_values_at_points))

def compute_feature_consistency_loss(surface_features_ref, surface_features_src):
    """
    Computes consistency between reference and source view features.
    """
    # loss = mean((surface_features_ref - surface_features_src)^2)
    return torch.mean((surface_features_ref - surface_features_src) ** 2)

def compute_eikonal_loss(geometry_net, points):
    """
    Enforces ||grad f(x)|| = 1
    """
    points.requires_grad_(True)
    sdf = geometry_net(points)
    grad = torch.autograd.grad(sdf.sum(), points, create_graph=True)[0]
    grad_norm = torch.norm(grad, dim=-1)
    return torch.mean((grad_norm - 1.0) ** 2)

def train_step(rays_o, rays_d, images, geometry_net, radiance_net, optimizer, s, stage):
    optimizer.zero_grad()
    
    # Render
    color, t_surface, z_surface, sdf_all, t_all = volume_render(rays_o, rays_d, geometry_net, radiance_net, s)
    
    # Color Loss (e.g., L1)
    color_loss = torch.mean(torch.abs(color - images))
    
    # Eikonal Loss
    eikonal_loss = compute_eikonal_loss(geometry_net, rays_o + rays_d * t_all)
    
    # Geometry Bias Loss
    # Back-project distance maps (simplified)
    # Assume back_projected_points and sdf_at_back_projected are computed
    # geometry_bias_loss = compute_geometry_bias_loss(back_projected_points, sdf_at_back_projected)
    geometry_bias_loss = 0.0 # Placeholder for specific back-projection logic
    
    # Feature Consistency Loss
    # Assume surface_features_ref and surface_features_src are computed
    # feature_loss = compute_feature_consistency_loss(surface_features_ref, surface_features_src)
    feature_loss = 0.0 # Placeholder for specific feature matching logic
    
    # Stage-specific weights
    if stage == 1:
        beta = 0.01
        gamma = 0.0
    elif stage == 2:
        beta = 0.1
        gamma = 0.5
    else:
        beta = 0.01
        gamma = 0.05
        
    alpha = 0.1
    
    total_loss = color_loss + alpha * eikonal_loss + beta * geometry_bias_loss + gamma * feature_loss
    
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()
```

## Operating characteristics
*   **Time Complexity:** Training requires 300,000 iterations. On a single NVIDIA Titan RTX, this takes approximately 19 hours [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Inference Time:** Rendering a single image of resolution $1200 \times 1600$ using standard volume rendering takes approximately 7 minutes on the described hardware [S1, unofficial].
*   **Mesh Extraction Time:** Marching Cubes on a $512^3$ voxel grid takes about 57 seconds [S1, unofficial].
*   **Strengths:** Recovers high-fidelity surface details and achieves high-accuracy surface reconstruction, especially on complex scenes [S1, unofficial]. Outperforms NeRF, VolSDF, and NeuS in Chamfer distance on the DTU dataset [S1, unofficial].
*   **Limitations:** Slow inference time (7 minutes per image) [S1, unofficial]. Relies on multi-view input and known camera poses [S1, unofficial].
*   **When to Use:** When high-accuracy surface reconstruction with fine details is required from multi-view images, and computational time is not a primary constraint [S1, unofficial].
*   **When Not to Use:** When real-time rendering is needed, or when only a single view or sparse views are available [S1, unofficial].

## Background
D-NeuS was introduced by Chen et al. in 2023 [S1, unofficial]. It builds on NeuS, which itself is based on Neural Implicit Surfaces (IDR) and VolSDF [S1, unofficial]. The method was evaluated on the DTU and BlendedMVS datasets [S1, unofficial]. The code is available at https://github.com/fraunhoferhhi/D-NeuS [S1, unofficial].

## Sources
[S1] WACV 2023 Open Access Repository, https://openaccess.thecvf.com/content/WACV2023/html/Chen_Recovering_Fine_Details_for_Neural_Implicit_Surface_Reconstruction_WACV_2023_paper.html, fetched 2026-10-29 (unofficial)
