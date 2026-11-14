# Volume-Augmented NeRF

Also called: Fluid volume augmented NeRF, Volume-Augmented Radiance Field

## Overview
Volume-Augmented NeRF is a neural radiance field variant designed for dynamic scenes, specifically fluid dynamics, where the density and color of sample points change over time due to fluid flow [S1, unofficial]. Unlike the standard static NeRF architecture [S6, unofficial], this method binds an explicit fluid volume value $v_i$ to each sample point, allowing the renderer to learn density and color based on the spatial distribution provided by a differentiable fluid simulator [S1, unofficial]. It serves as the rendering component in a joint differentiable framework that bridges a neural Euler simulator and the radiance field [S1, unofficial].

## How it works
**Inputs and Outputs:**
The network $F_\Psi$ takes three inputs for each sample point $x_i$ along a ray:
1.  Positional encoded position $\gamma(\mathbf{x}_i)$.
2.  Fluid volume value $v_i$ at position $\mathbf{x}_i$.
3.  Positional encoded viewing direction $\gamma(\mathbf{d})$.

The network outputs:
1.  Volume density $\sigma_i$.
2.  A 256-dimensional latent feature vector $\mathbf{h}_i$.
3.  RGB color $\mathbf{c}_i$ (derived from $\mathbf{h}_i$ and $\gamma(\mathbf{d})$).

**Data Structures and Layout:**
The architecture consists of two Multi-Layer Perceptrons (MLPs) sharing a latent feature vector, similar to the coarse/fine structure but adapted for volume augmentation.
*   **Density MLP:** Takes $\gamma(\mathbf{x}_i)$ and $v_i$.
*   **Color MLP:** Takes $\mathbf{h}_i$ and $\gamma(\mathbf{d})$.

**Step-by-Step Method:**

1.  **Fluid Volume Interpolation:**
    Assign the fluid volume field $V$ (inferred by a neural Euler simulator) into the scene space. For each sampled point $x_i$ on a ray, perform trilinear interpolation on $V$ to capture the corresponding fluid volume value $v_i$ [S1, unofficial].

2.  **Density and Feature Regression (Density MLP):**
    Concatenate the positional encoding $\gamma(\mathbf{x}_i)$ and the fluid volume $v_i$. Pass this concatenated vector through a network of 8 fully-connected ReLU layers, each with 256 channels [S1, unofficial].
    *   Include a skip connection that concatenates the initial input $(\gamma(\mathbf{x}_i), v_i)$ to the activation of the 5th layer [S1, unofficial].
    *   Pass the output of the 8th layer through an additional fully-connected layer.
    *   This final layer outputs the density $\sigma_i$ and a 256-dimensional feature vector $\mathbf{h}_i$.
    *   Apply a ReLU activation to $\sigma_i$ to ensure non-negativity [S1, unofficial].

3.  **Color Regression (Color MLP):**
    Concatenate the feature vector $\mathbf{h}_i$ with the positional encoding of the viewing direction $\gamma(\mathbf{d})$ [S1, unofficial].
    *   Process this concatenated vector through a fully-connected ReLU layer with 128 channels [S1, unofficial].
    *   Pass the output through a final fully-connected layer with a sigmoid activation to output the RGB color $\mathbf{c}_i$ [S1, unofficial].

4.  **Volumetric Rendering:**
    Compute the rendered color $\hat{C}(r)$ for a ray $r$ by integrating the contributions of all $N$ sample points along the ray using the standard volume rendering equation [S1, unofficial]:
    $$
    \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i$ is the accumulated transmittance from the near bound to point $x_i$, and $\delta_i$ is the distance between adjacent samples:
    $$
    T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)
    $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, num_freq_bands=10):
    """
    Standard NeRF positional encoding.
    x: tensor of shape (..., D)
    Returns: tensor of shape (..., D + 2*D*num_freq_bands)
    """
    scales = [2**i for i in range(num_freq_bands)]
    scaled_x = list()
    for scale in scales:
        scaled_x.append(torch.sin(x * scale * math.pi))
        scaled_x.append(torch.cos(x * scale * math.pi))
    return torch.cat([x] + scaled_x, dim=-1)

class VolumeAugmentedNeRF(nn.Module):
    def __init__(self, pos_enc_dim=15, view_enc_dim=15):
        super().__init__()
        # Density MLP
        # Input: pos_enc_dim + 1 (for volume v)
        # 8 layers of 256, skip connection at layer 5
        self.density_layers = nn.ModuleList()
        in_dim = pos_enc_dim + 1
        for i in range(8):
            self.density_layers.append(nn.Linear(in_dim, 256))
            in_dim = 256
        
        # Output layer for density and feature vector
        self.density_out = nn.Linear(256, 256 + 1) # 256 features + 1 density
        
        # Color MLP
        # Input: 256 (features) + view_enc_dim
        self.color_layers = nn.ModuleList()
        in_dim = 256 + view_enc_dim
        self.color_layers.append(nn.Linear(in_dim, 128))
        self.color_out = nn.Linear(128, 3) # RGB

    def forward_density(self, x_enc, v):
        """
        x_enc: positional encoding of position, shape (..., pos_enc_dim)
        v: fluid volume, shape (..., 1)
        """
        input_vec = torch.cat([x_enc, v], dim=-1)
        x = input_vec
        
        for i, layer in enumerate(self.density_layers):
            x = layer(x)
            if i < 7: # First 8 layers are ReLU
                x = F.relu(x)
            
            # Skip connection at the 5th layer (index 4)
            if i == 4:
                x = torch.cat([x, input_vec], dim=-1)
        
        # Output layer
        out = self.density_out(x)
        features = out[..., :256]
        density = out[..., 256:]
        density = F.relu(density) # Ensure non-negative
        return density, features

    def forward_color(self, features, d_enc):
        """
        features: 256-dim feature vector
        d_enc: positional encoding of view direction
        """
        input_vec = torch.cat([features, d_enc], dim=-1)
        x = self.color_layers[0](input_vec)
        x = F.relu(x)
        rgb = torch.sigmoid(self.color_out(x))
        return rgb

    def forward(self, x, d, v):
        """
        x: 3D positions, shape (..., 3)
        d: view directions, shape (..., 3)
        v: fluid volume values, shape (..., 1)
        """
        x_enc = positional_encoding(x)
        d_enc = positional_encoding(d)
        
        density, features = self.forward_density(x_enc, v)
        rgb = self.forward_color(features, d_enc)
        
        return density, rgb

def render_ray(ray_origin, ray_direction, t_near, t_far, num_samples, nerf_model, fluid_volume_field):
    """
    Renders a single ray using the Volume-Augmented NeRF.
    fluid_volume_field: A function or grid that allows trilinear interpolation of volume V.
    """
    t_vals = torch.linspace(t_near, t_far, num_samples)
    # Add small epsilon to avoid sampling exactly at boundaries if needed
    points = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(-1)
    
    # Interpolate fluid volume for each point
    # Assuming fluid_volume_field(points) returns shape (num_samples, 1)
    volumes = fluid_volume_field(points)
    
    # Get positional encodings
    x_enc = positional_encoding(points)
    d_enc = positional_encoding(ray_direction)
    
    # Forward pass through NeRF
    densities, rgbs = nerf_model(points, ray_direction, volumes)
    
    # Volumetric rendering
    # delta_t is the distance between samples
    delta_t = t_vals[1] - t_vals[0]
    
    # Alpha: 1 - exp(-sigma * delta)
    alphas = 1.0 - torch.exp(-densities * delta_t)
    
    # Transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Accumulate opacity
    accumulated_opacity = torch.cumsum(alphas, dim=0)
    # T_i = 1 - accumulated_opacity_{i-1} ? 
    # Standard formula: T_i = exp(-sum_{j=1}^{i-1} sigma_j delta_j)
    # Let's compute T directly
    # sigma * delta is the opacity contribution
    sigma_delta = densities * delta_t
    # Cumulative sum of sigma_delta
    cum_sigma_delta = torch.cumsum(sigma_delta, dim=0)
    # T_i = exp(-cum_sigma_delta_{i-1})
    # Shift cum_sigma_delta by 1 to get T_i for each i
    # T_1 = 1 (exp(0))
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j delta_j)
    
    # Create T tensor
    # T[0] = 1.0
    # T[i] = exp(-cum_sigma_delta[i-1])
    T = torch.zeros_like(cum_sigma_delta)
    T[0] = 1.0
    if num_samples > 1:
        T[1:] = torch.exp(-cum_sigma_delta[:-1])
        
    # Color = sum(T_i * alpha_i * c_i)
    # Note: alpha_i = 1 - exp(-sigma_i * delta_i)
    # The term in the sum is T_i * (1 - exp(-sigma_i * delta_i)) * c_i
    # Which is T_i * alphas * rgbs
    
    weights = T * alphas
    color = torch.sum(weights.unsqueeze(-1) * rgbs, dim=0)
    
    return color
```

**Parameters:**
*   Hidden layer size for the first 8 density layers: 256 channels [S1, unofficial].
*   Hidden layer size for the view-dependent color layer: 128 channels [S1, unofficial].
*   Number of positional encoding frequencies: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Fluid Volume Availability:** The method requires a pre-computed or simultaneously differentiable fluid volume field $V$. If $V$ is not available or inaccurate, the density and color predictions will be incorrect [S1, unofficial].
*   **Trilinear Interpolation:** The fluid volume $v_i$ is obtained via trilinear interpolation. Discontinuities or aliasing in the fluid field may affect the smoothness of the rendered output [S1, unofficial].
*   **Dynamic Consistency:** The method assumes the fluid volume field $V$ accurately reflects the physical state of the fluid at the sampled time step. Temporal inconsistencies in the simulator can lead to visual artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N$ per ray and the depth of the MLPs (8 layers for density, 1 for color). The interpolation step adds overhead proportional to $N$.
*   **Memory Complexity:** The network weights are fixed. The memory usage is dominated by the storage of the fluid volume field $V$ and the intermediate activations during backpropagation.
*   **Strengths:** Enables the rendering of dynamic fluid scenes by explicitly conditioning the radiance field on fluid volume. Allows for joint optimization of the fluid simulator and the radiance field [S1, unofficial].
*   **Limitations:** Restricted to scenes where a fluid volume representation is available or can be simulated. Not suitable for general dynamic scenes without a corresponding volume field.
*   **When to use:** When rendering dynamic fluid simulations (e.g., water, smoke) where the fluid dynamics are known or can be modeled by a differentiable simulator [S1, unofficial].
*   **When not to use:** For static scenes (standard NeRF is sufficient) or dynamic scenes without a clear volumetric representation [S1, unofficial].

## Background
Volume-Augmented NeRF was introduced in the paper "Inferring Fluid Dynamics via Inverse Rendering" [S1, unofficial]. It extends the original NeRF architecture [S6, unofficial] to handle dynamic scenes by incorporating fluid volume information. The method is part of a framework that combines a neural Euler fluid simulator with a differentiable renderer, allowing the simulation to be driven by scene radiance field learned from images [S1, unofficial]. The authors demonstrated the method on datasets such as "Fluid Fall" and "Dam Break" [S1, unofficial].

## Sources
[S1] Inferring Fluid Dynamics via Inverse Rendering, https://doi.org/10.48550/arxiv.2304.04446, fetched 2026-10-26 (unofficial)
