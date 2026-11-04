# Direct Voxel Grid Optimization (DVGO)
Also called: SimpleDVGO
## Overview
Direct Voxel Grid Optimization (DVGO) is a differentiable volume rendering method that combines ray sampling, coordinate rescaling, voxel grid interpolation for density and color, and volume rendering into a single forward pass [S1, unofficial]. It sets itself apart from standard NeRF by replacing the multi-layer perceptron (MLP) with direct lookup and interpolation in learned voxel grids, significantly reducing memory usage and training time while maintaining competitive rendering quality [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   Input: `rays` tensor of shape $(N, 6)$, where the first 3 components are ray origins and the last 3 are ray directions.
*   Output: Rendered color image tensor of shape $(N, 3)$.

**Data Structures:**
*   `t_values`: 1D tensor of shape $(num\_samples)$, storing the midpoints between sampled depths.
*   `min`, `max`: Buffers storing the minimum and maximum corners of the scene bounding box in scene coordinates.
*   `shape`: Buffer storing the dimensions of the voxel grid.
*   `rgb_voxel_grid`: A voxel grid data structure with shape `config.shape` and dimension 3, storing RGB color values.
*   `density_voxel_grid`: A voxel grid data structure with shape `config.shape` and dimension 1, storing density values.

**Parameters:**
*   `near`: Float, near plane distance for ray sampling. Default: 1.5 [S1, unofficial].
*   `far`: Float, far plane distance for ray sampling. Default: 3.5 [S1, unofficial].
*   `num_samples`: Integer, number of sample points along each ray. Default: 64 [S1, unofficial].
*   `min_corner`: Tuple of floats, minimum corner of the scene bounding box. Default: $(-1.0, -1.0, 0.0)$ [S1, unofficial].
*   `max_corner`: Tuple of floats, maximum corner of the scene bounding box. Default: $(1.0, 1.0, 1.0)$ [S1, unofficial].
*   `shape`: Tuple of ints, dimensions of the voxel grid. Default: $(128, 128, 128)$ [S1, unofficial].
*   `background`: Color value used for background in volume rendering. Default: WHITE [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Compute `depths` as linearly spaced values from `near` to `far` with `num_samples + 1` points.
    *   Compute `t_values` as the midpoints between adjacent `depths`: $t_i = \frac{depths_{i+1} + depths_i}{2}$.
    *   Initialize `rgb_voxel_grid` with shape `config.shape`, dimension 3, filled with BLUE.
    *   Initialize `density_voxel_grid` with shape `config.shape`, dimension 1, filled with 0.001.
    *   Register `background` color.
2.  **Forward Pass:**
    *   Sample points along rays using `sample_rays` with `t_values` and input rays.
    *   Rescale sampled points from scene coordinates to grid coordinates: $rescaled = shape \times \frac{samples - min}{max - min}$.
    *   Query density grid: $density = \text{ReLU}(\text{squeeze}(\text{density\_voxel\_grid}(rescaled)))$.
    *   Query RGB grid: $rgb = \text{clamp}(\text{rgb\_voxel\_grid}(rescaled), 0.0, 1.0)$.
    *   Render using `render_along_ray` with density, rgb, and background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Assuming VoxelGrid, sample_along_ray, render_along_ray, BLUE, WHITE are defined elsewhere
# VoxelGrid is a differentiable interpolation structure
# sample_along_ray(ts, origins, directions) returns points in scene coordinates

class SimpleDVGO(nn.Module):
    def __init__(self, config=None):
        super().__init__()
        if config is None:
            config = Config()
        
        # Initialize sampler parameters
        self.depths = torch.linspace(config.near, config.far, config.num_samples + 1, dtype=torch.float32)
        self.register_buffer('t_values', 0.5 * (self.depths[1:] + self.depths[:-1]))
        
        # Set up conversion from scene coordinates to grid coordinates
        self.register_buffer('min', torch.tensor(config.min_corner, dtype=torch.float32))
        self.register_buffer('max', torch.tensor(config.max_corner, dtype=torch.float32))
        self.register_buffer('shape', torch.tensor(config.shape, dtype=torch.float32))
        
        # Initialize color grid to blue
        self.rgb_voxel_grid = VoxelGrid(config.shape, d=3, fill=BLUE)
        
        # Initialize density grid with very low density
        self.density_voxel_grid = VoxelGrid(config.shape, d=1, fill=torch.full((), 0.001, dtype=torch.float32))
        
        # Record background color for rendering
        self.register_buffer('background', config.background)

    def forward(self, rays, training=True):
        # Sample points along rays
        samples = sample_rays(self.t_values, rays, training=training)
        
        # Rescale to fit within the grid
        # samples shape: (N, num_samples, 3)
        # min/max/shape shape: (3,)
        rescaled = self.shape * (samples - self.min) / (self.max - self.min)
        
        # Query Density Voxel Grid
        # density_voxel_grid(rescaled) returns shape (N, num_samples, 1)
        density = torch.squeeze(self.density_voxel_grid(rescaled))
        density = F.relu(density)
        
        # Query RGB Voxel Grid
        # rgb_voxel_grid(rescaled) returns shape (N, num_samples, 3)
        rgb = torch.clamp(self.rgb_voxel_grid(rescaled), 0.0, 1.0)
        
        # Render
        return render_along_ray(density, rgb, self.background)

def sample_rays(t_values, rays, training=True):
    """Sample points along the rays."""
    # Extract ray origins and directions from rays
    origins = rays[..., :3].to(dtype=torch.float32)
    directions = rays[..., 3:].to(dtype=torch.float32)
    
    # Add a small random scalar to t_values during training
    if training:
        with torch.no_grad():
            n = t_values.size(0)
            random_scalar = (torch.rand(n, device=rays.device) - 0.5) / n
            actual_ts = t_values.clone() + random_scalar
    else:
        actual_ts = t_values.clone()
        
    # Sample along the ray
    return sample_along_ray(actual_ts, origins, directions)
```

**Edge Cases and Pitfalls:**
*   Density values must be non-negative; achieved by applying ReLU nonlinearity after querying the density grid [S1, unofficial].
*   RGB values must be in $[0, 1]$; achieved by clamping after querying the RGB grid [S1, unofficial].
*   Training a NeRF can take a long time, so float32 is used everywhere to reduce GPU resources compared to float64 [S1, unofficial].

## Subtopics

### [DVGO with Reparameterized Rendering](3525-dvgo-with-reparameterized-rendering.md)
This method adapts the Direct Voxel Grid Optimization (DVGO) architecture by replacing its standard volume rendering integral with a reparameterized estimate based on differentiable ray point sampling. It solves the problem of high computational cost in standard ray marching by allowing the number of radiance samples to be explicitly controlled and minimized, particularly during the coarse training stage. It sets itself apart from standard DVGO by trading off rendering quality (PSNR) for significant training speed improvements in the coarse stage, while maintaining competitive performance in the fine stage only when sample counts are kept low.

## Operating characteristics
DVGO uses float32 everywhere to reduce GPU resources compared to float64, acknowledging that NeRF training takes a long time [S1, unofficial]. The method replaces the MLP with voxel grid lookups, which generally reduces memory usage and can speed up training compared to standard NeRF, although specific performance metrics are not provided in the sources. The default configuration uses a $128^3$ voxel grid and 64 samples per ray.
## Background
DVGO (Direct Voxel Grid Optimization) was introduced by Sun et al. [2022] [S1, unofficial]. It is a differentiable volume rendering method that combines ray sampling, coordinate rescaling, voxel grid interpolation for density and color, and volume rendering into a single forward pass [S1, unofficial].
## Sources
[S1] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
