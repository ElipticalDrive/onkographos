# Point Cloud Rendering

Also called: Depth-based sprite, Nailboard, Textured Particles, Textured Particles / Point Cloud Rendering

## Overview
Point cloud rendering treats raw 3D point data as discrete sprites or spheres to generate 2D images, serving as a bridge between explicit geometric representations and differentiable rendering pipelines [S3, unofficial]. Unlike mesh-based rendering which relies on rasterization of triangles, or voxel-based methods which fill space with grid cells, point cloud rendering directly projects individual points, often using spatial influence functions to handle sparsity and occlusion [S3, unofficial]. It is distinct from its sibling [Deferred Splatting with Temporal Coherency and Bi-Bi-1 Visibility Pass](12274-deferred-splatting-with-temporal-coherency-and-bi-bi-1-visib.md) by its focus on explicit point-to-pixel aggregation rather than deferred visibility buffers, and from [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md) by its specific algorithmic choices for influence calculation and occlusion handling [S3, unofficial].

## How it works
The method consists of three primary phases: projection, influence calculation, and aggregation [S3, unofficial].

### Inputs and Outputs
*   **Input:** A point cloud $P = \{P_{xyz}^{(i)}\}_{i=1}^N$, where each point has 3D coordinates and associated features (e.g., color $C_i$). Camera parameters (intrinsics $K$, extrinsics $R, t$) defining the view.
*   **Output:** A 2D image $I$ of resolution $H \times W$ with RGB channels.

### Data Structures
*   **Point Cloud:** A list or tensor of shape $(N, 3 + F)$, where $N$ is the number of points and $F$ is the number of feature channels (e.g., 3 for RGB).
*   **Camera:** Defined by projection matrix $M = K[R|t]$.
*   **Pixel Grid:** A 2D array of size $H \times W$.

### Steps

1.  **Calculate Screen Space Coordinates**
    For each 3D point $P_{xyz}^{(i)}$, compute its screen space coordinates $p_{uv}^{(i)} = (u_i, v_i)$ via matrix multiplication with the camera projection matrix $M$ [S3, unofficial]. This step is differentiable with respect to $P_{xyz}$ [S3, unofficial].
    $$
    \begin{bmatrix} u_i \\ v_i \\ w_i \end{bmatrix} = M \cdot \begin{bmatrix} P_{xyz}^{(i)} \\ 1 \end{bmatrix}
    $$
    Normalize by $w_i$ to get normalized device coordinates, then map to pixel indices.

2.  **Compute Influence**
    Calculate the spatial influence $I_i$ of each point $P_{xyz}^{(i)}$ on target pixels. A naive approach assumes a size of one pixel, but this leads to sparse images [S3, unofficial]. To solve this, influences larger than one pixel are created using:
    *   Truncated Gaussian blur [S3, unofficial].
    *   Distance-based influence values [S3, unofficial].
    
    Let $d_i$ be the distance from the pixel center to $p_{uv}^{(i)}$. The influence $I_i$ is a function of $d_i$ and potentially the point's depth.

3.  **Aggregate Points**
    Determine the final color for each pixel by aggregating the contributions of points based on their influence $I_i$ and z-values (depth) to handle occlusion [S3, unofficial]. Several aggregation strategies exist:
    *   **Weighted Sum:** Compute a weighted sum of point colors based on influences $I_i$ [S3, unofficial]. This does not account for occlusion [S3, unofficial].
    *   **Closest Point (Lin et al.):** Select the single closest point to the camera for a given pixel [S3, unofficial]. This prevents optimization of occluded points [S3, unofficial].
    *   **K-Nearest (Li et al.):** Select the $K$ nearest 3D points to the camera and weigh them based on spatial influence $I_i$ [S3, unofficial].
    *   **Distance-Weighted (Wiles et al.):** Weigh all 3D points according to distances from the camera in addition to spatial influence $I_i$ [S3, unofficial].
    *   **Voxelization (Insafutdinov et al.):** Generate 3D voxels from the point cloud and render them using a volume rendering method [S3, unofficial].

### Pseudocode
The following pseudocode implements the standard differentiable point cloud rendering pipeline using influence-based aggregation [S3, unofficial]. It assumes a framework supporting automatic differentiation.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_point_cloud(points, features, camera_matrix, image_size):
    """
    points: Tensor of shape (N, 3), 3D coordinates of points.
    features: Tensor of shape (N, C), features per point (e.g., RGB).
    camera_matrix: Matrix of shape (3, 4), projection matrix M.
    image_size: Tuple (H, W).
    """
    H, W = image_size
    N = points.shape[0]
    
    # Step 1: Project points to screen space
    # Homogeneous coordinates
    points_homo = torch.cat([points, torch.ones(N, 1)], dim=1) # (N, 4)
    projected = torch.matmul(camera_matrix, points_homo.t()) # (3, N)
    u = projected[0, :] / projected[2, :] # Normalize by depth (w)
    v = projected[1, :] / projected[2, :]
    depth = projected[2, :]
    
    # Filter points behind camera or outside bounds
    valid_mask = (depth > 0) & (u > 0) & (u < W) & (v > 0) & (v < H)
    u = u[valid_mask]
    v = v[valid_mask]
    depth = depth[valid_mask]
    features = features[valid_mask]
    
    # Step 2: Compute Influence
    # Create a grid of pixel centers
    y_coords, x_coords = torch.meshgrid(torch.arange(H), torch.arange(W), indexing='ij')
    # Flatten for broadcasting
    x_grid = x_coords.flatten().float()
    y_grid = y_coords.flatten().float()
    
    # Calculate distances from each pixel to each point
    # u, v are (N_valid,), x_grid, y_grid are (H*W,)
    # We need to compute influence for each (pixel, point) pair
    # Influence I_i based on distance d_i
    # Example: Gaussian influence
    sigma = 1.0 # Parameter: standard deviation of influence
    # Reshape for broadcasting: points (N_valid, 1, 2), pixels (1, H*W, 2)
    point_uv = torch.stack([u, v], dim=1).unsqueeze(1) # (N_valid, 1, 2)
    pixel_uv = torch.stack([x_grid, y_grid], dim=1).unsqueeze(0) # (1, H*W, 2)
    
    dist_sq = torch.sum((point_uv - pixel_uv) ** 2, dim=2) # (N_valid, H*W)
    influence = torch.exp(-dist_sq / (2 * sigma ** 2)) # (N_valid, H*W)
    
    # Step 3: Aggregate
    # Initialize image
    image = torch.zeros(H * W, features.shape[1], device=points.device)
    
    # Weighted sum aggregation (does not handle occlusion)
    # features: (N_valid, C), influence: (N_valid, H*W)
    # Result: (H*W, C)
    aggregated = torch.matmul(influence.t(), features) # (H*W, C)
    
    # Normalize by sum of influences to get weighted average
    influence_sum = torch.sum(influence, dim=0) # (H*W,)
    # Avoid division by zero
    influence_sum = torch.clamp(influence_sum, min=1e-6)
    image = aggregated / influence_sum.unsqueeze(1)
    
    # Reshape to image
    image = image.view(H, W, -1)
    
    return image
```

### Parameters
*   **Influence Size ($\sigma$):** Controls the spread of the point's contribution. Large values prevent derivatives from becoming zero for distant points but reduce rendering quality [S3, unofficial].
*   **Aggregation Method:** Choice between weighted sum, closest point, or K-nearest [S3, unofficial].

### Edge Cases and Pitfalls
*   **Sparsity:** Assuming point influence size is one pixel leads to very sparse images [S3, unofficial].
*   **Occlusion:** Simple weighted sum approaches do not take occlusion into account [S3, unofficial].
*   **Optimization Trade-off:** There is a trade-off between rendering quality and optimization convergence; large influence sizes help gradient flow but blur details [S3, unofficial].
*   **Depth Precision:** Using 3D textures for point clouds can cause resolution to lose precision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W)$ for naive influence calculation, where $N$ is the number of points and $H \times W$ is the image resolution. Spatial partitioning (e.g., octree) can reduce this [S1, unofficial].
*   **Memory Complexity:** $O(N + H \cdot W)$ for storing points and the output image. Intermediate influence matrices can be $O(N \cdot H \cdot W)$ if not optimized [S3, unofficial].
*   **Strengths:** Differentiable with respect to point positions and features, enabling use in neural network training [S3, unofficial]. Simple to implement.
*   **Limitations:** Can produce sparse images if influence is not properly sized. Occlusion handling is approximate unless specific strategies (like K-nearest) are used [S3, unofficial].
*   **When to use:** When working with raw point cloud data, especially in differentiable rendering contexts or when mesh reconstruction is not available [S3, unofficial].

## Background
Point cloud rendering has gained prominence with the advent of differentiable rendering and its application in scene understanding with reduced 3D supervision [S3, unofficial]. It is often used in self-supervised learning of monocular depth estimation, where a color image is used to estimate depth, which is then converted to a point cloud and projected to a virtual camera view [S3, unofficial]. The method is also relevant in visualizing volumetric data, where points are treated as particles or sprites [S1, unofficial].

## Sources
[S1] OpenGL/GLSL: What is the best algorithm to render clouds/smoke out of volumetric data?, https://stackoverflow.com/questions/2591956/opengl-glsl-what-is-the-best-algorithm-to-render-clouds-smoke-out-of-volumetric, fetched 2026-10-26 (unofficial)
[S2] Shreyas120/l3d_assignment3, https://github.com/Shreyas120/l3d_assignment3, fetched 2026-10-26 (unofficial)
[S3] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
