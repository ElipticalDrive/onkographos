# DVGO Density and Color Rendering

Also called: DVGO volume rendering

## Overview
DVGO (Direct Volume Gaussian Optimization) is a method for rendering 3D scenes from images by combining a sparse voxel grid with neural networks to represent density and view-dependent color. It solves the problem of efficiently synthesizing novel views of complex scenes by decoupling the representation of geometry (density) from appearance (color). Unlike standard NeRF which uses a single MLP for both, DVGO uses a voxel grid for density and a shallow MLP for color, allowing for faster training and inference while maintaining high fidelity. It is distinct from [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md) by its specific use of shifted softplus activation for density and post-activation interpolation to achieve sharp surfaces with fewer cells.

## How it works
**Inputs:**
- A set of $N$ input images with known camera poses (position and orientation).
- A voxel grid resolution $N_x \times N_y \times N_z$.
- A feature grid $V^{(f)}$ of size $R^{C \times N_x \times N_y \times N_z}$ storing color features.
- A density grid $V^{(d)}$ of size $R^{N_x \times N_y \times N_z}$ storing raw density values.
- A shallow Multi-Layer Perceptron (MLP) parameterized by $\Theta$.
- Hyperparameters: shift $b$ for softplus, number of samples $n$ per ray.

**Outputs:**
- A rendered color image for a given camera view.

**Data Structures:**
1.  **Density Grid ($V^{(d)}$):** A 3D tensor of shape $(N_x, N_y, N_z)$ storing raw scalar values $\ddot{\sigma}$. These values are not densities directly but inputs to the activation function.
2.  **Feature Grid ($V^{(f)}$):** A 3D tensor of shape $(C, N_x, N_y, N_z)$ storing $C$-dimensional feature vectors at each voxel.
3.  **MLP ($MLP_\Theta$):** A small neural network that takes concatenated inputs of interpolated features, positional encodings of position, and positional encodings of direction.

**Numbered Steps:**

1.  **Ray Casting:** For each pixel, cast a ray $r$ from the camera origin through the pixel center. Determine the intersection interval $[t_n, t_f]$ with the volume bounds.
2.  **Sampling:** Sample $n$ points $x_i$ uniformly along the ray within $[t_n, t_f]$. Let $\delta_i$ be the distance between adjacent sampled points (step size).
3.  **Density Retrieval and Activation:**
    - Retrieve the raw density value $\ddot{\sigma}_i$ from the density grid $V^{(d)}$ at position $x_i$ using trilinear interpolation:
      $$ \ddot{\sigma}_i = \text{interp}(x_i, V^{(d)}) $$
    - Apply the shifted softplus activation to obtain the volume density $\sigma_i$:
      $$ \sigma_i = \text{softplus}(\ddot{\sigma}_i) = \log(1 + \exp(\ddot{\sigma}_i + b)) $$
      where $b$ is a learnable or fixed hyperparameter shift. This ensures $\sigma_i \ge 0$.
4.  **Color Feature Retrieval:**
    - Retrieve the color feature vector $f_i$ from the feature grid $V^{(f)}$ at position $x_i$ using trilinear interpolation:
      $$ f_i = \text{interp}(x_i, V^{(f)}) $$
5.  **Color Emission Computation:**
    - Apply positional encoding $\gamma(\cdot)$ to the position $x_i$ and the ray direction $d$.
    - Concatenate the interpolated feature $f_i$, encoded position $\gamma(x_i)$, and encoded direction $\gamma(d)$.
    - Pass the concatenated vector through the shallow MLP to get the view-dependent color emission $c_i^{(rgb)}$:
      $$ c_i^{(rgb)} = MLP_\Theta([f_i; \gamma(x_i); \gamma(d)]) $$
6.  **Termination Probability:**
    - Calculate the probability $\alpha_i$ that the ray terminates at sample $i$:
      $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
7.  **Accumulated Transmittance:**
    - Calculate the accumulated transmittance $T_i$ from the near plane to sample $i$:
      $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
      with $T_1 = 1$.
8.  **Volume Rendering Quadrature:**
    - Accumulate the color contributions from all samples and add the background color $c_{bg}$ attenuated by the final transmittance:
      $$ \hat{C}(r) = \sum_{i=1}^{n} T_i \alpha_i c_i^{(rgb)} + T_{n+1} c_{bg} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def softplus(x, b=0.0):
    """Shifted softplus activation."""
    return np.log(1 + np.exp(x + b))

def trilinear_interpolate(grid, x, y, z):
    """
    Trilinear interpolation for a 3D grid.
    grid: numpy array of shape (Nx, Ny, Nz) or (C, Nx, Ny, Nz)
    x, y, z: float coordinates in [0, 1] relative to grid bounds
    Returns: scalar or vector
    """
    # Implementation details for trilinear interpolation are assumed standard
    # This is a placeholder for the actual interpolation logic
    pass

def positional_encoding(x, min_deg, max_deg):
    """
    Standard positional encoding for NeRF-like methods.
    """
    # Implementation details for positional encoding are assumed standard
    pass

def render_ray(ray_origin, ray_direction, near, far, 
               density_grid, feature_grid, mlp, b, n_samples, c_bg):
    """
    Renders a single ray using DVGO.
    """
    # 1. Sample points along the ray
    t_vals = np.linspace(near, far, n_samples)
    points = ray_origin + t_vals[:, np.newaxis] * ray_direction
    
    # Calculate step sizes (delta)
    # delta_i is distance to adjacent point. For uniform sampling:
    delta = (far - near) / (n_samples - 1)
    
    accumulated_color = np.zeros(3)
    accumulated_transmittance = 1.0
    
    for i in range(n_samples):
        x, y, z = points[i]
        
        # 2. Retrieve density and apply activation
        raw_density = trilinear_interpolate(density_grid, x, y, z)
        sigma = softplus(raw_density, b)
        
        # 3. Calculate termination probability
        alpha = 1.0 - np.exp(-sigma * delta)
        
        # 4. Retrieve color features
        color_features = trilinear_interpolate(feature_grid, x, y, z)
        
        # 5. Compute color emission
        pos_enc = positional_encoding(points[i], min_deg=0, max_deg=10)
        dir_enc = positional_encoding(ray_direction, min_deg=0, max_deg=4)
        mlp_input = np.concatenate([color_features, pos_enc, dir_enc])
        color_emission = mlp(mlp_input)
        
        # 6. Accumulate
        weight = accumulated_transmittance * alpha
        accumulated_color += weight * color_emission
        accumulated_transmittance *= (1.0 - alpha)
        
    # 7. Add background
    accumulated_color += accumulated_transmittance * c_bg
    
    return accumulated_color
```

**Edge Cases and Pitfalls:**
- **Post-Activation Sharpness:** Applying non-linear activation functions (like softplus) *after* trilinear interpolation (post-activation) produces sharper surfaces than pre-activation. This allows the use of fewer grid cells for surface detail [S1].
- **Precision:** The exponential function in $\alpha_i$ and $T_i$ can suffer from underflow or overflow if $\sigma_i \delta_i$ is very large or very small. Standard numerical stability techniques (e.g., log-space accumulation) should be considered, though the source notes do not specify them [S1].
- **Boundary Conditions:** The interpolation must handle points outside the grid bounds appropriately (e.g., clamping or treating as zero density).

## Operating characteristics
- **Time Complexity:** $O(n \cdot N)$ where $n$ is the number of samples per ray and $N$ is the number of pixels. The MLP inference is constant time per sample relative to grid size.
- **Memory Complexity:** $O(N_x N_y N_z \cdot (1 + C))$ for storing the density and feature grids. This is significantly less than dense voxel grids used in traditional methods if the scene is sparse, but more than pure neural representations for simple scenes.
- **Performance:** DVGO is generally faster to train than standard NeRF because the density grid can be optimized directly and the MLP is shallow. Inference speed depends on the number of samples $n$ and the cost of the MLP.
- **Strengths:** High-quality view-dependent rendering; efficient storage compared to dense voxel grids; sharp surfaces with fewer cells due to post-activation.
- **Limitations:** Requires careful tuning of grid resolution and hyperparameters (like $b$); may struggle with very large scenes without hierarchical structures.

## Background
DVGO was introduced in the paper "NeRF Synthesis with Shading Guidance" [S1]. It builds upon the principles of NeRF but introduces a hybrid representation using voxel grids for density and features. The method leverages the shifted softplus activation from Mip-NeRF [S1] to ensure non-negative densities. The key insight is that post-activation interpolation leads to sharper surfaces, reducing the need for high-resolution grids.

## Sources
[S1] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
