# MegaNeRF

Also called: Mega-NeRF, Mega-NeRF framework

## Overview
MegaNeRF is a scalable framework for training large-scale 3D neural radiance fields (NeRFs) that supports interactive, human-in-the-loop fly-throughs of urban environments [S1]. It addresses the computational infeasibility of training a single NeRF on thousands of images by decomposing the scene into spatial cells, each represented by a separate, specialized NeRF submodule [S1, S3]. Unlike methods like DeRF and KiloNeRF that use spatial partitioning primarily to accelerate inference of a pre-trained global network, MegaNeRF uses partitioning to enable data parallelism for scalable training, as obtaining a pre-trained NeRF for large scales is infeasible [S1]. It further improves upon NeRF++ by modeling the foreground volume as an ellipsoid rather than a unit sphere to better fit aerial viewpoints and by using camera altitude constraints to avoid querying underground regions [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ training images with known camera poses (extrinsics and intrinsics) and, optionally, camera altitude measurements [S1]. For initialization, Structure-from-Motion (SfM) point clouds or geometry-aware viewpoints are used [S3].
*   **Outputs:** A set of $N$ independent NeRF submodules, each defined by an MLP $\Theta_i$, and a spatial partitioning scheme (Voronoi cells) defining the region of responsibility for each submodule [S1, S3].

**Data Structures:**
1.  **Spatial Partitioning:** The scene is divided into $N$ spatial cells using spatial Voronoi partitioning based on cluster centroids $\bar{c}_i$ [S1].
    *   **Centroids:** $\bar{c}_i = (\bar{c}_{ix}, \bar{c}_{iy}, \bar{c}_{iz})$ for $i = 1, \dots, N$. The height $z$ of centroids is often fixed to a common value if the altitude variation between camera poses is small relative to latitude/longitude differences [S1].
    *   **Assignment:** Each 3D point $p$ is assigned to the closest submodule index $i = \arg \min_{j \in \{1, \dots, N\}} \|\bar{c}_j - p\|_2^2$ [S3].
2.  **Submodule MLP:** Each submodule $i$ is an MLP $\Theta_i$ that maps a 3D position $p$ and viewing direction $d$ to volume density $\theta$ and color $r$ [S3].
    *   **Signature:** $\text{MLP}_{\Theta_i}: (p, d) \rightarrow (r, \theta)$ [S3].
3.  **Foreground/Background Decomposition:** The scene is split into a foreground volume (enclosing camera poses) and a background volume (complement) [S1].
    *   **Foreground:** Modeled by an ellipsoid parameterization (extending NeRF++'s inverse sphere) [S1, S2].
    *   **Background:** Uses the same 4D outer volume parameterization and raycasting formulation as NeRF++ [S1].
4.  **Training Data Partitioning:** A mapping from pixels to submodules. A pixel from a training image is added to the training set of submodule $i$ only if the ray corresponding to that pixel intersects the spatial cell $i$ [S1].
5.  **Rendering Cache (Optional/Advanced):** An octree caching coarse rendering of densities and colors for accelerated inference [S2].

**Numbered Steps:**

1.  **Initialization:**
    *   Estimate geometry-aware viewpoints or use SfM point clouds to determine $N$ cluster centroids $\bar{c}_i$ [S3].
    *   Define the spatial partitioning (Voronoi cells) based on these centroids [S1].
    *   Define the foreground ellipsoid bounds and background volume bounds [S1, S2].

2.  **Data Partitioning (Training Preparation):**
    *   For each training image $k$ with camera pose $C_k$:
        *   For each pixel $u$ in image $k$:
            *   Construct the ray $r_{k,u}$ from the camera center through pixel $u$.
            *   Sample points along $r_{k,u}$ within the scene bounds.
            *   Identify all spatial cells $i$ that intersect with the ray segment within the scene [S1].
            *   Add the pixel $(u, k)$ to the training dataset $\mathcal{D}_i$ for each identified submodule $i$ [S1].
    *   *Note:* This reduces the size of each submodule's training set by approximately 10x compared to using all data [S1].

3.  **Parallel Training:**
    *   Initialize $N$ independent MLPs $\Theta_1, \dots, \Theta_N$ [S3].
    *   Train each submodule $\Theta_i$ in parallel using its partitioned dataset $\mathcal{D}_i$ [S1].
    *   Loss function: Standard NeRF photometric loss (MSE between predicted and ground truth pixel colors) [S1, S3].
    *   *Optional:* Incorporate per-image appearance embeddings (latent codes) from NeRF-W to handle varying lighting conditions [S2].

4.  **Inference (Rendering):**
    *   For a new query ray $r$:
        *   Sample points $p_t$ along the ray.
        *   **Boundary Handling:** If a point $p_t$ falls within the boundaries of multiple modules (due to overlapping Voronoi regions or proximity to centroids), determine the color and density by combining outputs from relevant NeRFs in proportion to the inverse distance to the cluster centroids [S3].
            *   Let $w_i(p) = \frac{1}{\|\bar{c}_i - p\|_2^2 + \epsilon}$ be the weight for module $i$.
            *   Combined density: $\theta(p) = \frac{\sum_{i \in \text{relevant}} w_i(p) \theta_i(p)}{\sum_{i \in \text{relevant}} w_i(p)}$ [S3].
            *   Combined color: $r(p) = \frac{\sum_{i \in \text{relevant}} w_i(p) r_i(p)}{\sum_{i \in \text{relevant}} w_i(p)}$ [S3].
        *   If a point $p_t$ is clearly within one cell, query only that submodule $\Theta_i$ [S1].
        *   **Altitude Constraint:** Use camera altitude measurements to constrain ray sampling bounds. Terminate rays near ground level to avoid querying underground regions [S1, S3]. A parameter $z_{\text{ground}}$ indicates the lowest point to keep ray sampling above ground level [S3].
        *   Perform standard volume rendering (compositing) using the sampled $(r, \theta)$ values [S1].
    *   **Interactive Rendering Optimization:**
        *   Exploit temporal coherence by caching occupancy and color values from nearby previous views [S1].
        *   *Alternative/Additional:* Cache a coarse rendering of densities and colors into an octree for faster lookups [S2].

**Formulas:**
*   **Submodule Selection:** $i = \arg \min_{j \in \{1, \dots, N\}} \|\bar{c}_j - p\|_2^2$ [S3].
*   **Scene Representation:** $\text{MLP}_{\Theta_i}: (p, d) \rightarrow (r, \theta)$ [S3].
*   **Blending at Boundaries:**
    $$ \theta(p) = \frac{\sum_{i} w_i(p) \theta_i(p)}{\sum_{i} w_i(p)}, \quad r(p) = \frac{\sum_{i} w_i(p) r_i(p)}{\sum_{i} w_i(p)} $$
    where $w_i(p) = \frac{1}{\|\bar{c}_i - p\|_2^2 + \epsilon}$ [S3].

**Parameters:**
*   $N$: Number of spatial cells/submodules. Not given in the sources.
*   $z_{\text{ground}}$: Parameter indicating the lowest point to keep ray sampling above ground level. Not given in the sources.
*   $\epsilon$: Small constant for numerical stability in blending weights. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Underground Sampling:** Omission of the ground-level parameter ($z_{\text{ground}}$) leads to invalid geometry and wasted computation querying underground regions [S3].
*   **Boundary Artifacts:** Points near the boundaries of Voronoi cells may be queried by multiple submodules. The blending strategy (distance-weighted) is required to prevent visible seams [S3].
*   **Aerial Viewpoint Fit:** Using a unit sphere (as in NeRF++) for foreground modeling is less efficient for aerial drone footage than an ellipsoid, which tightly encloses the camera poses and relevant foreground detail [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_voronoi_assignment(points, centroids):
    """Assign each point to the closest centroid index."""
    dists = np.linalg.norm(points[:, np.newaxis, :] - centroids[np.newaxis, :, :], axis=2)
    return np.argmin(dists, axis=1)

def construct_ray_from_pose(camera_pose, pixel_u, pixel_v, intrinsics):
    """Construct a ray from camera center through pixel (u, v)."""
    cam_center = np.linalg.inv(camera_pose)[:3, 3]
    uv = np.array([pixel_u, pixel_v, 1.0])
    ray_dir = np.linalg.inv(intrinsics) @ uv
    ray_dir = camera_pose[:3, :3] @ ray_dir
    ray_dir = ray_dir / np.linalg.norm(ray_dir)
    return cam_center, ray_dir

def sample_uniform_points(ray_origin, ray_dir, near, far, num_samples):
    """Sample points uniformly along a ray between near and far bounds."""
    t_vals = np.linspace(near, far, num_samples)
    return ray_origin + t_vals[:, np.newaxis] * ray_dir

def filter_below_ground(points, z_ground):
    """Mask out points below the ground level."""
    return points[points[:, 2] >= z_ground]

def blend_submodule_outputs(points, centroids, mlps, ray_dir):
    """Blend outputs from relevant submodules based on inverse distance to centroids."""
    dists_sq = np.sum((points[:, np.newaxis, :] - centroids[np.newaxis, :, :])**2, axis=2)
    weights = 1.0 / (dists_sq + 1e-8)
    weights = weights / np.sum(weights, axis=1, keepdims=True)
    blended_density = np.zeros(len(points))
    blended_color = np.zeros((len(points), 3))
    for i in range(len(mlps)):
        d_i, c_i = mlps[i](points, ray_dir)
        blended_density += weights[:, i] * d_i
        blended_color += weights[:, i][:, np.newaxis] * c_i
    return blended_density, blended_color

def standard_volume_render(points, densities, colors):
    """Standard volume rendering compositing along a ray."""
    T = 1.0
    final_color = np.zeros(3)
    for i in range(len(points)):
        d = densities[i]
        c = colors[i]
        alpha = 1.0 - np.exp(-d)
        contribution = alpha * T * c
        final_color += contribution
        T *= (1.0 - alpha)
        if T < 1e-4:
            break
    return final_color

def train_mega_nerf(camera_centers, training_images, num_clusters, z_ground, num_epochs):
    """Initialize, partition data, and train MegaNeRF submodules."""
    centroids = camera_centers[:num_clusters]
    training_datasets = {i: [] for i in range(num_clusters)}
    for img_idx, (image, camera_pose, intrinsics) in enumerate(training_images):
        H, W, _ = image.shape
        for u in range(H):
            for v in range(W):
                cam_center, ray_dir = construct_ray_from_pose(camera_pose, u, v, intrinsics)
                points = sample_uniform_points(cam_center, ray_dir, near=1.0, far=100.0, num_samples=50)
                cell_indices = compute_voronoi_assignment(points, centroids)
                unique_cells = np.unique(cell_indices)
                for cell_idx in unique_cells:
                    training_datasets[cell_idx].append((img_idx, u, v))
    mlps = [None] * num_clusters
    for i in range(num_clusters):
        mlp = initialize_mlp()
        for epoch in range(num_epochs):
            for img_idx, u, v in training_datasets[i]:
                gt_color = training_images[img_idx][0][u, v]
                cam_center, ray_dir = construct_ray_from_pose(training_images[img_idx][1], u, v, training_images[img_idx][2])
                points = sample_uniform_points(cam_center, ray_dir, near=1.0, far=100.0, num_samples=128)
                points = filter_below_ground(points, z_ground)
                directions = np.tile(ray_dir, (len(points), 1))
                densities, colors = mlp(points, directions)
                loss = np.mean((colors - gt_color)**2)
                update_mlp(mlp, loss)
        mlps[i] = mlp
    return centroids, mlps

def render_mega_nerf(ray_origin, ray_dir, centroids, mlps, z_ground):
    """Render a single ray using trained MegaNeRF submodules."""
    points = sample_uniform_points(ray_origin, ray_dir, near=1.0, far=100.0, num_samples=256)
    points = filter_below_ground(points, z_ground)
    densities, colors = blend_submodule_outputs(points, centroids, mlps, ray_dir)
    final_color = standard_volume_render(points, densities, colors)
    return final_color
```

## Subtopics

### NeRFPlayer
A method for large-scale urban reconstruction from aerial drone images that extends NeRF++'s inverse sphere parameterization to ellipsoids [S2]. It focuses on caching coarse renderings into an octree for faster inference [S2]. [Link](9440-nerfplayer.md)

### Spatial Partitioning for Distributed NeRF
A method that decouples rendering time from scene size by dividing the scene into blocks, each represented by a small MLP, and aligning appearances between adjacent NeRFs for seamless integration [S7]. [Link](10332-spatial-partitioning-for-distributed-nerf.md)

## Operating characteristics
*   **Time Complexity:** Training is parallelized across $N$ submodules, reducing the training time per GPU by a factor of approximately 10x due to data partitioning [S1]. Overall training speed is improved by 3x compared to baseline approaches on existing datasets [S1].
*   **Memory Complexity:** Each submodule stores a smaller MLP, reducing memory requirements per GPU [G]. However, the total memory across all GPUs is higher than a single global NeRF [G]. The octree cache (if used) adds memory overhead proportional to the scene size [S2].
*   **Measured Performance:** Achieves a 40x speedup over conventional NeRF rendering while remaining within 0.8 dB in PSNR quality [S1][S6, unofficial]. Improves PSNR by 12% compared to baseline approaches on existing datasets [S1].
*   **Scaling:** Scales to scenes 1000x larger than prior work (e.g., 150,000 to over 1,300,000 $m^2$) [S1]. Rendering time is decoupled from scene size, allowing rendering to scale to arbitrarily large environments [S7].
*   **Strengths:** Enables training on massive datasets that are infeasible for a single NeRF. Supports interactive fly-throughs. Handles varying lighting via appearance embeddings [S2].
*   **Limitations:** Requires careful tuning of the ground-level parameter to avoid underground sampling [S3]. Boundary artifacts may occur if blending is not handled correctly [S3].

## Background
MegaNeRF was introduced in 2021/2022 by Xiangli et al. [S1][S6, unofficial]. It was motivated by the need for search-and-rescue applications where drones provide inexpensive means of quickly surveying large areas [S1]. The method builds on NeRF++ [S1, S2] by replacing the unit sphere foreground model with an ellipsoid and adding altitude constraints. It also incorporates appearance embeddings from NeRF-W [S2]. The core insight is that visibility statistics for large-scale scenes show that only a small fraction of training images are visible from any particular scene point, motivating a sparse network structure with specialized parameters for different regions [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
[S4] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30
[S6] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31 (unofficial)
