# Neural Sparse Voxel Fields (NSVF)

Also called: Sparse Voxel Fields

## Overview
Neural Sparse Voxel Fields (NSVF) is a hybrid neural scene representation that accelerates free-viewpoint rendering by combining the high-quality implicit modeling of Neural Radiance Fields (NeRF) with an explicit sparse voxel grid structure [S3, S4, unofficial]. It solves the problem of inefficient sampling in empty space inherent to dense NeRF by restricting ray-marching computations to non-empty voxels, thereby enabling faster inference and higher resolution details through progressive training [S1, S3, S5, unofficial]. Unlike NeRF, which uses a monolithic MLP for the entire volume, NSVF decomposes the scene into local voxel-bounded implicit functions, allowing parameter growth to be additive rather than multiplicative [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed 2D images (camera poses and RGB values), initial coarse geometry (optional), and hyperparameters (see Parameters).
*   **Outputs:** Rendered RGB images and depth maps for novel viewpoints.

**Data Structures:**
The scene is represented by a **Sparse Voxel Grid**.
*   **Voxel:** A cubic volume element. Each non-empty voxel stores a learnable feature vector (embedding) at its 8 vertices.
*   **Feature Vector:** A 32-dimensional vector per vertex [S1, unofficial].
*   **Storage:** Only voxels containing geometry or significant density are stored. Empty voxels are pruned [S3, unofficial].
*   **Bounding Box:** An initial axis-aligned bounding box (AABB) encloses the scene with sufficient margin, subdivided into approximately 1000 initial voxels [S3, unofficial].

**Method Steps:**

1.  **Voxel Initialization:**
    *   If coarse geometry is provided, initialize voxels by voxelizing the coarse geometry [S3, unofficial].
    *   Otherwise, subdivide the initial bounding box into ~1000 voxels and initialize vertex features [S3, unofficial].

2.  **Ray-Voxel Intersection:**
    *   For each ray, perform an Axis-Aligned Bounding Box (AABB) intersection test to determine entry into the voxel grid [S3, unofficial].
    *   Identify the sequence of non-empty voxels intersected by the ray [S1, unofficial].

3.  **Ray Marching and Sampling:**
    *   Inside each intersected voxel, uniformly sample points along the ray segment contained within the voxel [S3, unofficial].
    *   The number of coarse samples per ray is $N_{samples}$ (default 64) [S2, unofficial].
    *   If importance sampling is enabled ($N_{importance} > 0$), generate fine samples based on coarse weights [S2, unofficial].

4.  **Feature Interpolation and Embedding:**
    *   For a query point $p$ inside voxel $V_i$, retrieve the 8 corner vertex features of the containing voxel [S1, unofficial].
    *   Perform trilinear interpolation of these 8 features to obtain the voxel embedding $E_v$ for point $p$ [S3, unofficial].
    *   Apply positional embedding (e.g., Fourier features) to the interpolated embedding and the point coordinates [S3, unofficial].
    *   Concatenate the embedded position with the embedded ray direction $d$ [S2, unofficial].

5.  **Neural Network Inference:**
    *   Pass the concatenated embedding through a query MLP to predict volume density $\sigma$ and RGB color $C$ [S3, unofficial].
    *   The output shape is $(N_{rays}, N_{samples}, 4)$ for RGB+Sigma or $(N_{rays}, N_{samples})$ for Sigma only [S2, unofficial].
    *   Add noise to sigma predictions: $\sigma_{noisy} = \sigma + \text{noise}$, where $\text{noise} \sim \mathcal{N}(0, \text{noise\_std}^2)$ [S2, unofficial].

6.  **Volume Rendering Integration:**
    *   Compute deltas $\delta_i$ (distances between samples). Set the last delta to infinity ($10^{10}$) to account for the ray extending to the far plane [S2, unofficial].
    *   Convert deltas to real-world distances by multiplying by the norm of the ray direction $\|d\|$ [S2, unofficial].
    *   Compute alpha (transmittance weight) for each sample:
        $$ \alpha_i = 1 - \exp(-\delta_i \cdot \text{ReLU}(\sigma_i + \text{noise})) $$
    *   Compute accumulated color $C$ and depth $D$ using standard volume rendering equations:
        $$ C = \sum_{i=1}^{N} T_i \alpha_i C_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        $$ D = \sum_{i=1}^{N} T_i \alpha_i z_i $$
        where $z_i$ is the depth of sample $i$ [S2, unofficial].
    *   Define a learnable background RGB color for rays that miss the object within the voxel [S1, unofficial].

7.  **Early Termination:**
    *   Terminate volume accumulation when the accumulated transparency $A$ (or transmittance $T$) drops below a threshold $\varepsilon$ (default 0.01) [S3, unofficial]. This avoids unnecessary computation behind the surface [S1, unofficial].

8.  **Self-Pruning:**
    *   Periodically (e.g., every 2.5K iterations), check the maximum predicted density from sampled points inside each voxel [S3, unofficial].
    *   If all uniformly sampled points within a voxel have very low volume density, prune the voxel (remove it from the sparse grid) [S1, unofficial].

9.  **Progressive Training:**
    *   Halve the voxel size and ray-marching step size at specific intervals (e.g., 5K, 25K, 75K iterations) [S3, unofficial].
    *   Subdivide each voxel into 8 sub-voxels [S1, unofficial].
    *   Initialize the feature representations of the new vertices via trilinear interpolation of the feature representations at the original eight voxel vertices [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ray_voxel_intersection(ray_origin, ray_direction, voxel_grid):
    """
    Perform AABB intersection test for a ray against the sparse voxel grid.
    Returns a list of intersected voxel IDs and entry/exit depths.
    """
    # Implementation depends on specific sparse grid structure (e.g., hash map or octree)
    # For NSVF, this typically involves checking the bounding box and then traversing
    # the sparse structure to find non-empty voxels along the ray.
    intersected_voxels = []
    # ... (Specific traversal logic for sparse grid)
    return intersected_voxels

def trilinear_interpolate(features, point_coords, voxel_bounds):
    """
    Interpolate vertex features for a point inside a voxel.
    features: Tensor of shape (8, embedding_dim) for the 8 corners of the voxel.
    point_coords: Tensor of shape (3) normalized to [0, 1] within the voxel.
    voxel_bounds: Bounds of the voxel in world space.
    """
    # Normalize point to local voxel coordinates [0,1]
    local_coords = (point_coords - voxel_bounds.min) / (voxel_bounds.max - voxel_bounds.min)
    
    # Trilinear interpolation
    # ... (Standard trilinear interpolation logic)
    interpolated_feature = ...
    return interpolated_feature

def fourier_embedding(x, L=6):
    """
    Apply Fourier feature embedding to input x.
    L: Maximum frequency.
    """
    freq_bands = 2 ** torch.linspace(0, L-1, L)
    out = [x]
    for freq in freq_bands:
        out.append(torch.sin(freq * x))
        out.append(torch.cos(freq * x))
    return torch.cat(out, dim=-1)

def render_ray(ray_origin, ray_direction, voxel_grid, model, N_samples=64, epsilon=0.01):
    """
    Render a single ray using NSVF.
    """
    # 1. Ray-Voxel Intersection
    intersected_voxels = ray_voxel_intersection(ray_origin, ray_direction, voxel_grid)
    
    if not intersected_voxels:
        return model.background_color, 0.0

    # 2. Sampling
    # Determine total length of ray within voxels
    total_length = sum([v.exit_depth - v.entry_depth for v in intersected_voxels])
    
    # Uniform sampling within intersected voxels
    # ... (Generate sample points along the ray within the intersected voxels)
    sample_points = ...
    sample_depths = ...
    
    # 3. Feature Interpolation and Embedding
    # For each sample point, find the containing voxel and interpolate features
    embedded_points = []
    for point in sample_points:
        voxel = find_containing_voxel(point, voxel_grid)
        if voxel is None:
            continue
        features = voxel.vertex_features # Shape (8, 32)
        local_coords = get_local_coords(point, voxel)
        interp_feature = trilinear_interpolate(features, local_coords, voxel.bounds)
        # Apply Fourier embedding
        emb_point = fourier_embedding(point)
        emb_dir = fourier_embedding(ray_direction)
        emb_feature = fourier_embedding(interp_feature)
        embedded_points.append(torch.cat([emb_point, emb_dir, emb_feature]))
    
    if not embedded_points:
        return model.background_color, 0.0
        
    embedded_points = torch.stack(embedded_points)
    
    # 4. Neural Network Inference
    # Chunk inference to manage memory
    chunk_size = 32768
    sigmas = []
    rgbs = []
    for i in range(0, len(embedded_points), chunk_size):
        chunk = embedded_points[i:i+chunk_size]
        sigma, rgb = model(chunk)
        sigmas.append(sigma)
        rgbs.append(rgb)
        
    sigmas = torch.cat(sigmas, dim=0)
    rgbs = torch.cat(rgbs, dim=0)
    
    # 5. Volume Rendering
    # Add noise
    noise_std = 0.01 # Example value
    sigmas = sigmas + torch.randn_like(sigmas) * noise_std
    
    # Compute deltas
    deltas = torch.diff(sample_depths)
    deltas = torch.cat([deltas, torch.tensor([1e10])]) # Last delta to infinity
    
    # Convert to real-world distances
    deltas = deltas * torch.norm(ray_direction)
    
    # Compute alphas
    alphas = 1 - torch.exp(-deltas * torch.relu(sigmas))
    
    # Compute transmittance
    transmittance = torch.cumprod(1 - alphas, dim=0)
    transmittance = torch.cat([torch.tensor([1.0]), transmittance[:-1]])
    
    # Early termination
    # ... (Stop accumulation if transmittance < epsilon)
    
    # Accumulate color and depth
    weights = alphas * transmittance
    color = torch.sum(weights.unsqueeze(-1) * rgbs, dim=0)
    depth = torch.sum(weights * sample_depths, dim=0)
    
    return color, depth

def progressive_training_step(voxel_grid, iteration):
    """
    Halve voxel size and prune empty voxels at specific iterations.
    """
    if iteration in [5000, 25000, 75000]:
        # Halve voxel size
        new_voxel_grid = split_voxels(voxel_grid)
        # Interpolate features
        new_voxel_grid = interpolate_features(new_voxel_grid)
        voxel_grid = new_voxel_grid
        
    if iteration % 2500 == 0:
        # Self-pruning
        voxel_grid = prune_empty_voxels(voxel_grid)
        
    return voxel_grid
```

**Parameters:**
*   $N_{samples}$: Number of coarse samples per ray. Default: 64 [S2, unofficial].
*   $N_{importance}$: Number of fine samples per ray. Default: 0 [S2, unofficial].
*   $\varepsilon$: Early termination threshold for transparency. Default: 0.01 [S3, unofficial].
*   Vertex feature embedding dimension: 32 [S1, unofficial].
*   Fourier transformation maximum frequency $L$: 6 [S3, unofficial].
*   Batch size: 32 images/batch with 2048 rays/image [S3, unofficial].
*   Inference chunk size: 1024*32 (32768) [S2, unofficial].
*   Halving interval: Every 2500 steps (for pruning), specific iterations for halving (5K, 25K, 75K) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero in Importance Sampling:** When computing PDF for fine sampling, add a small epsilon ($10^{-5}$) to weights to avoid division by zero [S2, unofficial].
*   **Denominator Zero in Linear Interpolation:** If the CDF difference is zero (bin has zero weight), set denominator to 1 [S2, unofficial].
*   **Precision:** The 32-dim embedding with sinusoidal positional encoding leads to relatively large GPU time compared to lighter embeddings [S1, unofficial].
*   **Complex Backgrounds:** The current model struggles with complex backgrounds [S3, unofficial].

## Subtopics

### [SuRFNet](7895-surfnet.md)
SuRFNet is a 3D pipeline that distills multiple input image views into partial 3D Sparse Radiance Fields (SRFs) and learns within the 3D sparse voxel space to generate full SRFs from these partial inputs. It addresses the challenge of few-view novel view synthesis by ensuring multi-view consistency, particularly for out-of-distribution views, unlike 2D prior-based methods. It sets itself apart by operating directly on irregular volumetric sparse voxel structures using 3D sparse convolutions, rather than projecting 2D features or relying on dense volumetric grids.

## Operating characteristics
*   **Time Complexity:** Rendering is faster than NeRF because it skips empty space sampling [S1, S5, unofficial]. However, the use of 32-dim embeddings and Fourier features increases per-sample computation time compared to lighter embeddings [S1, unofficial].
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels, which is significantly less than dense voxel grids or the parameter count of a monolithic MLP for large scenes [S4, unofficial].
*   **Measured Performance:** On a single V100 GPU, NSVF achieves real-time rendering speeds (e.g., 2.62 s/frame for complex scenes, though this varies by resolution and scene complexity) [S3, unofficial]. PSNR of 33.58 and SSIM of 0.954 on synthetic datasets [S3, unofficial].
*   **Scaling:** Parameter growth is additive with respect to scene size, allowing scalability to large scenes [S4, unofficial].
*   **Strengths:** Efficient sampling in empty space, high-quality rendering, progressive training allows coarse-to-fine learning, explicit geometry enables spatial editing [S3, S4, unofficial].
*   **Limitations:** Struggles with complex backgrounds, relatively high GPU time due to embedding size, requires posed images for training [S1, S3, unofficial].
*   **When to Use:** When high-quality free-viewpoint rendering is needed with faster inference than NeRF, and the scene has significant empty space.
*   **When Not to Use:** When real-time performance on low-end hardware is critical (due to embedding cost) or when unposed images are the only input (NSVF requires posed images).

## Background
NSVF was introduced by Liu et al. in 2020 at NeurIPS [S3, unofficial]. It extends the ideas of Neural Radiance Fields (NeRF) by incorporating a sparse voxel grid structure to improve rendering efficiency and quality [S2, S3, unofficial]. The method was developed to address the limitations of NeRF in handling large scenes and complex geometries efficiently [S4, unofficial]. It has been used in various applications including free-viewpoint rendering, scene editing, and dynamic scene representation [S3, unofficial].

## Sources
[S1] Neural Sparse Voxel Fields, https://github.com/chaerinkong/paper-reviews/issues/85, fetched 2026-10-26 (unofficial)
[S2] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
[S3] Learning Neural Sparse Voxel Fields for Free-viewpoint ..., https://irc.cs.sdu.edu.cn/2020-summer-school/static/LingjieLiu.pdf, fetched 2026-10-29 (unofficial)
[S4] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
