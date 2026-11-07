# Soft Rasterizer

Also called: Liu et al. method, SoftRas, SoftRas Rasterization

## Overview
Soft Rasterizer is a differentiable rendering method that replaces the deterministic, z-buffer-based triangle selection of traditional rasterization with a probabilistic aggregation approach [S1, unofficial][S2]. It computes pixel color as a weighted sum of contributions from all mesh triangles, including occluded ones, by modeling each triangle as a probabilistic cloud with a blur radius [S1, unofficial][S2]. This allows gradients to flow to blocked and distant triangle vertices, making it particularly suitable for geometry optimization tasks where other methods fail to propagate gradients through occlusions [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A mesh defined by vertex positions $V \in \mathbb{R}^{N \times 3}$ and triangle indices; camera parameters (intrinsics and extrinsics); image resolution $(H, W)$; and a blur radius parameter $\sigma$ [S1, unofficial][S2].
*   **Outputs:** A differentiable image tensor $C \in \mathbb{R}^{H \times W \times 3}$ and its gradients with respect to the input vertices [S1, unofficial][S2].

**Data Structures:**
*   **Triangle List:** An array of triangles, each defined by three vertices.
*   **Probability Map:** For each pixel, a set of weights corresponding to each triangle, derived from spatial and depth distances [S1, unofficial][S2].

**Algorithm Steps:**
1.  **Projection:** Project all mesh vertices from 3D world space to 2D screen space using the camera projection matrix [S1, unofficial][S2].
2.  **Probabilistic Cloud Modeling:** For each triangle, model its contribution to the image plane as a probabilistic cloud. The contribution is not binary (inside/outside) but depends on the distance between the pixel and the triangle in 2D screen space [S2].
3.  **Weight Calculation:** Compute the weight $w_{ij}$ for triangle $i$ at pixel $j$ based on two factors [S1, unofficial]:
    *   The 2D spatial distance between the pixel center and the projected triangle.
    *   The distance between the camera and the triangle along the viewing direction (depth) [S1, unofficial].
    *   The specific functional form is not detailed in the provided sources, but it acts as a smooth kernel that decays with distance [S1, unofficial][S2].
4.  **Aggregation:** Fuse the color probabilities for each pixel using an aggregation function. The final pixel color $C_j$ is computed as a weighted sum of the colors of all relevant triangles [S1, unofficial]:
    $$ C_j = \sum_{i} w_{ij} C_i $$
    where $C_i$ is the color of triangle $i$ [S1, unofficial].
5.  **Differentiability:** Because the weights $w_{ij}$ are continuous functions of the vertex positions (via projection and distance calculations), the entire process is differentiable [S2]. Gradients flow from the pixel error back to all triangle vertices, including those that are occluded or far from the pixel [S2].

**Parameters:**
*   **Blur Radius ($\sigma$):** Controls the spread of the probabilistic cloud. A larger radius increases smoothness but may reduce sharpness [S2]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Blurry Edges:** The probabilistic approach results in blurry and smooth edges, removing sharp corners from the scene parameters [S1, unofficial]. This is a trade-off for differentiability [S1, unofficial].
*   **Occlusion Handling:** Unlike traditional rasterizers, Soft Rasterizer does not use z-culling. Occluded triangles still contribute to the pixel color, which is beneficial for gradient flow but computationally more expensive [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_rasterizer(vertices, triangles, camera_params, image_size, blur_radius):
    """
    Differentiable rasterization using probabilistic triangle clouds.
    
    Args:
        vertices: Tensor of shape (N, 3), vertex positions.
        triangles: Tensor of shape (M, 3), indices of vertices for each triangle.
        camera_params: Camera intrinsics and extrinsics.
        image_size: Tuple (H, W).
        blur_radius: Float, controls the probabilistic spread.
        
    Returns:
        image: Tensor of shape (H, W, 3), rendered image.
    """
    H, W = image_size
    
    # Step 1: Project vertices to 2D screen space
    projected_vertices = project_to_screen(vertices, camera_params)
    
    # Step 2: Compute triangle projections and depths
    # For each triangle, compute its 2D bounding box and depth
    triangle_projections = []
    triangle_depths = []
    triangle_colors = [] # Assuming flat shading or interpolated colors
    
    for tri_idx in range(triangles.shape[0]):
        v_indices = triangles[tri_idx]
        v_proj = projected_vertices[v_indices]
        
        # Compute 2D bounding box for culling/optimization
        min_x, max_x = v_proj[:, 0].min(), v_proj[:, 0].max()
        min_y, max_y = v_proj[:, 1].min(), v_proj[:, 1].max()
        
        # Compute average depth for the triangle
        depth = v_proj[:, 2].mean()
        
        # Compute triangle color (e.g., mean of vertex colors)
        color = vertices[v_indices].mean(dim=0)
        
        triangle_projections.append((min_x, max_x, min_y, max_y))
        triangle_depths.append(depth)
        triangle_colors.append(color)
        
    # Step 3: Initialize output image
    image = torch.zeros((H, W, 3))
    
    # Step 4: For each pixel, compute weighted sum of triangle contributions
    for y in range(H):
        for x in range(W):
            pixel_pos = torch.tensor([x, y])
            pixel_weight_sum = 0.0
            pixel_color = torch.zeros(3)
            
            for tri_idx in range(triangles.shape[0]):
                min_x, max_x, min_y, max_y = triangle_projections[tri_idx]
                
                # Check if pixel is within triangle's 2D bounding box
                if not (min_x <= x <= max_x and min_y <= y <= max_y):
                    continue
                
                # Compute 2D distance from pixel to triangle
                # This is a simplified distance metric; actual implementation may use
                # distance to triangle edges or centroid
                tri_center_2d = (projected_vertices[triangles[tri_idx][0]] + 
                                 projected_vertices[triangles[tri_idx][1]] + 
                                 projected_vertices[triangles[tri_idx][2]]) / 3.0
                dist_2d = torch.norm(pixel_pos - tri_center_2d[:2])
                
                # Compute depth distance
                depth_dist = abs(triangle_depths[tri_idx] - camera_params['z_near']) # Simplified
                
                # Compute weight based on blur radius and distances
                # The exact formula is not provided in sources, but it decays with distance
                weight = compute_probabilistic_weight(dist_2d, depth_dist, blur_radius)
                
                # Accumulate weighted color
                pixel_color += weight * triangle_colors[tri_idx]
                pixel_weight_sum += weight
                
            # Normalize by total weight to get final color
            if pixel_weight_sum > 0:
                image[y, x] = pixel_color / pixel_weight_sum
            else:
                image[y, x] = 0.0
                
    return image

def compute_probabilistic_weight(dist_2d, depth_dist, blur_radius):
    """
    Computes the weight for a triangle at a pixel based on distances.
    The exact functional form is not specified in the sources.
    [G] Assuming a Gaussian-like decay for differentiability.
    """
    # Placeholder for the actual weight function
    # Sources mention weights are based on 2D distance and depth distance
    # and that triangles are modeled as probabilistic clouds with a blur radius.
    # A common approach is:
    # weight = exp(-(dist_2d^2 + depth_dist^2) / (2 * blur_radius^2))
    # However, the exact formula is not given.
    return 1.0 / (1.0 + dist_2d + depth_dist) # Simplified fallback
```

## Operating characteristics
*   **Time Complexity:** The method requires iterating over all triangles for each pixel (or at least those within the bounding box), leading to $O(H \times W \times M)$ complexity, where $M$ is the number of triangles. This is significantly slower than traditional rasterization [S2].
*   **Memory Complexity:** Requires storing the mesh and intermediate projection data. No additional large buffers are mentioned [S2].
*   **Measured Performance:** On a 280-triangle bowl at 256x256 resolution, Jrender's SoftRas implementation takes 7.0ms for forward rendering + gradient, compared to 7.2ms for the official SoftRas [S2]. Performance scales with resolution and triangle count [S2].
*   **Strengths:**
    *   Gradients flow to occluded and distant triangles, aiding geometry optimization [S2].
    *   Completely differentiable [S2].
    *   Suitable for tasks requiring refinement of object vertices [S2].
*   **Limitations:**
    *   Produces blurry edges due to the probabilistic cloud representation [S1, unofficial].
    *   Computationally expensive due to the lack of z-culling and the need to aggregate contributions from all triangles [S2].
    *   No control over the forward pass to avoid blurriness, unlike DIB-R which separates foreground and background [S1, unofficial].

## Background
*   **Origin:** Proposed by Liu et al. [S1, unofficial][S2].
*   **History:** It is a representative method of modifying the rasterization stage to make it differentiable [S2]. It was validated in single-view mesh reconstruction and image-based shape-fitting tasks, achieving state-of-the-art results at the time [S2].
*   **Usage:** PyTorch3D provides a highly optimized implementation for SoftRas [S2]. It is used in differentiable rendering libraries for geometry optimization and shape fitting [S2].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
