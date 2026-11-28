# Coordinate Rectification for Point Cloud Rendering

Also called: Surface Coordinate Estimation

## Overview
Coordinate Rectification is a surface estimation technique used in point cloud rendering to determine the precise 3D position of a ray-surface intersection from discrete point data [S2]. It solves the problem of inaccurate depth sampling in sparse or noisy point clouds by correcting the raw depth value based on the angle between the viewing ray and the scene's primary axis [S2]. Unlike standard depth buffering which assumes a planar intersection perpendicular to the view ray, this method accounts for surface orientation, enabling photorealistic rendering from discrete points by providing accurate input coordinates for subsequent neural radiance mapping [S2]. It serves as the foundational geometry estimation step in the End-to-End Rendering Pipeline, preceding the frequency-modulated radiance mapping stage [S2].

## How it works
The method estimates the 3D surface coordinate $\mathbf{x}$ for each pixel by rasterizing the point cloud to a depth buffer and applying a geometric correction factor [S2].

**Inputs and Outputs:**
*   **Input:** Point cloud $P = \{\mathbf{p}_1, \mathbf{p}_2, \dots, \mathbf{p}_N\}$ where $\mathbf{p}_k \in \mathbb{R}^3$; Camera position $\mathbf{o}$; Normalized ray directions $\mathbf{d}$; Image resolution $H \times W$ [S2].
*   **Output:** Estimated surface coordinates $\mathbf{x} \in \mathbb{R}^3$ for each pixel [S2].

**Data Structures:**
*   **Point Cloud:** List of $N$ points. Each point $\mathbf{p}_k$ has coordinates $(\mathbf{p}_k^x, \mathbf{p}_k^y, \mathbf{p}_k^z)$ [S2].
*   **Depth Buffer:** A 2D array $z \in \mathbb{R}^{H \times W}$ storing the corrected depth value for each pixel [S2].
*   **Pixel Coordinates:** Array $I \in \mathbb{R}^{H \times W \times 2}$ representing pixel centers in normalized device coordinates [S2].

**Steps:**

1.  **Coordinate Transformation:** Transform the point cloud $P$ from world space to normalized device coordinate (NDC) space such that $\mathbf{p}_k \in [-1, 1] \times [-1, 1] \times [z_{near}, z_{far}]$ [S2]. Simultaneously, transform the pixel coordinates $I$ to the same NDC space, where each pixel center $\mathbf{I}_{i,j} \in [-1, 1]^2$ [S2].
2.  **Disk Expansion:** For each point $\mathbf{p}_k$, define a disk of radius $r_k$ in the $xOy$ plane (the plane perpendicular to the z-axis) centered at $(\mathbf{p}_k^x, \mathbf{p}_k^y)$ [S2].
3.  **Depth Buffer Query:** For each pixel $(i, j)$, find the depth value $\mathbf{z}_{i,j}$ corresponding to the nearest point whose disk covers the pixel [S2]. This is calculated as:
    $$
    \mathbf{z}_{i,j} = \min_k{\mathbf{p}^z_k}, \quad \text{subject to } (\mathbf{p}_k^x - \mathbf{I}_{i,j}^x)^2 + (\mathbf{p}_k^y - \mathbf{I}_{i,j}^y)^2 < r_k^2
    $$
    where $\mathbf{z}_{i,j}$ is the depth of the nearest disk covering pixel $\mathbf{I}_{i,j}$ [S2]. If no disk covers the pixel, the depth is undefined [G].
4.  **Angle Calculation:** Compute the angle $\theta$ between the normalized viewing ray direction $\mathbf{d}$ and the scene's z-axis (pointing to the center of the scene) [S2]. Let $\mathbf{z}_{axis}$ be the unit vector $(0, 0, 1)$ in the local coordinate system aligned with the scene's primary axis. Then:
    $$
    \theta = \arccos(\mathbf{d} \cdot \mathbf{z}_{axis})
    $$
    [S2].
5.  **Coordinate Rectification:** Calculate the estimated 3D surface coordinate $\mathbf{x}$ using the rectification formula [S2]:
    $$
    \mathbf{x} = \mathbf{o} + \frac{z}{\cos \theta} \times \mathbf{d}
    $$
    where:
    *   $\mathbf{x}$ is the estimated surface coordinate [S2].
    *   $\mathbf{o}$ is the camera position [S2].
    *   $\mathbf{d}$ is the normalized ray direction [S2].
    *   $z$ is the depth value $\mathbf{z}_{i,j}$ obtained from the depth buffer [S2].
    *   $\theta$ is the angle between the ray and the z-axis [S2].

**Parameters:**
*   $r_k$: Disk radius for point $\mathbf{p}_k$. Not given in the sources.
*   $z_{near}, z_{far}$: Clipping plane distances in NDC space. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\theta = \pi/2$ (ray perpendicular to the z-axis), $\cos \theta = 0$, leading to division by zero [G]. A small epsilon or clamping of $\cos \theta$ is required [G].
*   **Occlusion and Sparsity:** The depth buffer query finds the *nearest* point in z-depth among those covering the pixel. If the point cloud is sparse, this may not represent the true surface, leading to artifacts [G].
*   **Noise:** Noisy point clouds or missing surface parts inevitably harm rendering quality or produce artifacts [S2]. The source describes an optional preprocessing module for point cloud denoising and completion to mitigate this [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def coordinate_rectification(points_ndc, radii, pixel_centers_ndc, ray_directions, camera_o):
    """
    Estimate 3D surface coordinates for each pixel using coordinate rectification.
    
    Args:
        points_ndc: Array of shape (N, 3) containing point cloud coordinates in NDC space.
        radii: Array of shape (N,) or scalar, disk radius r_k in the xOy plane.
        pixel_centers_ndc: Array of shape (H, W, 2) with pixel centers in NDC [-1, 1]^2.
        ray_directions: Array of shape (H, W, 3) with normalized viewing ray directions.
        camera_o: Array of shape (3,) camera position.
        
    Returns:
        surface_coords: Array of shape (H, W, 3) containing estimated 3D surface coordinates.
        depth_buffer: Array of shape (H, W) containing depth values z_{i,j}.
    """
    H, W = pixel_centers_ndc.shape[:2]
    N = len(points_ndc)
    
    # Initialize depth buffer with infinity
    depth_buffer = np.full((H, W), np.inf)
    
    # Step 3: Depth Buffer Query
    # For each pixel, find min z_k among points whose disk covers the pixel
    for i in range(H):
        for j in range(W):
            px, py = pixel_centers_ndc[i, j, 0], pixel_centers_ndc[i, j, 1]
            min_z = np.inf
            for k in range(N):
                pk_x, pk_y, pk_z = points_ndc[k, 0], points_ndc[k, 1], points_ndc[k, 2]
                r_k = radii[k] if np.ndim(radii) > 0 else radii
                # Check disk condition: (p^x - I^x)^2 + (p^y - I^y)^2 < r_k^2
                if (pk_x - px)**2 + (pk_y - py)**2 < r_k**2:
                    if pk_z < min_z:
                        min_z = pk_z
            if min_z < np.inf:
                depth_buffer[i, j] = min_z
                
    # Step 4: Angle Calculation
    z_axis = np.array([0.0, 0.0, 1.0])
    dot_product = np.sum(ray_directions * z_axis, axis=-1)
    dot_product = np.clip(dot_product, -1.0, 1.0)
    theta = np.arccos(dot_product)
    
    # Step 5: Coordinate Rectification
    # x = o + (z / cos(theta)) * d
    cos_theta = np.cos(theta)
    epsilon = 1e-6
    # Guard against division by zero when theta = pi/2
    cos_theta_safe = np.where(np.abs(cos_theta) < epsilon, epsilon * np.sign(cos_theta), cos_theta)
    
    scale_factor = depth_buffer / cos_theta_safe
    scale_factor_3d = np.expand_dims(scale_factor, axis=-1)
    surface_coords = camera_o + scale_factor_3d * ray_directions
    
    return surface_coords, depth_buffer
```

## Subtopics

### End-to-End Rendering Pipeline
This is a complete rendering framework that uses Coordinate Rectification as its first stage [S2]. It consists of three stages: Surface Estimation (using Coordinate Rectification), Radiance Mapping (using AFNet to predict raw RGB and feature vectors), and Refinement (using a U-Net to predict offsets) [S2]. The final RGB is the sum of raw RGB and offsets [S2]. It is trained with L2 loss and perceptual loss [S2]. This variant is chosen when a full photorealistic rendering pipeline from point clouds is required, integrating neural radiance fields with explicit geometry [S2].

## Operating characteristics
*   **Time Complexity:** The rasterization step (finding the nearest point for each pixel) is $O(H \times W \times N)$ in the naive implementation, where $H, W$ are image dimensions and $N$ is the number of points [G]. The rectification formula is $O(H \times W)$ [S2].
*   **Memory Complexity:** $O(N)$ for storing the point cloud and $O(H \times W)$ for the depth buffer and output coordinates [S2].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Provides accurate surface coordinates for neural radiance mapping, improving photorealism compared to standard depth buffering [S2]. Enables easy editing and scene composition [S2].
*   **Limitations:** Sensitive to noise and missing parts in the point cloud, which can cause artifacts [S2]. Requires preprocessing (denoising, completion) for best results [S2].
*   **When to Use:** When rendering photorealistic images from point clouds with accurate geometry estimation is required, and the point cloud is relatively clean or has been preprocessed [S2].
*   **When Not to Use:** Not given in the sources.

## Background
Coordinate Rectification was introduced in the paper "Frequency-Modulated Point Cloud Rendering with Easy Editing" [S2]. The method is part of a broader effort to combine the efficiency of point-based rendering with the photorealism of neural rendering [S2]. It addresses the limitations of traditional point cloud rendering methods, which lack view-dependent appearance and smoothness [S2]. The technique is used in the End-to-End Rendering Pipeline for object-level editing and scene composition [S2].

## Sources
[S1] Boosting Point Clouds Rendering via Radiance Mapping, https://ojs.aaai.org/index.php/AAAI/article/download/25175/24947, fetched 2026-10-27
[S2] Frequency-Modulated Point Cloud Rendering with Easy Editing, https://doi.org/10.1109/cvpr52729.2023.00020, fetched 2026-10-29
