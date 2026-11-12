# Neural Multi-View Image Reconstruction with Hash Encoding

Also called: Ours (multi-view)

## Overview
This method reconstructs 3D geometry and synthesizes novel views from multi-view images by combining a hash-encoded implicit surface representation with a neural radiance field [S1]. It differs from standard NeRF by replacing volume density with a Signed Distance Function (SDF) to enforce explicit geometric constraints, and by using multi-scale hash encoding to capture high-frequency details more efficiently than dense voxel grids or standard MLPs [S1]. Unlike NeuS, which also uses SDFs, this approach utilizes a hierarchical hash grid for the geometry network, resulting in faster training and lower memory usage while preserving fine geometric details in textureless areas [S1].

## How it works
The method consists of two main networks: a geometry network that predicts the SDF and a light field network that predicts color. The geometry network uses hash encoding to map 3D positions to features, while the light field network conditions on view direction and surface normal.

### Inputs and Outputs
**Inputs:**
*   A set of $N$ calibrated images with known camera poses.
*   A bounding volume $\Omega$ containing the scene.
*   Ray origins $\mathbf{o}$ and directions $\mathbf{d}$ for sampling.

**Outputs:**
*   Novel view images synthesized via volume or surface rendering.
*   A 3D mesh extracted from the zero-level set of the SDF.

### Data Structures

**1. Hash Encoding Grid**
The geometry representation relies on a multi-scale hash grid.
*   **Structure:** A list of $L$ hash tables (layers).
*   **Layers:** $L = 16$ [S1].
*   **Feature Dimension:** Each grid cell stores a feature vector of size $F = 2$ [S1].
*   **Hashing:** Spatial coordinates are hashed to indices in these tables. The specific hash function and resolution per level are not explicitly detailed in the provided text, but the structure is defined as 16 layers with 2 features per grid feature [S1].

**2. Network Architectures**
*   **Geometry Network:**
    *   **Type:** MLP with 6 layers [S1].
    *   **Activation:** Softplus with $\beta = 100$ (inherited from point cloud architecture description, applied to geometry network in multi-view task) [S1].
    *   **Input:** Fourier positional encoding of spatial location $\mathbf{x}$ (dimension 6) [S1].
    *   **Internal Connection:** The input to the third layer concatenates the hash encoding of $\mathbf{x}$ and the output of the second hidden layer [S1].
    *   **Output:** SDF value $S(\mathbf{x})$ and a geometry feature vector $\mathbf{f}_{geo}$ [S1].
*   **Light Field Network:**
    *   **Type:** MLP with 4 hidden layers [S1].
    *   **Hidden Units:** 256 units per layer [S1].
    *   **Input:** Concatenation of:
        1.  Fourier positional encoding of view direction $\mathbf{v}$ (dimension 6) [S1].
        2.  Fourier positional encoding of surface normal $\mathbf{n}$ (dimension 6) [S1].
        3.  Geometry feature vector $\mathbf{f}_{geo}$ from the geometry network [S1].
    *   **Output:** Color $\mathbf{c}$ [S1].

### Training Objective
The method minimizes a loss function composed of image reconstruction loss, SDF regularization, and consistency loss. The weights are set to $\alpha = 0.1$, $\beta = 0.1$, and $\gamma = 5 \times 10^{-4}$ [S1]. The specific form of Eq. (8) is not fully provided in the text, but the weights correspond to the terms in the total loss [S1].

### Algorithm Steps

1.  **Initialization:** Initialize the geometry network using a modified strategy from [12] (as referenced in [S1]) and the light field network. Initialize hash grid features to zero or small random values [S1].
2.  **Sampling:** For each training iteration, sample rays from the input views. Use a hierarchical sampling strategy to sample points $\mathbf{x}_i$ along each ray [S1].
    *   *Note:* Hierarchical sampling typically involves coarse and fine stages, but the exact sampling distribution parameters are not given [G].
3.  **Geometry Evaluation:** For each sampled point $\mathbf{x}_i$:
    *   Compute Fourier positional encoding $\gamma(\mathbf{x}_i)$.
    *   Compute hash encoding $H(\mathbf{x}_i)$.
    *   Pass through Geometry Network to get SDF $S_i$ and feature $\mathbf{f}_i$.
4.  **Invalid Point Elimination:** Calculate the mean SDF of the sampled points on the ray. Use this mean as a threshold to eliminate invalid sampled points (points far from the surface) [S1].
5.  **Normal Computation:** Compute the surface normal $\mathbf{n}_i$ at each valid point by taking the gradient of the SDF: $\mathbf{n}_i = \nabla S(\mathbf{x}_i)$ [S1].
6.  **Color Prediction:** Pass $\mathbf{n}_i$, view direction $\mathbf{v}$, and $\mathbf{f}_i$ through the Light Field Network to predict color $\mathbf{c}_i$ [S1].
7.  **Rendering:** Render the image using volume rendering or surface rendering based on the SDF and colors [S1].
8.  **Loss Calculation:** Compute the total loss with weights $\alpha=0.1, \beta=0.1, \gamma=5 \times 10^{-4}$ [S1].
9.  **Optimization:** Update network parameters using gradient descent.
10. **Mesh Extraction:** After training, extract the final geometry using the marching cubes algorithm at a resolution of $512^3$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class HashGrid:
    """
    Simplified representation of the multi-scale hash grid.
    In practice, this uses CUDA kernels for efficient hashing and interpolation.
    """
    def __init__(self, num_layers=16, feature_dim=2):
        self.num_layers = num_layers
        self.feature_dim = feature_dim
        # Initialize grid features. Size depends on hash table capacity.
        # Not given in sources, assumed to be initialized to 0 or small random.
        self.grids = [nn.Parameter(torch.zeros(2**19, feature_dim)) for _ in range(num_layers)]

    def forward(self, x):
        """
        x: (N, 3) coordinates
        Returns: (N, num_layers * feature_dim) concatenated features
        """
        # Hashing and interpolation logic is implementation-specific.
        # We assume a function `hash_and_interpolate` exists.
        features = []
        for i in range(self.num_layers):
            # Placeholder for actual hash grid lookup
            feat = hash_and_interpolate(self.grids[i], x, level=i)
            features.append(feat)
        return torch.cat(features, dim=-1)

class GeometryNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.num_layers = 6
        self.hidden_dim = 128 # Inherited from point cloud architecture description
        self.hash_grid = HashGrid(num_layers=16, feature_dim=2)
        
        # MLP layers
        self.fc1 = nn.Linear(6, self.hidden_dim) # Input: Fourier pos encoding of x
        self.fc2 = nn.Linear(self.hidden_dim, self.hidden_dim)
        # Layer 3 input is concatenated hash features
        self.fc3 = nn.Linear(self.hidden_dim + 32, self.hidden_dim) # 16 layers * 2 features
        self.fc4 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc5 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc6 = nn.Linear(self.hidden_dim, 1) # Output: SDF
        
        # Softplus activation
        self.softplus = nn.Softplus(beta=100)

    def forward(self, x):
        # Fourier positional encoding of x (dim 6)
        x_enc = fourier_pos_encoding(x, num_freqs=6)
        
        h1 = self.softplus(self.fc1(x_enc))
        h2 = self.softplus(self.fc2(h1))
        
        # Hash encoding of x
        h_hash = self.hash_grid(x)
        
        # Concatenate h2 and hash features for layer 3
        h3_input = torch.cat([h2, h_hash], dim=-1)
        h3 = self.softplus(self.fc3(h3_input))
        h4 = self.softplus(self.fc4(h3))
        h5 = self.softplus(self.fc5(h4))
        
        sdf = self.fc6(h5)
        
        # Compute geometry feature vector (output of connected layer, likely h5 or h3)
        # Source says "geometry feature vector output from the connected layer of the geometry network"
        # Assuming h5 is the final hidden state before SDF output
        geo_feat = h5 
        
        return sdf, geo_feat

class LightFieldNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.num_layers = 4
        self.hidden_dim = 256
        
        self.fc1 = nn.Linear(6 + 6 + 128, self.hidden_dim) # v(6) + n(6) + geo_feat(128)
        self.fc2 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc3 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc4 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc5 = nn.Linear(self.hidden_dim, 3) # Output: Color RGB
        
        self.relu = nn.ReLU()

    def forward(self, v, n, geo_feat):
        v_enc = fourier_pos_encoding(v, num_freqs=6)
        n_enc = fourier_pos_encoding(n, num_freqs=6)
        
        input_vec = torch.cat([v_enc, n_enc, geo_feat], dim=-1)
        
        h = self.relu(self.fc1(input_vec))
        h = self.relu(self.fc2(h))
        h = self.relu(self.fc3(h))
        h = self.relu(self.fc4(h))
        
        color = torch.sigmoid(self.fc5(h))
        return color

def train_step(ray_origins, ray_directions, gt_images, geometry_net, light_field_net):
    """
    Single training step.
    """
    # 1. Hierarchical sampling
    points, weights = hierarchical_sampling(ray_origins, ray_directions)
    
    # 2. Geometry Network Forward
    sdf, geo_feat = geometry_net(points)
    
    # 3. Eliminate invalid points using mean SDF threshold
    mean_sdf = torch.mean(sdf, dim=0, keepdim=True)
    # Thresholding logic: keep points where SDF is close to mean or within a range
    # Source says "use the mean of the SDF of sampled points as a threshold to eliminate various invalid sampled points"
    # Implementation detail: mask out points with SDF > threshold or similar
    valid_mask = torch.abs(sdf - mean_sdf) < threshold_value # Placeholder for exact threshold logic
    points = points[valid_mask]
    sdf = sdf[valid_mask]
    geo_feat = geo_feat[valid_mask]
    
    # 4. Compute Normals (Gradient of SDF)
    normals = compute_sdf_gradient(points, geometry_net)
    
    # 5. Light Field Network Forward
    colors = light_field_net(ray_directions[valid_mask], normals, geo_feat)
    
    # 6. Render Image
    rendered_image = volume_rendering(points, colors, weights, sdf)
    
    # 7. Compute Loss
    # Loss = alpha * L_image + beta * L_sdf + gamma * L_consistency
    # Weights: alpha=0.1, beta=0.1, gamma=5e-4
    loss = 0.1 * image_loss(rendered_image, gt_images) + \
           0.1 * sdf_loss(sdf) + \
           5e-4 * consistency_loss(sdf, normals)
           
    return loss

def extract_mesh(geometry_net, resolution=512):
    """
    Extract mesh using marching cubes.
    """
    # Create a 3D grid of points
    grid_points = create_grid(resolution)
    
    # Evaluate SDF on grid
    sdf_values = []
    for batch in split_into_batches(grid_points, batch_size=10000):
        sdf, _ = geometry_net(batch)
        sdf_values.append(sdf)
    sdf_grid = torch.cat(sdf_values).reshape(resolution, resolution, resolution)
    
    # Marching Cubes
    vertices, faces = marching_cubes(sdf_grid, level=0)
    return vertices, faces
```

### Parameters and Typical Values
*   **Hash Encoding Layers:** 16 [S1].
*   **Hash Encoding Feature Dimension:** 2 per grid feature [S1].
*   **Fourier Position Encoding Dimension:** 6 [S1].
*   **Light Field MLP Hidden Layer Units:** 256 [S1].
*   **Light Field MLP Layers:** 4 hidden layers [S1].
*   **Geometry Network Layers:** 6 MLP layers [S1].
*   **Loss Weights:** $\alpha = 0.1$, $\beta = 0.1$, $\gamma = 5 \times 10^{-4}$ [S1].
*   **Marching Cubes Resolution:** $512^3$ [S1].
*   **Training Hardware:** GeForce RTX 3090 GPU [S1].
*   **Rays per Iteration:** 512 [S1].

### Edge Cases and Pitfalls
*   **Textureless Areas:** Baselines like NeuS, IDR, VolSDF, and UNISURF perform poorly in textureless areas. This method mitigates this by using direct SDF constraints, but accurate normal estimation is critical [S1].
*   **Invalid Points:** The method explicitly eliminates invalid sampled points using the mean SDF as a threshold. Failure to correctly identify this threshold may lead to noise or artifacts [S1].
*   **Memory Usage:** While efficient compared to some baselines, the hash grid still consumes memory. The method uses 5 GB on RTX 3090, compared to 6.5 GB for IDR and 7 GB for NeuS [S1].

## Operating characteristics
*   **Time Complexity:** Training time is 1.8 hours on the DTU dataset, significantly faster than IDR (5.2 hours), NeuS (7.2 hours), UNISURF (21 hours), VolSDF (9 hours), and NeRF (9.1 hours) [S1].
*   **Memory Complexity:** Memory usage is 5 GB, lower than IDR (6.5 GB), NeuS (7 GB), UNISURF (6 GB), VolSDF (13.8 GB), and NeRF (8 GB) [S1].
*   **Performance:**
    *   **PSNR:** Mean PSNR of 30.85 dB, outperforming IDR (23.20 dB) and NeuS (29.66 dB) [S1].
    *   **Chamfer Distance:** Mean Chamfer-L2 distance of $0.72 \times 10^{-6}$, outperforming IDR ($0.90 \times 10^{-6}$) and NeuS ($0.77 \times 10^{-6}$) [S1].
*   **Strengths:** Reconstructs more geometric detail than baselines, especially in textureless areas and high-frequency details (e.g., bird feathers) [S1]. Faster training and lower memory usage than comparable SDF-based methods [S1].
*   **Limitations:** Requires careful tuning of the SDF threshold for invalid point elimination. Performance depends on the quality of the hash encoding implementation.

## Background
This method was proposed by Z. Deng, H. Xiao, Y. Lang, et al. in the paper "Multi-scale hash encoding based neural geometry representation" [S1]. It builds upon the concept of neural implicit surfaces (like NeuS) but introduces multi-scale hash encoding to improve efficiency and detail recovery. It is designed for multi-view image reconstruction and novel view synthesis, addressing the limitations of NeRF (rough geometry, noise) and other SDF methods (poor performance in textureless areas, high computational cost) [S1].

## Sources
[S1] Multi-scale hash encoding based neural geometry representation, https://doi.org/10.1007/s41095-023-0340-x, fetched 2026-10-26
