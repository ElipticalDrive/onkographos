# HoloINR
Also called: Implicit Neural Representation for Near-Field Holotomography
## Overview
HoloINR solves the coupled inverse problem of holotomography (holography and tomography) using Implicit Neural Representations (INRs) for near-field setups where Fresnel propagation models the physics between object and detector [S1, unofficial]. It differs from standard tomographic INRs by integrating wave-optics propagation (Fresnel diffraction) into the forward model, allowing reconstruction from holographic measurements rather than simple line integrals [S1, unofficial].
## How it works
### Inputs and outputs
**Inputs:**
*   Holographic measurement data: An HDF5 file containing `angles` (list of floats in $[0, \pi)$) and `data` (holographic measurements, first axis angles, last two axes image dimensions) [S1, unofficial].
*   Geometry parameters: Defined by the `ParallelGeometry` or `ConeBeam` classes, including Source-Detector distances, pixel sizes, and voxel grid dimensions [S1, unofficial].
*   Network hyperparameters: `hidden_dim`, `num_layers`, `skips`, `bound` (spatial normalization), `netchunk`, `num_grad_samples`, `downsampling` [S1, unofficial].

**Outputs:**
*   Reconstructed 3D volume represented by the trained INR weights, encoding the complex refractive index or absorption coefficient [S1, unofficial].

### Data structures
**ParallelGeometry** [S1, unofficial]:
*   `DSO`: float, Source-Origin distance.
*   `DSD`: float, Source-Detector distance.
*   `nDetector`: 2x1 array, pixel count.
*   `dDetector`: 2x1 array, pixel size.
*   `sDetector`: 2x1 array, total detector size.
*   `nVoxel`: 3x1 array, voxel count.
*   `dVoxel`: 3x1 array, voxel size.
*   `sVoxel`: 3x1 array, total image size.
*   `offOrigin`: 3x1 array, image center offset.
*   `offDetector`: 2x1 array, detector center offset.
*   `rotDetector`: 3x1 array, detector rotation.
*   `mode`: string, 'parallel' or 'cone'.
*   `filter`: string.

**ConeBeam** [S1, unofficial]:
*   `energy`: float, X-ray energy.
*   `_px_size`: float, pixel size.
*   `active_px_size`: float, effective pixel size after downsampling.
*   `z02`: float, Source-Detector distance ($z_0 + z_{12}$).
*   `z01`: float, Source-Object distance ($z_0$).

**INR Model** [S1, unofficial]:
*   Input dimension: `encoder.output_dim + input_cat_dims`.
*   Layers: `nn.ModuleList` of `nn.Linear` layers. Skip connections add input dimensions to hidden layers at indices specified by `skips`.
*   Activations: `nn.LeakyReLU` for hidden layers, `last_activation` for output.
*   Normalization: Input coordinates divided by `bound`.

### Numbered steps
1.  **Geometry Setup**: Initialize `ParallelGeometry` or `ConeBeam` with experimental parameters. For cone-beam, calculate magnification $M = (z_{12} + z_{01}) / z_{01}$ and effective Fresnel number [S1, unofficial].
2.  **Ray Sampling**:
    *   Compute normalized t-values along the ray.
    *   Interpolate between near and far bounds to get z-values.
    *   Compute 3D points: `pts = rays_o + rays_d * z_vals`.
    *   Clamp points to `[-bound, bound]` [S1, unofficial].
3.  **Network Evaluation**:
    *   Split rays into batches of size `netchunk`.
    *   Pass points to the INR model.
    *   Sum the output over the z-dimension (`dim=-2`).
    *   Concatenate results along the ray dimension [S1, unofficial].
4.  **Fresnel Propagation** (if applicable): Apply Fresnel propagator to simulate near-field diffraction effects between object and detector [S1, unofficial].
5.  **Segment Distance Weighting**:
    *   Compute segment distances: `dists = (z_vals[..., 1:] - z_vals[..., :-1]) * ||rays_d||`.
    *   Append `1e-10` to the last segment to prevent zero-length division.
    *   Compute accumulation: `acc = sum(raw * dists, dim=-2)` [S1, unofficial].
6.  **Regularization**: Compute Isotropic Total Variation (TV) regularization:
    $$
    \text{TV} = \text{mean}(\sqrt{dx^2 + dy^2 + dz^2 + 10^{-6}})
    $$
    where $dx, dy, dz$ are finite differences along the x, y, and z axes respectively [S1, unofficial].
7.  **Optimization**: Minimize loss using Adam or SGD, updating INR weights. Use `downsampling` factor (4 or 8) to reduce computation during optimization [S1, unofficial].

### Formulas
**Fresnel Number Calculation** (Cone-Beam) [S1, unofficial]:
$$
\lambda = \frac{1.2398}{\text{energy}}
$$
$$
M = \frac{z_{12} + z_{01}}{z_{01}}
$$
$$
dx_{\text{eff}} = \frac{\text{active\_px\_size}}{M}
$$
$$
z_{\text{eff}} = \frac{z_{12}}{M}
$$
$$
\text{fr}_{\text{eff}} = \frac{dx_{\text{eff}}^2}{\lambda \cdot z_{\text{eff}}}
$$

**Total Variation Regularization** [S1, unofficial]:
$$
\text{TV} = \frac{1}{N} \sum_{i=1}^{N} \sqrt{(\Delta_x \phi_i)^2 + (\Delta_y \phi_i)^2 + (\Delta_z \phi_i)^2 + 10^{-6}}
$$

**Accumulation** [S1, unofficial]:
$$
\text{acc} = \sum_{j} \text{raw}_j \cdot \text{dists}_j
$$

### Parameters
*   `netchunk`: Controls VRAM usage by limiting rays per forward pass. Default: 1024 [S1, unofficial].
*   `num_grad_samples`: Maximum rays for gradient computation. Default: 1000 [S1, unofficial].
*   `downsampling`: Factor applied to `n_samples` during optimization. Typical values: 4 or 8 [S1, unofficial].

### Edge cases and pitfalls
*   **Zero-distance numerical instability**: The last segment distance is explicitly set to `1e-10` to prevent division by zero or NaN propagation when computing `acc` [S1, unofficial].
*   **Memory overflow**: If VRAM is exhausted, decrease `netchunk` or `num_grad_samples` [S1, unofficial].
*   **NaN/Inf propagation**: The render function checks for NaN or Inf values in accumulated output and prints a warning if detected [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class HoloINRModel(nn.Module):
    def __init__(self, hidden_dim, num_layers, skips, bound, encoder, input_cat_dims, last_activation):
        super().__init__()
        self.bound = bound
        self.encoder = encoder
        self.input_cat_dims = input_cat_dims
        self.in_dim = encoder.output_dim + input_cat_dims
        
        self.layers = nn.ModuleList([nn.Linear(self.in_dim, hidden_dim)])
        for i in range(1, num_layers - 1):
            if i in skips:
                self.layers.append(nn.Linear(hidden_dim + self.in_dim, hidden_dim))
            else:
                self.layers.append(nn.Linear(hidden_dim, hidden_dim))
        self.layers.append(nn.Linear(hidden_dim, 1)) # out_dim=1 for scalar field
        
        self.activations = nn.ModuleList([nn.LeakyReLU() for _ in range(num_layers - 1)])
        self.last_activation = last_activation

    def forward(self, x, cat_args=None):
        x = x / self.bound
        x = self.encoder(x, 1)
        if cat_args is not None:
            x = torch.cat((x, cat_args), dim=-1)
        
        input_pts = x[..., :self.in_dim]
        
        for i, layer in enumerate(self.layers):
            if i in self.skips:
                x = torch.cat([x, input_pts], dim=-1)
            x = layer(x)
            if i < len(self.layers) - 1:
                x = self.activations[i](x)
            else:
                x = self.last_activation(x)
        return x

def render_rays(rays_o, rays_d, z_vals, model, netchunk=1024, bound=1.0):
    """
    Render rays through the INR volume.
    """
    # Clamp points to bound
    pts = rays_o + rays_d.unsqueeze(-1) * z_vals.unsqueeze(-1)
    pts = torch.clamp(pts, -bound, bound)
    
    # Chunked evaluation
    num_rays = rays_o.shape[0]
    all_raw = []
    
    for i in range(0, num_rays, netchunk):
        chunk_pts = pts[i:i+netchunk]
        # Flatten for network input if needed, assuming model handles batch dim
        raw = model(chunk_pts) # Shape: [netchunk, num_samples, 1]
        all_raw.append(raw)
        
    raw = torch.cat(all_raw, dim=0) # Shape: [num_rays, num_samples, 1]
    
    # Segment distance weighting
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    dists = dists * torch.norm(rays_d, dim=-1, keepdim=True)
    # Append small value to last segment
    last_dist = torch.ones_like(dists[..., :1]) * 1e-10
    dists = torch.cat([dists, last_dist], dim=-1)
    
    # Accumulate
    acc = torch.sum(raw * dists, dim=-2)
    
    # Check for NaN/Inf
    if torch.isnan(acc).any() or torch.isinf(acc).any():
        print("Warning: NaN or Inf detected in accumulation")
        
    return acc

def compute_tv_loss(volume_grid):
    """
    Compute isotropic Total Variation regularization.
    """
    dx = volume_grid[..., 1:, :, :] - volume_grid[..., :-1, :, :]
    dy = volume_grid[..., :, 1:, :] - volume_grid[..., :, :-1, :]
    dz = volume_grid[..., :, :, 1:] - volume_grid[..., :, :, :-1]
    
    tv = torch.mean(torch.sqrt(dx**2 + dy**2 + dz**2 + 1e-6))
    return tv
```

## Subtopics

### Neural Adaptive Tomography (NeAT)
Uses adaptive sampling and neural networks for sparse-view tomography. Choose when data is extremely sparse. [Neural Adaptive Tomography (NeAT)](1862-neural-adaptive-tomography-neat.md)

### Canonical Neural Radiance Field for X-ray Tomography
Standard NeRF applied to X-ray tomography with parallel beam assumptions. Choose for standard far-field tomography. [Canonical Neural Radiance Field for X-ray Tomography](11451-canonical-neural-radiance-field-for-x-ray-tomography.md)

### VolumeNeRF
Uses volumetric NeRF for tomographic reconstruction. Choose for high-quality rendering of complex scenes. [VolumeNeRF](13110-volumenerf.md)

## Operating characteristics
*   **Time Complexity**: Depends on the number of rays, samples per ray, and network depth. Chunked evaluation (`netchunk`) limits memory but increases overhead.
*   **Memory Complexity**: Proportional to `netchunk` and `num_grad_samples`. High-resolution volumes require careful tuning of these parameters to avoid VRAM overflow [S1, unofficial].
*   **Performance**: Tested on Nvidia H100/H200 GPUs. Performance is significantly slower on older GPUs [S1, unofficial].
*   **Strengths**: Handles near-field holographic data; integrates wave optics; differentiable end-to-end.
*   **Limitations**: Computationally expensive; sensitive to VRAM limits; requires careful tuning of `netchunk` and `downsampling` [S1, unofficial].
*   **When to use**: Near-field holotomography with holographic measurements.
*   **When not to use**: Far-field tomography with simple projection data (use standard NeRF or algebraic methods).

## Background
HoloINR was introduced by Johannes Gruen et al. in 2026 at ICASSP 2026 [S1, unofficial]. It addresses the limitations of traditional tomography in near-field setups where Fresnel diffraction effects are significant. The method leverages Implicit Neural Representations to provide a continuous, differentiable volume representation suitable for gradient-based optimization [S1, unofficial].

## Sources
[S1] DESY-FS-PETRA/HoloINR, https://github.com/DESY-FS-PETRA/HoloINR, fetched 2026-10-27 (unofficial)
