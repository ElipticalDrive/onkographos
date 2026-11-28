# INGeo

Also called: Instant-NGP with Geometry Priors

## Overview
INGeo is a method that accelerates 3D scene reconstruction training by leveraging noisy geometry priors (e.g., from RGB-D sensors or COLMAP) converted into an occupancy grid, aiming for instant reconstruction on edge devices [S1]. It builds on [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by initializing the occupancy grid with prior geometry and applying density scaling to ensure rays terminate quickly at the surface, addressing the slow convergence of randomly initialized weights [S1].

## How it works
**Inputs:**
1. Noisy geometry priors: Point clouds or depth images [S1].
2. Base Instant-NGP model parameters: Hash grid and MLP weights [S1].

**Outputs:**
1. Trained neural radiance field (density and color) [S1].
2. Optimized occupancy grid [S1].

**Data Structures:**
*   **Occupancy Grid:** A 3D tensor where each cell indicates the presence of geometry [S1]. The grid is initialized from geometry priors and updated continuously during training [S1].
*   **Model Weights:** The MLP weights for density and color estimation [S1]. When inheriting a pretrained occupancy grid, approximately 89% of these weights remain randomly initialized [S1].

**Steps:**

1.  **Convert Geometry Priors to Occupancy Grid:**
    *   If the input is a point cloud, perform **point-cloud splatting** to convert points to grid cells [S1]. Points are not treated as infinitesimal; instead, each point is splatted to all nearby grid cells within a specific radius to complete small missing regions caused by sparse priors, at the cost of adding some empty grid cells [S1].
    *   If the input is depth images, convert them to a 3D occupancy representation [S1].
    *   This initial grid serves as the starting state for the reconstruction model [S1].

2.  **Initialize Model with Pretrained Occupancy Grid (Optional Bootstrap):**
    *   To further accelerate, one can inherit an occupancy grid from a pretrained model [S1]. Even if 89% of the model weights are randomly initialized, inheriting the grid provides a significant speedup [S1].
    *   Obtaining a high-quality occupancy grid from scratch via boot-strapping (iteratively updating grid and model) takes significant time (e.g., ~30 minutes on a TX2 device for 88% IoU on the Lego scene) [S1]. INGeo aims to bypass this by using priors [S1].

3.  **Apply Density Scaling (Inductive Bias):**
    *   During training, multiply the initial density prediction $\sigma$ by a scaling factor $k > 1$ [S1].
    *   This ensures that rays terminate soon after encountering the object surface identified by the occupancy grid [S1]. Without this, randomly initialized low density predictions cause accumulated transmittance to drop slowly, broadcasting color gradients to all sampled points (including occluded ones) and slowing training [S1]. The scaled density makes the rendered image appear more “solid” early in training [S1].

4.  **Continuous Occupancy Grid Update:**
    *   Geometry priors are inherently sparse and noisy [S1]. To address sparsity and errors, do not rely solely on the initial grid [S1].
    *   Re-evaluate volume density at a certain frequency during training and update the occupancy grid dynamically [S1].
    *   Combine the initialized occupancy grid with these continuously updated grids to refine the representation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ingeo_train(geometry_priors, instant_ngp_model, k, update_freq, splat_radius, target_iterations):
    # Step 1: Convert priors to initial occupancy grid
    initial_occupancy_grid = {}
    if isinstance(geometry_priors, list):  # Point cloud input
        for point in geometry_priors:
            # Splat point to nearby cells within radius
            for dx in range(-splat_radius, splat_radius + 1):
                for dy in range(-splat_radius, splat_radius + 1):
                    for dz in range(-splat_radius, splat_radius + 1):
                        if dx*dx + dy*dy + dz*dz <= splat_radius*splat_radius:
                            cell = (point[0] + dx, point[1] + dy, point[2] + dz)
                            initial_occupancy_grid[cell] = 1.0
    else:  # Depth images input
        # Standard back-projection: map each depth pixel to 3D coordinates, mark cells as occupied
        for depth_img in geometry_priors:
            for u, v, d in depth_img.to_points():
                cell = _world_to_grid_cell((u, v, d))  # Standard coordinate transformation
                initial_occupancy_grid[cell] = 1.0

    instant_ngp_model.occupancy_grid = initial_occupancy_grid
    # Model weights (MLP) are typically randomly initialized, except if inheriting from a pretrained model

    for iteration in range(target_iterations):
        # Step 2: Sample rays and compute density with scaling
        rays = _sample_rays()  # Standard stratified ray sampling through camera frustum
        densities = []
        for ray in rays:
            base_density = instant_ngp_model.density_mlp(ray.sample_points)
            # Apply density scaling factor k
            scaled_density = base_density * k
            densities.append(scaled_density)

        # Perform volume rendering with scaled densities
        colors = _volume_render(rays, densities, instant_ngp_model.color_mlp)
        # Standard volume rendering: T=1.0, color=0; for each point: alpha=1-exp(-sigma*dt), color+=alpha*T*color, T*=(1-alpha)

        # Compute loss and backpropagate
        target_images = _get_target_images(iteration)
        loss = _compute_loss(colors, target_images)
        instant_ngp_model.update_weights(loss)

        # Step 3: Update occupancy grid periodically
        if iteration % update_freq == 0:
            # Re-evaluate volume density to update grid
            updated_grid = {}
            for cell in instant_ngp_model.occupancy_grid.keys():
                density = instant_ngp_model.density_mlp(_grid_cell_to_world(cell))
                if density > 0.0:
                    updated_grid[cell] = 1.0
            # Combine initial grid with updated grid
            instant_ngp_model.occupancy_grid = {**initial_occupancy_grid, **updated_grid}

    return instant_ngp_model
```

**Parameters:**
*   **Density Scaling Factor ($k$):** A value $> 1$ applied to initial density predictions [S1]. Exact value not given in the sources.
*   **Splatting Radius:** The radius within which point cloud points are splatted to grid cells [S1]. Exact value not given in the sources.
*   **Update Frequency:** How often the occupancy grid is re-evaluated and updated [S1]. Exact value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Low Initial Density:** Without density scaling, randomly initialized weights produce low density, causing slow transmittance drop and gradient broadcast to occluded points, slowing training [S1].
*   **Sparse Priors:** Point clouds may have missing regions, requiring splatting which adds empty cells [S1].
*   **Noisy Priors:** Geometry priors from RGB-D or COLMAP are noisy, necessitating continuous grid updates [S1].

## Operating characteristics
*   **Speedup:** INGeo achieves approximately 2x speedup over standard Instant-NGP on the NeRF Synthetic dataset, reaching an average test PSNR of >30 in half the training iterations [S1].
*   **Hardware:** Tested on edge devices like NVIDIA TX2. Obtaining an occupancy grid with 88% IoU on the Lego scene takes about 30 minutes using the boot-strapping strategy without priors [S1].
*   **Memory:** Inherits the memory footprint of Instant-NGP (hash grid + MLP). The occupancy grid is initialized from priors, potentially reducing the need for extensive initial sampling [S1].
*   **Strengths:** Enables instant reconstruction on edge devices (mobile phones, AR/VR headsets) by leveraging existing geometry priors [S1]. Reduces spatial redundancy and mitigates hash collisions [S1].
*   **Limitations:** Relies on the availability of noisy geometry priors. Performance depends on the quality of priors; sparse or noisy priors require careful handling (splatting, updates) [S1].

## Background
INGeo was proposed by Jian Li, Bichen Wu, Albert Pumarola, Peizhao Zhang, Yingyan Lin, and Peter Vajda from Georgia Institute of Technology and Meta Reality Labs [S1]. The method aims to bridge the gap between high-end GPU reconstruction and instant training on edge devices for AR/VR applications [S1]. It builds on Instant-NGP and addresses limitations of Depth-Supervised NeRF by using occupancy grids rather than depth as training signals [S1].

## Sources
[S1] INGeo: Accelerating Instant Neural Scene Reconstruction with Noisy Geometry Priors, https://doi.org/10.1007/978-3-031-25066-8_40, fetched 2026-10-26
