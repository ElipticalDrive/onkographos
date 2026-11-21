# Deep Reflectance Volumes

Also called: DRV, DRV rendering, Deep Reflectance Volume Rendering, discretized differentiable volume rendering

## Overview
Deep Reflectance Volumes (DRV) is a differentiable volume rendering method that reconstructs scene appearance from unstructured images captured under collocated point lighting by optimizing a volumetric representation of opacity, surface normal, and reflectance parameters [S1]. It distinguishes itself from standard volume rendering by learning a global opacity field that accumulates along both view and light directions, enabling the synthesis of accurate hard shadows and complex specular reflections under novel lighting conditions, which standard alpha-compositing methods typically fail to model [S1].

## How it works
**Inputs:** A set of $K$ captured images $\{I_k\}_{k=1}^K$ with known camera poses and collocated light positions.
**Outputs:** A learned volumetric scene representation consisting of voxel grids for opacity, normal, and reflectance, optimized to minimize the difference between rendered and captured images.

### Data Structures
The scene is represented in a "warped" latent space before being mapped to world space for rendering. The core data structure is an 8-channel tensor of resolution $128 \times 128 \times 128$ [S1]:
*   **Channel 0:** Opacity $\alpha_w$ (scalar).
*   **Channels 1–3:** Surface normal $\mathbf{n}_w$ (3D vector, typically normalized).
*   **Channels 4–6:** Diffuse albedo (3D vector, RGB).
*   **Channel 7:** Specular roughness (scalar).

Additionally, the warping function requires:
*   **Global Affine Matrix:** $W_g$ ($4 \times 4$ matrix).
*   **Basis Affine Matrices:** $\{W_j\}_{j=1}^{16}$ (16 matrices, each $4 \times 4$).
*   **Weight Volume:** $B$ ($32 \times 32 \times 32$ tensor, 16 channels). $B_j(\mathbf{x})$ denotes the $j$-th channel value at position $\mathbf{x}$.

### Warping Function
To increase effective resolution, world-space positions $\mathbf{x}$ are warped to a transformed space $\mathbf{x}_w$ using a learned affine-based warping function [S1]:

$$
W(\mathbf{x}) = \left[ \sum_{j=1}^{16} B_j(\mathbf{x}) W_j \right]^{-1} W_g \mathbf{x}
$$

The volume is decoded into this warped space. During ray marching, points are sampled in world space, but the network queries the volume at the warped coordinates $W(\mathbf{x})$ [S1].

### Rendering Pipeline
The rendering module is a differentiable volume ray marcher based on classical alpha compositing [S1]. It computes reflected radiance by accumulating contributions from sampled points, modeling single-bounce light transport.

**1. Ray Sampling:**
For a camera at position $\mathbf{c}$ and direction $-\boldsymbol{\omega}_o$, points are sampled along the ray with a fixed step size $\Delta x$:
$$
\mathbf{x}_s = \mathbf{x}_{s-1} - \boldsymbol{\omega}_o \Delta x = \mathbf{c} - s \boldsymbol{\omega}_o \Delta x
$$
Sampling continues until the ray exits the volume bounds (defined as a cube $[-1, 1]^3$) [S1].

**2. Opacity Accumulation:**
Unlike standard volume rendering which only accumulates opacity along the view direction, DRV accumulates opacity from both the camera and the light source to model shadows [S1].
*   **Camera Opacity:** $\alpha_{c \to s}$ accumulates occlusion along the view ray.
*   **Light Opacity:** $\alpha_{l \to t}$ accumulates occlusion along the ray from the sample point to the light source $\mathbf{l}$.

During training with collocated camera and light, $\alpha_{l \to t} = \alpha_{c \to s}$, allowing a single-pass accumulation. For novel lighting, two-pass accumulation is required [S1].

**3. Radiance Compositing:**
At each sample point $\mathbf{x}_s$, the local normal $\mathbf{n}(\mathbf{x}_s)$ and reflectance parameters $R(\mathbf{x}_s)$ (albedo and roughness) are queried from the volume. The reflected radiance $L(\mathbf{x}_s)$ is computed using a simplified Disney BRDF model [S1]:
$$
L(\mathbf{x}_s) = f_r(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i, \mathbf{n}(\mathbf{x}_s), R(\mathbf{x}_s)) L_l(\mathbf{x}_s, \boldsymbol{\omega}_i)
$$
where $\boldsymbol{\omega}_i$ is the direction from $\mathbf{x}_s$ to the light $\mathbf{l}$, and $L_l$ is the light radiance.

The final accumulated radiance $L_s$ and camera opacity $\alpha_{c \to s}$ are updated recursively [S1]:
$$
L_s = L_{s-1} + [1 - \alpha_{c \to (s-1)}][1 - \alpha_{l \to (t-1)}]\alpha(\mathbf{x}_s)L(\mathbf{x}_s)
$$
$$
\alpha_{c \to s} = \alpha_{c \to (s-1)} + [1 - \alpha_{c \to (s-1)}] \alpha(\mathbf{x}_s)
$$
$$
\alpha_{l \to t} = \alpha_{l \to (t-1)} + [1 - \alpha_{l \to (t-1)}] \alpha(\mathbf{x}_t)
$$
The rendered color for the pixel is $L_{final}$ when the ray exits the volume [S1].

### Optimization
The network parameters (encoding vector, transposed convolutional layers for upsampling) and the volume values are optimized per scene using gradient descent [S1]. The loss function combines rendering error and regularization terms [S1]:

$$
\mathcal{L} = \sum_k \| L_k - \tilde{L}_k \|^2 + \beta_1 \sum_k \left[ \log(\alpha_{c_k \to s'}) + \log(1 - \alpha_{c_k \to s'}) \right] + \beta_2 \sum \| \nabla_{\mathbf{x}} \log \alpha(\mathbf{x}) \|
$$

*   **Rendering Loss:** Mean squared error between rendered image $L_k$ and captured image $\tilde{L}_k$.
*   **Opacity Regularization:** Encourages accumulated opacity to be binary (0 or 1) to reduce artifacts [S1].
*   **Sparsity Regularization:** Encourages spatial sparsity of the volume opacity [S1].

### Parameters and Settings
*   **Volume Bounds:** Cube $[-1, 1]^3$ [S1].
*   **Ray Marching Step Size ($\Delta x$):** $1/64$ during training [S1]. Note that the voxel opacity $\alpha$ is dependent on this step size, representing one minus the transmission over segment $\Delta x$ [S1].
*   **Training Batch:** 128×128 pixels sampled from 8 captured images [S1].
*   **Regularization Weights:** Initial $\beta_1 = 0.01, \beta_2 = 0.01$. After 300,000 iterations, increased to $\beta_1 = 1.0, \beta_2 = 0.1$ to sharpen boundaries [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class DeepReflectanceVolume:
    def __init__(self, volume_size=128, warping_grid_size=32, num_basis=16):
        # Initialize learnable parameters
        # 8-channel volume: 1 alpha, 3 normal, 3 albedo, 1 roughness
        self.volume = nn.Parameter(torch.randn(8, volume_size, volume_size, volume_size))
        
        # Warping parameters
        self.W_g = nn.Parameter(torch.eye(4)) # Global affine
        self.W_bases = nn.Parameter(torch.randn(num_basis, 4, 4)) # 16 basis matrices
        self.B_weights = nn.Parameter(torch.randn(16, warping_grid_size, warping_grid_size, warping_grid_size)) # 16-channel weight volume
        
        # Hyperparameters
        self.step_size = 1.0 / 64.0
        self.volume_bounds_min = -1.0
        self.volume_bounds_max = 1.0
        
        # Reflectance model parameters (simplified Disney BRDF)
        # Note: Actual BRDF implementation is complex; here we assume a callable
        self.brdf = SimplifiedDisneyBRDF() 

    def warp(self, x_world):
        """
        Warp world-space position x to transformed space.
        x_world: (N, 3) tensor
        Returns: (N, 3) tensor in warped space
        """
        # Convert to homogeneous coordinates (N, 4)
        x_homo = torch.cat([x_world, torch.ones(x_world.shape[0], 1, device=x_world.device)], dim=1)
        
        # Compute spatially varying basis matrix sum
        # B_weights is (16, 32, 32, 32). We need to interpolate B at x_world.
        # Assuming x_world is normalized to [0,1] for grid lookup or scaled appropriately.
        # For implementation, we assume a helper function 'trilinear_sample' exists.
        B_vals = trilinear_sample(self.B_weights, x_world) # Shape: (N, 16)
        
        # Sum of B_j * W_j
        # W_bases is (16, 4, 4)
        # Result is (N, 4, 4)
        M_spatial = torch.einsum('nj,jkl->nkl', B_vals, self.W_bases)
        
        # Inverse of the spatial matrix
        # Add small epsilon for stability
        M_inv = torch.inverse(M_spatial + 1e-6 * torch.eye(4).unsqueeze(0).repeat(M_spatial.shape[0], 1, 1))
        
        # Apply global warp and inverse spatial warp
        # W(x) = [Sum B_j W_j]^-1 W_g x
        x_warped_homo = torch.bmm(M_inv, torch.bmm(self.W_g.unsqueeze(0).repeat(M_spatial.shape[0], 1, 1), x_homo.unsqueeze(2)).squeeze(2))
        
        # Convert back to 3D
        x_warped = x_warped_homo[:, :3] / x_warped_homo[:, 3:4]
        return x_warped

    def query_volume(self, x_warped):
        """
        Query the 8-channel volume at warped coordinates.
        x_warped: (N, 3) tensor, coordinates in [0, 1] range relative to volume grid
        Returns: (N, 8) tensor
        """
        # Normalize coordinates to [-1, 1] for trilinear interpolation if needed, 
        # or scale to [0, 1] depending on PyTorch grid_sample convention.
        # Assuming grid_sample expects normalized coordinates [-1, 1]
        coords = 2.0 * x_warped - 1.0
        coords = coords.unsqueeze(0).unsqueeze(0) # (1, 1, N, 3)
        
        # grid_sample expects (N, C, H, W) or (N, C, D, H, W)
        # We need to permute volume to (1, 8, D, H, W)
        vol = self.volume.unsqueeze(0) # (1, 8, 128, 128, 128)
        
        # Sample
        # grid_sample output shape: (1, 8, 1, N)
        samples = torch.nn.functional.grid_sample(vol, coords, align_corners=True)
        return samples.squeeze(2).squeeze(0).T # (N, 8)

    def render_ray(self, camera_pos, ray_dir, light_pos):
        """
        Render a single ray using DRV.
        """
        # Determine ray bounds intersection with [-1, 1]^3
        # Simplified: assume ray starts inside or we clip.
        # For robustness, compute t_min and t_max.
        # Omitting detailed AABB intersection for brevity, assuming valid samples.
        
        num_samples = 100 # Example
        t_values = torch.linspace(0, 1, num_samples, device=camera_pos.device) * 10.0 # Arbitrary max depth
        
        # Sample points in world space
        x_world = camera_pos.unsqueeze(0) + ray_dir.unsqueeze(0) * t_values.unsqueeze(1)
        
        # Warp points
        x_warped = self.warp(x_world)
        
        # Query volume
        # Volume values: alpha, normal, albedo, roughness
        vol_data = self.query_volume(x_warped) # (N, 8)
        
        alpha = vol_data[:, 0].sigmoid() # Ensure alpha in [0,1]
        normal = torch.nn.functional.normalize(vol_data[:, 1:4], dim=1)
        albedo = vol_data[:, 4:7]
        roughness = vol_data[:, 7].sigmoid()
        
        # Initialize accumulators
        L_acc = torch.zeros_like(alpha[0])
        alpha_c = torch.zeros_like(alpha[0]) # Camera opacity
        alpha_l = torch.zeros_like(alpha[0]) # Light opacity
        
        for s in range(num_samples):
            # Check if ray exited volume (opacity reached 1)
            if alpha_c[s] >= 1.0:
                break
                
            # Compute light direction
            light_dir = light_pos - x_world[s]
            light_dir = torch.nn.functional.normalize(light_dir, dim=0)
            
            # Compute light opacity (occlusion from light to point)
            # In collocated training, alpha_l == alpha_c. 
            # For general case, we need to march towards light.
            # Simplified: assume alpha_l is computed via a separate pass or approximation.
            # Here we use the accumulated camera opacity as a proxy for shadow if collocated,
            # or compute it explicitly. The source says for non-collocated, two-pass is needed.
            # For this pseudocode, we assume a helper `compute_light_opacity` exists.
            alpha_l_s = compute_light_opacity(self, x_world[s], light_pos)
            
            # Compute reflected radiance
            # View direction is -ray_dir
            view_dir = -ray_dir
            radiance = self.brdf(normal[s], view_dir, light_dir, albedo[s], roughness[s])
            
            # Compositing
            # L_s = L_{s-1} + [1 - alpha_c][1 - alpha_l] * alpha * L
            contribution = (1.0 - alpha_c[s]) * (1.0 - alpha_l_s) * alpha[s] * radiance
            L_acc = L_acc + contribution
            
            # Update camera opacity
            alpha_c_next = alpha_c[s] + (1.0 - alpha_c[s]) * alpha[s]
            alpha_c = torch.cat([alpha_c, alpha_c_next.unsqueeze(0)])
            
        return L_acc[-1]

    def forward(self, camera_pos, ray_dir, light_pos):
        return self.render_ray(camera_pos, ray_dir, light_pos)

def compute_light_opacity(drv, point, light_pos):
    """
    Helper to compute light opacity. 
    In practice, this requires marching from point to light.
    For collocated training, this equals camera opacity.
    """
    # Placeholder for the second ray march towards light
    # Returns accumulated opacity along the light ray
    return 0.0 # Simplified for collocated case where alpha_l = alpha_c
```

### Edge Cases and Pitfalls
*   **Step Size Dependency:** The voxel opacity $\alpha$ is dependent on the ray marching step size $\Delta x$. Changing $\Delta x$ without retraining or adjusting the volume values will alter the appearance [S1].
*   **Lighting Configuration:** During training with collocated camera and light, light opacity is equivalent to camera opacity, allowing single-pass accumulation. Re-rendering under non-collocated lighting requires a two-pass accumulation (one for view, one for light) to compute accurate shadows [S1].
*   **Multiple Scattering:** The method simplifies the radiative transfer equation to single scattering. Fully computing multiple scattering is computationally expensive and unsuitable for this deep learning framework [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N \cdot S)$ per pixel, where $N$ is the number of samples per ray and $S$ is the number of rays (pixels). The warping function adds overhead for coordinate transformation and volume interpolation.
*   **Memory Complexity:** The volume representation requires memory proportional to the resolution of the voxel grids. For a $128^3$ volume with 8 channels, this is significant but manageable. The warping grid adds $32^3 \times 16$ channels.
*   **Strengths:** Handles complex materials (high glossiness) and accurate hard shadows under novel lighting, which mesh-based methods and standard volume rendering struggle with [S1]. Editable reflectance volumes allow material editing [S1].
*   **Limitations:** Limited to single-bounce light transport. Performance is sensitive to the step size $\Delta x$ and the learned warping function. Requires per-scene optimization, making it slower than feed-forward methods [S1].

## Background
Deep Reflectance Volumes was introduced by Kriegman et al. from UC San Diego and Adobe Research [S1]. It addresses the limitations of previous view synthesis methods that only accumulated opacity along view directions and learned single colors per voxel, failing to model complex reflectance and shadows [S1]. The method builds on differentiable volume rendering principles [G] and physically-based shading models [S1].

## Sources
[S1] Deep Reflectance Volumes: Relightable Reconstructions from Multi-view Photometric Images, https://doi.org/10.1007/978-3-030-58580-8_18, fetched 2026-10-26
