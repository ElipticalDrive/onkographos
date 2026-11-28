# Geometry Decomposition and Normal Regularization

Also called: Depth-Dependent Normal Regularization, Normal Map Computation

## Overview
This method enhances the geometric consistency of Intrinsic NeRF by computing differentiable normal maps from the volume density field and applying a depth-dependent regularization loss. It solves the problem of noisy, inconsistent geometry in complex scenes, which degrades subsequent reflectance and shading decomposition. Unlike standard NeRF variants that optimize only for RGB consistency, this approach explicitly enforces surface smoothness weighted by physical area, ensuring that background pixels (covering larger physical areas) are regularized more strongly than foreground pixels.

## How it works
The method operates on the volume density field $\sigma$ and its gradient $\nabla \sigma$ to derive geometric attributes.

**Inputs and Outputs:**
- **Input:** Camera rays $\mathcal{R}$, near/far bounds $t_n, t_f$, and the trained density field $\sigma(\mathbf{x})$.
- **Output:** Depth map $\mathbf{D}$, Normal map $\mathbf{N}$, and the regularization loss $\mathcal{L}_d$.

**Data Structures:**
- **Density Field:** A neural network or voxel grid mapping 3D coordinates $\mathbf{x}$ to scalar density $\sigma$.
- **Ray Structure:** Each ray $\mathbf{r}$ is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- **Adjacency Map:** A set of neighboring pixel indices for each pixel $\mathbf{r}$ in the image plane.

**Formulas:**
1.  **Transmittance:** $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(s) ds\right)$.
2.  **Depth Map:** The expected depth along ray $\mathbf{r}$ [S1]:
    $$ \mathbf{D}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) t dt $$
3.  **Normal Map:** The expected surface normal along ray $\mathbf{r}$, derived from the negative gradient of density [S1]:
    $$ \mathbf{N}(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(t) (-\nabla \sigma(\mathbf{x}(t))) dt $$
    where $\mathbf{x}(t) = \mathbf{o} + t\mathbf{d}$.
4.  **Depth-Dependent Normal Regularization Loss:** Encourages consistency between adjacent pixels, weighted by the inverse square of depth [S1]:
    $$ \mathcal{L}_d = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left( 1 - \frac{\mathbf{N}(\mathbf{r}) \cdot \mathbf{N}(\mathbf{r}_{adj})}{|\mathbf{N}(\mathbf{r})| |\mathbf{N}(\mathbf{r}_{adj})|} \right) \frac{1}{\mathbf{D}(\mathbf{r})^2} $$
    where $\mathbf{r}_{adj}$ represents the set of adjacent pixels to $\mathbf{r}$.

**Parameters:**
- **Sampling Steps:** Number of samples per ray for integration. Not given in the sources.
- **Adjacency Window:** Typically immediate 4- or 8-neighbors in the image grid. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_geometry_loss(rays, density_net, t_n, t_f, num_samples):
    """
    Computes depth, normal maps, and the depth-dependent normal regularization loss.
    
    Args:
        rays: List of Ray objects, each with origin o and direction d.
        density_net: Neural network returning (sigma, grad_sigma) at point x.
        t_n: Near plane distance.
        t_f: Far plane distance.
        num_samples: Number of integration steps per ray.
        
    Returns:
        L_d: Scalar regularization loss.
    """
    total_loss = 0.0
    count = 0
    
    # Pre-compute normals for all rays to access neighbors
    # In practice, this is done via batched differentiable rendering
    normals = []
    depths = []
    
    for ray in rays:
        # Sample points along the ray
        t_vals = torch.linspace(t_n, t_f, num_samples)
        points = ray.o + t_vals.unsqueeze(1) * ray.d.unsqueeze(0) # Shape: [num_samples, 3]
        
        # Query density and gradient
        # density_net returns sigma and d_sigma/dx, d_sigma/dy, d_sigma/dz
        sigma, grad_sigma = density_net(points)
        
        # Compute transmittance
        # T(t_i) = exp(-sum_{j=0}^{i-1} sigma_j * delta_t)
        deltas = t_vals[1:] - t_vals[:-1]
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(1.0 - alpha + 1e-10, dim=0)
        T = torch.cat([torch.ones(1, device=T.device), T[:-1]], dim=0)
        
        # Compute Depth
        # D(r) = sum(T(t_i) * sigma(t_i) * t_i * delta_t)
        weights = T * alpha
        depth = torch.sum(weights * t_vals)
        
        # Compute Normal
        # N(r) = sum(T(t_i) * sigma(t_i) * (-grad_sigma(t_i)) * delta_t)
        # Note: grad_sigma is already the gradient of sigma
        normal = torch.sum(weights.unsqueeze(1) * (-grad_sigma) * deltas.unsqueeze(1), dim=0)
        
        normals.append(normal)
        depths.append(depth)
        
    normals = torch.stack(normals)
    depths = torch.stack(depths)
    
    # Compute Loss
    # For each ray r, compare with adjacent rays r_adj
    # Assuming rays are ordered in image raster scan for adjacency
    # This is a simplified representation; actual implementation requires 
    # reshaping to image grid to access neighbors efficiently.
    
    # Reshape to image grid (H, W)
    # H, W = image_height, image_width
    # normals_grid = normals.view(H, W, 3)
    # depths_grid = depths.view(H, W)
    
    # Iterate over pixels (conceptually)
    for i in range(H):
        for j in range(W):
            r = rays[i * W + j]
            n_r = normals_grid[i, j]
            d_r = depths_grid[i, j]
            
            # Get adjacent pixels (e.g., 4-neighbor)
            neighbors = get_neighbors(i, j, H, W)
            
            for ni, nj in neighbors:
                n_adj = normals_grid[ni, nj]
                
                # Cosine similarity
                cos_sim = torch.dot(n_r, n_adj) / (torch.norm(n_r) * torch.norm(n_adj) + 1e-8)
                
                # Depth-dependent weight
                # Avoid division by zero
                weight = 1.0 / (d_r**2 + 1e-8)
                
                # Loss term: (1 - cos_sim) * weight
                total_loss += (1.0 - cos_sim) * weight
                count += 1
                
    return total_loss / count
```

**Edge Cases and Pitfalls:**
- **Noise:** Without this regularization, normal maps exhibit significantly more noise, particularly in complex scenes [S1].
- **Background Weighting:** A pixel in the background corresponds to a larger physical area. The $1/\mathbf{D}(\mathbf{r})^2$ weight guides regularization to account for this, preventing over-smoothing of foreground details while ensuring background consistency [S1].
- **Gradient Estimation:** The normal map relies on $\nabla \sigma$. If the density field is not smooth, the gradient will be noisy, leading to unstable normals.

## Subtopics

### DINO Feature Regularization
Regularizes reflectance estimation by distilling DINO features from 2D images to 3D surfaces. It minimizes the distance between learnable DINO features on the surface and pre-trained ViT features. Steps: 1. Distill DINO feature to the initial geometry field. 2. Minimize the loss function by comparing learnable surface features with 2D image features. 3. Incorporate distilled DINO features into networks for specular albedo and roughness to enhance material decomposition accuracy. Parameter: Images are upsampled by a scaling factor of two to extract higher resolution DINO features for finer material decoupling [S2].

## Operating characteristics
- **Time Complexity:** Adds overhead for computing gradients of density $\nabla \sigma$ and integrating normal maps. The adjacency comparison is $O(|\mathcal{R}|)$ per batch.
- **Memory Complexity:** Requires storing normal maps for the current batch.
- **Strengths:** Significantly reduces noise in normal maps, leading to smoother geometry and better material decomposition [S1].
- **Limitations:** Requires differentiable density field to compute $\nabla \sigma$. Sensitive to the choice of adjacency window and depth weighting.
- **When to use:** When high-quality geometry is required for subsequent intrinsic decomposition tasks, especially in complex scenes [S1].

## Background
Originated in the context of Intrinsic NeRF to address the ill-posed nature of inverse rendering. The depth-dependent normal regularization was introduced to improve geometric consistency by leveraging the pinhole camera model's property that background pixels cover larger physical areas [S1]. DINO feature regularization was later introduced to further constrain reflectance estimation using self-supervised 2D features [S2].

## Sources
[S1] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S2] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
