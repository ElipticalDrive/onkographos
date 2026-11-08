# Depth Reinitialization

Also called: Depth-based densification

## Overview
Depth Reinitialization is a regularization strategy that alleviates 'overlapping' and 'under-reconstruction' artifacts in 3D Gaussian Splatting by leveraging dense depth information to reorganize the spatial distribution of Gaussians [S1]. It sets itself apart from sibling methods by rejecting standard alpha-blended depth maps in favor of geometric mid-point depths to generate clean, artifact-free point clouds for reinitialization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$, camera parameters, and ground truth color images.
*   **Output:** A reorganized set of Gaussian positions and colors, initialized from a dense point cloud derived from rendered depth.

**Data Structures:**
*   **Gaussian $G_i$:** Defined by center $\mathbf{c}_i$, scale $s_i = (s_{x,i}, s_{y,i}, s_{z,i})$, rotation, opacity $\alpha_i$, and spherical harmonics coefficients for color.
*   **Ray $r(t)$:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, transformed into the local coordinate system of the Gaussian ellipsoid.

**Formulas:**
The Gaussian is modeled as an ellipsoid in 3D space [S1]:
$$ g(x, y, z) = \frac{x^2}{s_x} + \frac{y^2}{s_y^2} + \frac{z^2}{s_z} = 1 $$
where $s = (s_x, s_y, s_z)$ is the scale vector. Note: The source text contains a typo in the denominator of the $x$ term ($s_x$ vs $s_x^2$). Standard ellipsoid formulation implies squared semi-axes. However, the source explicitly writes $\frac{x^2}{s_x}$. We retain the source's notation but note that $s_x$ represents the squared semi-axis length in this specific context or the source has a typo. Given the context of "scale", standard 3DGS uses covariance matrices. The source formula is:
$$ \frac{x^2}{s_x} + \frac{y^2}{s_y^2} + \frac{z^2}{s_z} = 1 $$
*Correction/Clarification:* The source text says "scale $s=(s_x, s_y, s_z)$" and formula $\frac{x^2}{s_x} + \frac{y^2}{s_y^2} + \frac{z^2}{s_z} = 1$. This is inconsistent dimensionally if $s$ are lengths. Usually, ellipsoid is $\frac{x^2}{a^2} + \dots = 1$. If $s$ are scales, the formula should likely be $\frac{x^2}{s_x^2} + \dots$. However, the source explicitly provides the formula with mixed powers. We follow the source's explicit formula structure but assume $s_x$ in the denominator corresponds to the squared semi-axis for the x-dimension to make physical sense, or that $s_x$ is defined as the squared scale. For implementation, one must solve for the intersection of the ray with the ellipsoid defined by the Gaussian's covariance/scale.

The depth of the mid-point for Gaussian $G_i$, denoted $d_{mid}^i$, is calculated as follows [S1]:
1.  Transform ray $\mathbf{o}, \mathbf{d}$ to the ellipsoid's local coordinate system.
2.  Calculate the two intersection points $t_1, t_2$ of the ray with the ellipsoid surface.
3.  The mid-point depth is the average of the intersection depths along the ray direction.

For multiple Gaussians contributing to a pixel, the method selects the Gaussian with the maximum weight (contribution) to avoid alpha blending artifacts [S1]:
$$ i_{max} = \arg \max_i w_i $$
$$ d_{mid} = d_{mid}^{i_{max}} $$
where $w_i$ is the contribution weight of Gaussian $i$ to the pixel.

**Parameters:**
*   **Total number of sampled points:** Empirically set at around 3.5 million for each scene [S1].

**Numbered Steps:**
1.  **Render Depth Map:** For each pixel, identify the Gaussian $G_{i_{max}}$ with the maximum contribution $w_i$. Calculate the mid-point depth $d_{mid}$ of this Gaussian using the ellipsoid intersection method. Store this as the depth value for the pixel.
    *   *Pitfall:* Do not use standard blending depth $d_{blend} = \sum_{i=1}^N w_i \cdot d_i$ (where $d_i$ is center depth), as it produces misleading depth maps with artifacts like depth collapse, object misalignment, and blending boundaries [S1].
2.  **Reproject to World Space:** Convert the 2D depth map values $d_{mid}$ back into 3D world coordinates using the camera intrinsic and extrinsic parameters. This generates a dense set of 3D points.
3.  **Sample Initial Points:** Randomly select a subset of these reprojected points. The total number of sampled points is approximately 3.5 million per scene [S1].
4.  **Assign Colors:** Assign the ground truth color from the corresponding pixel in the input image to each sampled point.
5.  **Reinitialize:** During optimization, repeatedly reinitialize the Gaussian representation (positions and colors) with these sampled points to reorganize the spatial distribution of Gaussians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_midpoint_depth(ray_origin, ray_direction, gaussian):
    """
    Calculates the depth of the mid-point between the two intersection points
    of a ray and a Gaussian's ellipsoid model.
    
    Args:
        ray_origin: (3,) array, ray origin in world space.
        ray_direction: (3,) array, ray direction in world space.
        gaussian: Object containing scale s=(sx, sy, sz) and transformation matrix.
        
    Returns:
        depth: float, depth of the mid-point along the ray.
    """
    # Transform ray to ellipsoid local coordinates
    # Note: The exact transformation depends on the Gaussian's rotation and translation.
    # Let T be the transformation matrix from world to local.
    local_origin = T @ ray_origin
    local_direction = T @ ray_direction # Assuming direction is not translated, or handled via rotation only
    
    # Solve for intersection of ray r(t) = local_origin + t * local_direction
    # with ellipsoid x^2/sx + y^2/sy^2 + z^2/sz = 1
    # This is a quadratic equation at^2 + bt + c = 0
    # Coefficients depend on local_direction and scale s.
    # Let a = local_direction^T * diag(1/sx, 1/sy^2, 1/sz) * local_direction
    # Let b = 2 * local_direction^T * diag(1/sx, 1/sy^2, 1/sz) * local_origin
    # Let c = local_origin^T * diag(1/sx, 1/sy^2, 1/sz) * local_origin - 1
    
    # Calculate discriminant to ensure intersection exists
    discriminant = b**2 - 4*a*c
    if discriminant < 0:
        return None # No intersection
        
    sqrt_disc = sqrt(discriminant)
    t1 = (-b - sqrt_disc) / (2*a)
    t2 = (-b + sqrt_disc) / (2*a)
    
    # Mid-point parameter
    t_mid = (t1 + t2) / 2
    
    # Depth is the distance along the ray direction from the original origin
    # In the local frame, the distance is t_mid * |local_direction|. 
    # However, depth usually refers to the z-component in camera space or distance along ray.
    # The source defines d_mid as the depth of the mid-point. 
    # Assuming standard depth definition (distance along ray):
    depth = t_mid * norm(ray_direction)
    
    return depth

def depth_reinitialization_step(gaussians, cameras, gt_images, num_samples=3500000):
    """
    Performs one step of depth reinitialization.
    """
    depth_points_world = []
    depth_colors = []
    
    for camera in cameras:
        # Render depth map using mid-point depth of max-weight Gaussian
        depth_map = render_depth_map_midpoint(gaussians, camera)
        
        # Reproject depth map to world space
        # For each pixel (u, v) with depth d:
        # 3D point P = camera.unproject(u, v, d)
        points_3d = reproject_to_world(depth_map, camera)
        colors = gt_images[camera.id] # Ground truth colors
        
        # Flatten and pair
        for p, c in zip(points_3d, colors):
            depth_points_world.append(p)
            depth_colors.append(c)
            
    # Randomly sample points
    indices = random.sample(range(len(depth_points_world)), min(num_samples, len(depth_points_world)))
    sampled_points = [depth_points_world[i] for i in indices]
    sampled_colors = [depth_colors[i] for i in indices]
    
    # Reinitialize Gaussians
    # This typically involves resetting Gaussian centers and colors to these points
    # and potentially resetting other attributes or keeping them.
    reinitialize_gaussians(sampled_points, sampled_colors)
```

**Edge Cases and Pitfalls:**
*   **Blending Depth Artifacts:** Using standard blending depth $d_{blend} = \sum w_i d_i$ introduces noise from depth artifacts (depth collapse, object misalignment) which degrades reinitialization quality [S1].
*   **Center vs. Mid-point:** While depth from Gaussian centers yields comparable rendering metrics (SSIM/PSNR) to mid-point, mid-point yields superior dense point cloud reconstruction [S1]. Mid-point is preferred for reinitialization.
*   **Precision:** The discriminant check in ellipsoid intersection is crucial to handle non-intersecting rays [S1].

## Operating characteristics
*   **Time Complexity:** The method adds overhead during the optimization loop for rendering depth maps and reprojecting points. The reinitialization step itself is $O(N)$ where $N$ is the number of sampled points (3.5M).
*   **Memory Complexity:** Requires storing the depth map and the sampled point cloud (3.5M points). Each point has 3D position and color (RGB), so roughly $3.5 \times 10^6 \times 12$ bytes $\approx 42$ MB for positions and colors, plus overhead.
*   **Measured Performance:** Adding Depth Reinitialization to Blur Split increases the number of Gaussians from 3.74M to 4.32M, SSIM from 0.819 to 0.832, and LPIPS from 0.195 to 0.175 [S1].
*   **Strengths:** Alleviates overlapping and under-reconstruction; produces dense and uniform point distributions; improves LPIPS (human perception).
*   **Limitations:** Increases Gaussian count; relies on accurate depth rendering; sensitive to the choice of depth metric (mid-point vs center vs blend).
*   **When to use:** When high-quality dense point cloud reconstruction is desired alongside rendering, or when facing severe overlapping/under-reconstruction issues.
*   **When not to use:** When computational resources are extremely limited and Gaussian count must be kept low, or if depth accuracy is poor.

## Background
Depth Reinitialization was introduced in the context of "Mini-Splatting," a method to represent scenes with a constrained number of Gaussians [S1]. It builds on the observation that utilizing depth to enhance neural rendering is effective, as seen in NeRF-based algorithms [S1]. The specific formulation using mid-point depths for 3D Gaussian Splatting was proposed to address the limitations of alpha-blended depth maps in Gaussian-based representations [S1].

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
