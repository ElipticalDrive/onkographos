# ProbeSDF

Also called: Light Field Probes For Neural Surface Reconstruction

## Overview
ProbeSDF is a neural surface reconstruction method that decouples spatial and angular light field components using two volumetric grids of different resolutions to achieve real-time rendering and accelerated training [S1]. It addresses the computational bottleneck of standard SDF-based neural fields by replacing the core appearance MLP with a lightweight model that uses spherical harmonic light field probes for angular variation and planar factorized features for spatial texture [S1]. Unlike Plenoxels or 3D Gaussian Splatting which require significantly more parameters per voxel (27 and 48 respectively), ProbeSDF uses as few as four parameters per voxel while maintaining high expressiveness [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera poses, and an initial sparse voxel grid structure.
*   **Outputs:** A signed distance function (SDF) field, spatial feature grid, angular feature grid (probes), and optionally per-camera bias vectors. The final output is rendered images via volume rendering.

**Data Structures:**
1.  **SDF Grid:** Implicit representation of 3D shape. Stores SDF values and derived surface normals $\mathbf{n}$ [S1].
2.  **Spatial Feature Grid ($F_s$):** High-resolution grid using planar factorization. Each $16^3$ tile contains three orthogonal planes of resolution $16 \times 16$. Each plane stores $n_s$ coefficients [S1].
3.  **Angular Feature Grid ($F_a$):** Coarser grid storing light field probes. Each probe consists of spherical harmonic coefficients [S1].
    *   Probes are stored at the corners of the spatial tiles.
    *   Probes at shared corners of adjacent tiles are duplicated in memory but maintain shared values to benefit from hardware-accelerated interpolation [S1].
    *   Each probe stores $l^2 \times n_a$ coefficients, where $l$ is the spherical harmonic order and $n_a$ is the number of angular features per coefficient [S1].
4.  **MLP:** A small neural network with 2 hidden layers, 32 neurons per layer, ReLU activations, and a sigmoid activation for the final output layer [S1].

**Parameters:**
*   $n_s$: Number of spatial coefficients per probe dimension. Typical value: 4 or 8 [S1].
*   $n_a$: Number of angular features. Typical value: 4 or 8 [S1].
*   $l$: Spherical harmonic order. Up to 16 coefficients ($l=4$). Typical value: 4 [S1].
*   Per-camera bias vectors: Optional MLP bias vectors trained per camera to handle inconsistent shadows or large exposure changes (used for DTU and BMVS) [S1].

**Method Steps:**

1.  **Ray Marching and SDF Evaluation:**
    For a given camera ray, sample points $\mathbf{x}$ along the ray. Query the SDF grid to get $s(\mathbf{x})$. Compute the surface normal $\mathbf{n} = \nabla s(\mathbf{x}) / \|\nabla s(\mathbf{x})\|$ [S1].
    *   *Note:* The conversion from SDF $s(\mathbf{x})$ to density $\sigma(\mathbf{x})$ follows the standard differentiable rendering pipeline principles defined in the parent article [S1, Parent].

2.  **Spatial Feature Extraction ($F_s$):**
    For point $\mathbf{x}$, identify the containing $16^3$ tile. Retrieve the three orthogonal planes ($F_x, F_y, F_z$) for that tile. Compute the spatial feature vector via planar factorization (outer product):
    $$F_s = F_x \otimes F_y \otimes F_z$$
    This reduces dimensionality by storing three orthogonal planes per tile instead of a dense 3D grid [S1].

3.  **Angular Feature Extraction ($F_a$):**
    Compute the reflected vector $\mathbf{r}$ based on the view direction $\mathbf{v}$ and surface normal $\mathbf{n}$ (typically $\mathbf{r} = 2(\mathbf{n} \cdot \mathbf{v})\mathbf{n} - \mathbf{v}$ or similar reflection geometry, though the text specifies evaluating at $\mathbf{r}$) [S1].
    Retrieve the 8 corner probes of the containing tile in the angular grid. Tri-linearly interpolate the spherical harmonic coefficients from these 8 corners to obtain the angular feature vector $F_a(\mathbf{x}, \mathbf{r})$ [S1].

4.  **Incidence Angle Embedding:**
    Compute the power terms of the cosine angle between the view direction and the surface normal to account for grazing angle reflectivity (Fresnel approximation):
    $$\text{incidence} = [(1 - \mathbf{n} \cdot \mathbf{v})^0, (1 - \mathbf{n} \cdot \mathbf{v})^1, \dots, (1 - \mathbf{n} \cdot \mathbf{v})^5]$$
    The term $(1 - \mathbf{n} \cdot \mathbf{v})$ approximates $\cos \theta$ where $\theta$ is the angle of incidence [S1].

5.  **Radiance Decoding:**
    Concatenate $F_s$, $F_a$, and the incidence angle embeddings. Pass them through the tiny MLP to predict the outgoing radiance $C$:
    $$C = MLP(F_s(\mathbf{x}), F_a(\mathbf{x}, \mathbf{r}), (1 - \mathbf{n} \cdot \mathbf{v})^0, \dots, (1 - \mathbf{n} \cdot \mathbf{v})^5, \dots)$$
    The MLP acts as a minimal BRDF decoder [S1].

6.  **Volume Rendering:**
    Integrate the predicted radiance $C$ and the density derived from the SDF along the ray to produce the final pixel color [S1, Parent].

7.  **Optimization:**
    Optimize the SDF grid values, spatial features, angular probe coefficients, and optionally per-camera bias vectors using gradient descent to minimize the difference between rendered and input images [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def probe_sdf_render(ray_origin, ray_direction, sdf_grid, spatial_grid, angular_grid, 
                     tile_size=16, ns=4, na=4, l=4, mlp_weights=None):
    """
    Renders a single ray using ProbeSDF.
    
    Args:
        ray_origin: (3,) array, camera ray origin.
        ray_direction: (3,) array, camera ray direction.
        sdf_grid: Sparse voxel grid storing SDF values.
        spatial_grid: Dictionary mapping tile_id to (3, 16, 16, ns) tensor.
        angular_grid: Dictionary mapping corner_id to (l^2, na) tensor.
        tile_size: Size of the spatial tile in voxels.
        ns: Number of spatial coefficients.
        na: Number of angular features.
        l: Spherical harmonic order.
        mlp_weights: Weights for the 2-layer MLP.
        
    Returns:
        color: (3,) array, rendered color.
    """
    # 1. Ray Marching (Standard SDF approach)
    # Sample points along ray, query SDF, compute density and normal
    # This part relies on the parent article's SDF rendering logic [Parent]
    samples = []
    for t in range(num_samples):
        x = ray_origin + t * ray_direction
        s_val = query_sdf(sdf_grid, x)
        normal = compute_normal(sdf_grid, x)
        density = sdf_to_density(s_val) # Implementation from parent article
        samples.append({'x': x, 'normal': normal, 'density': density})
        
    # 2. Volume Integration
    color = np.zeros(3)
    alpha_accum = 0.0
    
    for sample in samples:
        x = sample['x']
        n = sample['normal']
        sigma = sample['density']
        
        # 3. Spatial Features
        tile_id = get_tile_id(x, tile_size)
        # Retrieve 3 planes for this tile
        F_x = spatial_grid[tile_id][0] # (16, 16, ns)
        F_y = spatial_grid[tile_id][1]
        F_z = spatial_grid[tile_id][2]
        
        # Interpolate within tile to get F_x, F_y, F_z at local coordinates
        F_x_local = interpolate_3d(F_x, get_local_coords(x, tile_size))
        F_y_local = interpolate_3d(F_y, get_local_coords(x, tile_size))
        F_z_local = interpolate_3d(F_z, get_local_coords(x, tile_size))
        
        # Planar factorization: Outer product
        # F_s is a vector of size ns
        F_s = np.outer(np.outer(F_x_local, F_y_local), F_z_local).flatten()
        
        # 4. Angular Features
        # Compute reflected vector r
        # Note: Standard reflection r = 2(n.v)n - v, but ProbeSDF uses r for SH evaluation
        v = ray_direction
        r = 2 * np.dot(n, v) * n - v
        
        # Get 8 corner IDs of the tile containing x
        corner_ids = get_tile_corners(x, tile_size)
        
        # Tri-linear interpolation of SH coefficients
        # Each corner has l^2 * na coefficients
        # Interpolate to get Fa vector of size l^2 * na
        Fa = interpolate_angular_probes(angular_grid, corner_ids, x)
        
        # 5. Incidence Angle Embedding
        cos_theta = np.dot(n, v)
        incidence = np.array([(1 - cos_theta)**i for i in range(6)]) # 0 to 5
        
        # 6. MLP Decoding
        # Concatenate F_s, Fa, incidence
        input_vec = np.concatenate([F_s, Fa, incidence])
        
        # Tiny MLP: 2 hidden layers, 32 neurons, ReLU, Sigmoid output
        # Assuming input_dim = ns + l^2*na + 6
        # Layer 1
        h1 = relu(np.dot(input_vec, mlp_weights['W1']) + mlp_weights['b1'])
        # Layer 2
        h2 = relu(np.dot(h1, mlp_weights['W2']) + mlp_weights['b2'])
        # Output
        C = sigmoid(np.dot(h2, mlp_weights['W3']) + mlp_weights['b3'])
        
        # 7. Composite
        alpha = 1.0 - np.exp(-sigma * dt) # dt is step size
        weight = alpha * (1.0 - alpha_accum)
        color += weight * C
        alpha_accum += weight
        
        if alpha_accum > 0.99:
            break
            
    return color

def interpolate_angular_probes(angular_grid, corner_ids, x):
    """
    Tri-linearly interpolates spherical harmonic coefficients from 8 corners.
    """
    # Get local coordinates within the tile for interpolation weights
    local_coords = get_local_coords(x, tile_size)
    
    # Retrieve coefficients for each corner
    # Each corner has shape (l^2, na)
    coeffs = [angular_grid[cid].flatten() for cid in corner_ids]
    
    # Tri-linear interpolation weights based on local_coords
    w = compute_trilinear_weights(local_coords)
    
    # Weighted sum
    Fa = np.zeros(len(coeffs[0]))
    for i, c in enumerate(coeffs):
        Fa += w[i] * c
        
    return Fa

def compute_trilinear_weights(local_coords):
    """
    Computes 8 weights for trilinear interpolation.
    local_coords is normalized [0, 1] within the tile.
    """
    x, y, z = local_coords
    # Weights for corners (0,0,0) to (1,1,1)
    w000 = (1-x)*(1-y)*(1-z)
    w100 = x*(1-y)*(1-z)
    w010 = (1-x)*y*(1-z)
    w110 = x*y*(1-z)
    w001 = (1-x)*(1-y)*z
    w101 = x*(1-y)*z
    w011 = (1-x)*y*z
    w111 = x*y*z
    return [w000, w100, w010, w110, w001, w101, w011, w111]
```

## Operating characteristics
*   **Time Complexity:** Training converges in minutes (e.g., ~1 min for MVMannequins, ~150s for DTU) compared to hours for baselines like Voxurf or NeuS2 [S1]. Rendering is real-time (200-400 Hz on RTX A6000) [S1].
*   **Memory Complexity:** Highly memory efficient. Probes require 1/6th of the memory taken by spatial features when $l=4$ and $n_s=n_a$ [S1]. Model sizes are small (e.g., 30MB for MVMannequins, 56-88MB for DTU) [S1]. VRAM usage is low (e.g., 5GB for MVMannequins) [S1].
*   **Performance:**
    *   **MVMannequins:** PSNR 36.81-36.90, Chamfer 1.03-1.04mm [S1].
    *   **ActorsHQ:** PSNR 37.48 (r/1), 36.62 (r/4). Model size 232MB (r/1), 57MB (r/4) [S1].
    *   **DTU:** PSNR 37.03-37.74, Chamfer 0.68-0.71mm. Twice as fast as NeuS2 [S1].
    *   **BlendedMVS:** PSNR 35.19-35.89, Chamfer 2.22-2.37mm [S1].
*   **Strengths:** Real-time rendering, fast training, low memory footprint, high PSNR and geometric accuracy on generic objects and humans with mild specularities [S1].
*   **Limitations:**
    *   Lighting encoded locally degrades extrapolation capabilities, causing shadow artifacts when looking from above or below on datasets with limited camera overlap along latitude (e.g., ActorsHQ legs) [S1].
    *   Local minima due to shape-radiance ambiguities because light transport is not explicitly simulated [S1].
    *   Relighting and material editing are not supported [S1].
    *   Targets objects with mild specularities; cone tracing methods are better for strong reflections but require >10 GPU-hours [S1].

## Background
ProbeSDF was introduced in the paper "ProbeSDF: Light Field Probes For Neural Surface Reconstruction" [S1]. It builds on the hypothesis that angular components (lighting/environment) are predominantly extrinsic to the surface and locally invariant to parallax, allowing them to be encoded at a lower spatial resolution than intrinsic spatial texture [S1]. The method uses a sparse voxel grid based on Voxurf with targeted modifications to implement probes [S1]. It is evaluated on four representative datasets: MVMannequins, ActorsHQ, DTU, and BlendedMVS [S1].

## Sources
[S1] ProbeSDF: Light Field Probes For Neural Surface Reconstruction, https://doi.org/10.1109/CVPR52734.2025.01030, fetched 2026-10-26
