# Pulsar Sphere-based Neural Rendering

Also called: Pulsar

## Overview
Pulsar is an efficient, sphere-based differentiable renderer that maps a 3D scene representation of spheres to a feature image, enabling end-to-end optimization of scene geometry, features, and camera parameters [S1]. It solves the problem of slow rendering and topological inconsistencies inherent in mesh-based or voxel-based neural rendering by using a sparse, explicit sphere representation that naturally culls empty space without acceleration structures [S1]. Unlike its siblings, Pulsar parallelizes over pixels rather than spheres to avoid synchronization overhead, and it decouples geometry projection from appearance via a neural shading network, allowing unlimited spheres to contribute to color while restricting gradient propagation to the first few spheres to avoid high-frequency artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene representation $\mathcal{S}$ consisting of $M$ spheres; camera parameters (rotation $\mathbf{R}$, translation $\mathbf{t}$, intrinsic matrix $\mathbf{K}$); and neural shading parameters $\Theta_s$.
*   **Output:** A rendered feature image $\mathbf{F}$ or color image.

**Data Structures:**
The scene is converted from a "struct of arrays" to an "array of structs" for efficient GPU access [S1]. Two primary structures are used:
1.  **Intersection Information Struct:** Contains four `unsigned short` values representing the calculated min/max x and y limits (bounding box) of the sphere's projection on the sensor plane. Total size: 8 bytes. This fits exactly into one cache line on NVIDIA Turing GPUs (256 bytes / 32 threads = 8 bytes per thread), enabling coalesced loading [S1].
2.  **Draw Information Struct:** Contains the position vector (3 floats), up to three feature value floats or a float pointer (if >3 features), distance to sphere center, and sphere radius. Total size: 32 bytes [S1].
3.  **Depth Array:** A separate array storing the earliest possible intersection depth for each sphere (set to infinity if outside the sensor) [S1].

**Parameters:**
*   $\gamma$ (Gamma): Scaling factor for blending softness. Limits allowed: 1 and 1–5 for numerical stability. Small values (1–5) lead to 'hard' blending; large values (1) lead to 'soft' blending [S1].
*   $\xi$ (Xi): Background offset constant. A fixed small constant added to the denominator as $\exp(\xi/\gamma)$ to represent the weight for the background color [S1].
*   Block Size: 16x16 = 256 threads (8 warps per block). This size maximizes performance due to memory requirements [S1].

**Algorithm Steps:**

1.  **Scene Representation:** Each sphere $i$ is defined by position $\mathbf{p}_i \in \mathbb{R}^3$, neural feature vector $\mathbf{f}_i \in \mathbb{R}^d$, radius $r_i \in \mathbb{R}$, and opacity $o_i \in \mathbb{R}$ [S1].
2.  **Step 0: Enclosing Rectangle Calculation:**
    *   Parallelized over spheres using the full GPU.
    *   Compute the bounding box projection of each sphere on the sensor plane using $\mathbf{K}$, $\mathbf{p}_i$, and $r_i$ [S1].
    *   Robustify boundaries: Ensure every sphere is rendered with at least a size of one pixel to prevent spheres from being 'lost' between pixel rays and to ensure every sphere receives gradients [S1].
    *   Store results in `Intersection Information` and `Draw Information` structs.
    *   Store the earliest possible intersection depth in a separate array (infinity if outside sensor) [S1].
    *   Sort both arrays by earliest intersection depth using a parallel sorting library (e.g., CUB) [S1].
3.  **Step 1: Intersection Search:**
    *   Divide the image into nine parts (empirically derived). Process parts sequentially [S1].
    *   For each part, use the full GPU to iterate over the `Intersection Information` array to find spheres relevant to the pixels in that region (parallelized over spheres) [S1].
    *   Use `CUB select flags` to create sorted subsets of `Intersection Information` and `Draw Information` for the relevant spheres [S1].
    *   Parallelize over pixels using thread blocks (16x16=256 threads) and warps for coalesced processing [S1].
    *   **Early Stopping:** Stop the sphere intersection search for a pixel when the first tested sphere has a depth greater than an upper bound calculated using the current aggregated weights and Eq. 1. This ensures no remaining sphere can have a notable impact [S1].
4.  **Feature Aggregation (Blending):**
    *   Compute the feature image $\mathbf{F}$ by aggregating sphere features using blending weights $w_i$.
    *   Allow unlimited spheres to contribute to pixel color (or set a bound based on minimum contribution) to avoid high-frequency artifacts in complex scenes [S1].
    *   Use only the first few spheres for gradient propagation to avoid high-frequency artifacts [S1].
    *   **Blending Weight Formula:**
        $$w_i = \frac{o_i \cdot d_i \cdot \exp(o_i \cdot \frac{z_i}{\gamma})}{\exp(\frac{\xi}{\gamma}) + \sum_k o_k \cdot d_k \cdot \exp(o_k \cdot \frac{z_k}{\gamma})}$$
        Where:
        *   $o_i$: Sphere opacity.
        *   $d_i$: Normalized orthogonal distance from ray to sphere center.
        *   $z_i$: Normalized device coordinate depth ($[0,1]$, where 0 is max depth and 1 is min depth/closest to camera) [S1].
        *   $\gamma$: Temperature parameter for blending sharpness [S1].
        *   $\xi$: Background offset constant [S1].
        *   The sum in the denominator is over all spheres $k$ contributing to the pixel [S1].
    *   **Normalized Orthogonal Distance Formula:**
        $$d_i = \min(1, \frac{||\tilde{d}_i||_2}{R_i})$$
        Where $\tilde{d}_i$ is the vector pointing orthogonally from the ray to the sphere center, and $R_i$ is the sphere radius. This acts as a linear scaling factor in $[0, 1]$ [S1].
    *   **Opacity Integration:** Opacity $o_i$ is used in both the exponential depth term and the linear distance term to prevent spheres from being visible in front of the background (if only in exponential) or depth overriding opacity (if only in linear) [S1].
5.  **Neural Shading:**
    *   Convert the 2D screen space feature map $\mathbf{F}$ to the output color image using a neural network $\mathcal{N}$ (e.g., convolutional U-Net or per-pixel 1x1 convolution) [S1].
6.  **End-to-End Optimization:**
    *   Optimize scene representation $\mathcal{S}$ and shading parameters $\Theta_s$ by minimizing L1 loss between input images $\mathbf{I}_i$ and rendered images $\mathcal{N}(\mathcal{R}(\mathcal{S}, \mathbf{R}_i, \mathbf{t}_i, \mathbf{K}_i); \Theta_s)$ using ADAM [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def pulsar_render(spheres, camera_params, shading_net, gamma=1.0, xi=0.01):
    """
    spheres: List of Sphere objects or tensors containing p (3,), f (d,), r (1,), o (1,)
    camera_params: dict with 'R' (3x3), 't' (3,), 'K' (3x3)
    shading_net: Neural network module
    gamma: Blending softness parameter
    xi: Background offset constant
    """
    M = len(spheres)
    
    # Step 0: Enclosing Rectangle Calculation
    # Compute bounding boxes and sort spheres
    intersection_info, draw_info, depths = compute_enclosing_rectangles(spheres, camera_params['K'])
    sorted_indices = torch.argsort(depths)
    intersection_info = intersection_info[sorted_indices]
    draw_info = draw_info[sorted_indices]
    depths = depths[sorted_indices]
    
    # Step 1: Intersection Search & Feature Aggregation
    # Image dimensions
    H, W = camera_params['image_height'], camera_params['image_width']
    feature_image = torch.zeros(H, W, spheres[0].f.shape[0], device=spheres[0].f.device)
    
    # Process image in 9 parts (empirically derived)
    part_h, part_w = H // 3, W // 3
    for py in range(3):
        for px in range(3):
            y_start, y_end = py * part_h, (py + 1) * part_h
            x_start, x_end = px * part_w, (px + 1) * part_w
            
            # Find relevant spheres for this part
            # Parallelized over spheres to check if sphere bbox overlaps part bbox
            relevant_mask = check_sphere_overlap(intersection_info, y_start, y_end, x_start, x_end)
            relevant_indices = torch.where(relevant_mask)[0]
            
            if len(relevant_indices) == 0:
                continue
                
            # Select relevant spheres
            rel_intersection_info = intersection_info[relevant_indices]
            rel_draw_info = draw_info[relevant_indices]
            rel_depths = depths[relevant_indices]
            
            # Parallelize over pixels in the part
            # Use thread blocks of 16x16
            for y in range(y_start, y_end):
                for x in range(x_start, x_end):
                    # Early stopping bound calculation
                    current_weight_sum = 0.0
                    aggregated_feature = torch.zeros(spheres[0].f.shape[0], device=spheres[0].f.device)
                    
                    # Iterate through relevant spheres (sorted by depth)
                    for idx in relevant_indices:
                        # Get sphere data
                        p = rel_draw_info[idx].position
                        r = rel_draw_info[idx].radius
                        o = rel_draw_info[idx].opacity
                        f = rel_draw_info[idx].features
                        
                        # Calculate ray-sphere intersection depth z_i
                        # Note: z_i is normalized device coordinate [0,1], 0=max depth
                        z_i = calculate_normalized_depth(p, r, camera_params)
                        
                        # Early stopping check
                        # If current sphere depth is greater than bound derived from current_weight_sum, stop
                        if z_i > calculate_early_stop_bound(current_weight_sum, gamma, xi):
                            break
                            
                        # Calculate normalized orthogonal distance d_i
                        # Vector from ray to sphere center
                        ray_origin = camera_params['origin']
                        ray_direction = get_ray_direction(x, y, camera_params['K'])
                        d_tilde = p - (ray_origin + ray_direction * z_i) # Simplified projection logic
                        # Orthogonal component
                        d_orth = d_tilde - torch.dot(d_tilde, ray_direction) * ray_direction
                        d_i = min(1.0, torch.norm(d_orth) / r)
                        
                        # Calculate blending weight w_i
                        # Numerator
                        num = o * d_i * math.exp(o * (z_i / gamma))
                        # Denominator (sum over all k, but we approximate with current sum + current term for streaming)
                        # In practice, denominator requires sum over all contributing spheres.
                        # For exact implementation, a two-pass or atomic sum is needed.
                        # Here we assume a global sum or accumulation.
                        denom = math.exp(xi / gamma) + current_weight_sum + num
                        
                        w_i = num / denom
                        
                        # Aggregate feature
                        aggregated_feature += w_i * f
                        current_weight_sum += num # Approximation for denominator accumulation
                        
                        # Gradient propagation limit: only use first few spheres for gradients
                        # [G] Implementation detail: Flag spheres for gradient computation vs color contribution
                        if idx < 5: # Example limit
                            aggregated_feature_grad = w_i * f
                        else:
                            aggregated_feature_grad = torch.zeros_like(f)
                            
                    feature_image[y, x] = aggregated_feature
                    
    # Step 5: Neural Shading
    # Convert feature map to color
    color_image = shading_net(feature_image)
    
    return color_image

def compute_enclosing_rectangles(spheres, K):
    """
    Computes bounding boxes for spheres on the sensor plane.
    Returns intersection_info (M, 4 shorts), draw_info (M, struct), depths (M, float)
    """
    # [G] Implementation of pinhole projection bounding box calculation
    # Uses K, sphere position p, and radius r
    # Robustify: ensure min size of 1 pixel
    pass

def calculate_normalized_depth(p, r, camera_params):
    """
    Calculates normalized device coordinate depth z_i.
    0 = max depth, 1 = min depth (closest).
    """
    # [G] Project sphere center to camera space, add/subtract radius for depth range
    # Return normalized value
    pass

def check_sphere_overlap(intersection_info, y_start, y_end, x_start, x_end):
    """
    Checks if sphere's bounding box overlaps with the image part.
    """
    # Compare intersection_info min/max x/y with part boundaries
    pass

def calculate_early_stop_bound(current_weight_sum, gamma, xi):
    """
    Calculates upper bound for sphere depth to have notable impact.
    """
    # [G] Derived from Eq. 1 and current aggregated weights
    pass
```

**Edge Cases and Pitfalls:**
*   **High-frequency artifacts:** If only the first few points per pixel determine color, complex scenes suffer from artifacts. Solved by allowing unlimited spheres to contribute to color while restricting gradient propagation to the first few spheres [S1].
*   **Opacity integration:** Integrating opacity only as a depth scaling factor leaves spheres visible in front of the background; integrating only as a distance scaling factor makes depth override opacity. Solved by using opacity in both places (exponential and linear factor) [S1].
*   **Position gradients instability:** Position gradients are numerically unstable if a sphere projects to a radius of 3 pixels or fewer. Solved by providing position gradients only for spheres with a projected radius > 3 pixels [S1].
*   **Lost spheres:** To prevent spheres from being 'lost' between pixel rays, every sphere is rendered with at least a size of one pixel [S1].

## Subtopics

### [Neural Shading in Pulsar](9424-neural-shading-in-pulsar.md)
Neural Shading in Pulsar is a method for modeling view-dependent appearance by integrating a neural shading network that conditions on per-pixel view direction, shared across all pixels. It addresses the limitation of standard sphere-based rendering where color is often treated as a static attribute, failing to capture complex specularities or material variations that change with the viewing angle. Unlike standard splatting which assigns a fixed color to each primitive, this method decouples geometry projection from appearance, allowing the network to predict the final color based on the underlying sphere features and the camera's viewing vector.

### Sphere Subdivision Refinement
Sphere Subdivision Refinement is a technique to increase sphere count and detail by subdividing existing spheres into smaller ones, followed by re-optimization and pruning [S1]. It refines each sphere into 12 new spheres with radius $\sqrt{2} \cdot r$, placed in a face-centered cubic packing scheme relative to the parent sphere [S1]. The new set of spheres is then optimized, and spheres are pruned if their color converges towards the background color or if they are not visible (do not receive gradient updates from any viewpoint) [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time for up to one million spheres at 1024x1024 resolution is less than 22 ms; gradient calculation time is less than 6 ms [S1]. For 4.4M spheres at 4K resolution, runtime is 387 ms (forward/backward) on an NVIDIA RTX 2070 GPU [S1].
*   **Memory Complexity:** Moderate memory requirements; 3500 MB for 4.4M spheres at 4K resolution [S1].
*   **Scaling:** Performance scales favorably with future GPU generations due to the block size limit (256 threads) being determined by current hardware memory constraints [S1]. Rendering speed is largely independent of sphere sizes due to pixel-parallelization [S1].
*   **Strengths:** Orders of magnitude faster than competing techniques (e.g., PyTorch3D sphere renderer reaches 400 ms below 500K spheres, while Pulsar remains below 400 ms at 4.4M spheres) [S1]. Avoids topological inconsistencies and changing topology problems of meshes [S1]. Eliminates the need for acceleration structures (k-d tree, octree) [S1].
*   **Limitations:** Requires CUDA C and PyTorch integration [S1]. Numerical instability for small projected sphere radii (< 3 pixels) [S1].
*   **When to use:** When real-time rendering and optimization of millions of spheres are required, or when topological consistency is critical [S1].
*   **When not to use:** When hardware does not support CUDA or PyTorch, or when very small-scale features (sub-pixel spheres) are critical and cannot be robustified [S1].

## Background
Pulsar was introduced in 2021 by the authors of "Pulsar: Efficient Sphere-based Neural Rendering" [S1]. It builds on recent insights in differentiable rendering and neural networks, deliberately limiting complexity in the projection step to obtain high speed and scalability [S1]. It is implemented as a C++ and CUDA C PyTorch extension [S1]. The method was developed to alleviate challenges in gradient-based optimization of neural mesh, voxel, or function representations, such as topological inconsistencies, high memory footprints, and slow rendering speeds [S1].

## Sources
[S1] Pulsar: Efficient Sphere-based Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00149, fetched 2026-10-26
