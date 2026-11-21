# Urban Radiance Fields


## Overview
Urban Radiance Fields is a method for applying Neural Radiance Field (NeRF) based view synthesis and 3D reconstruction to urban environments using sparse multi-view images supplemented by LiDAR data [S1]. It addresses the challenges of unbounded outdoor scenes, limited camera pose variety, and large-scale scene requirements by integrating geometric priors from LiDAR into the optimization loop [S1]. Unlike standard NeRF which relies solely on photometric consistency, this method enforces geometric accuracy and semantic consistency through specific depth, sight, and segmentation losses [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse multi-view RGB images, corresponding LiDAR depth measurements, and camera poses [S1].
*   **Outputs:** A trained neural field representing the scene, from which novel views and 3D point clouds/meshes are extracted [S1].

**Data Structures:**
The method uses a standard NeRF architecture mapping 5D coordinates $(x, y, z, \theta, \phi)$ to density $\sigma$ and color $c$ [S1]. Specific architecture or training data details are not given in the sources [S1].

**Optimization Steps:**
1.  **Ray Casting:** For each pixel in the target virtual camera, a ray is cast through the scene [S1].
2.  **Neural Query:** The ray is sampled at discrete points. The neural network queries the density $\sigma(t)$ and color $c(t)$ at each point $t$ along the ray [S1].
3.  **Volume Rendering:** The color $\hat{C}(r)$ for the ray $r$ is computed using the standard volume rendering equation [S1]:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $T_i = \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$ is the accumulated transmittance, $\alpha_i = 1 - e^{-\sigma_i \delta_i}$ is the opacity, and $\delta_i$ is the interval length [S1].
4.  **Depth Optimization:** The estimated depth $\hat{z}$ is treated as an optimization parameter updated via the depth loss to match LiDAR measurements [S1].
5.  **Loss Calculation:** The total loss is the sum of the standard photometric loss and three auxiliary losses:
    *   **LiDAR Depth Loss ($L_{depth}$):** Forces the estimated depth $\hat{z}$ to match the LiDAR-measured depth $z$ [S1].
    *   **Sight Loss ($L_{sight}$):** Forces the radiance weight $w(t)$ to concentrate at the surface defined by the LiDAR depth $z$ [S1].
    *   **Sky Segmentation Loss ($L_{seg}$):** Forces point samples along rays passing through sky pixels to have zero density [S1].
6.  **Backpropagation:** The gradients are backpropagated to update the neural network weights [S1].

**Formulas:**
The auxiliary losses are defined as [S1]:

$$ L_{depth} = \mathbb{E}[(z - \hat{z})^2] $$

$$ L_{sight} = \mathbb{E}\left[ \int_{t_1}^{t_2} (w(t) - \delta(z))^2 dt \right] $$

$$ L_{seg} = \mathbb{E}\left[ \int_{t_1}^{t_2} S_i(r) (w(t) - \delta(z))^2 dt \right] $$

Where:
*   $z$ is the LiDAR measured depth [S1].
*   $\hat{z}$ is the estimated depth derived from the NeRF [S1].
*   $w(t) = T(t)\sigma(t)$ is the radiance weight at point $t$ [S1].
*   $\delta(z)$ is the Dirac delta function, concentrating the weight at the measured depth surface [S1].
*   $S_i(r)$ is a binary mask where $S_i(r) = 1$ if the ray $r$ passes through a sky pixel in the $i$-th image, and $0$ otherwise. Sky pixels are identified using a pretrained segmentation model [S1].
*   $t_1$ and $t_2$ define the integration bounds along the ray [S1].

**3D Reconstruction Pipeline:**
After training, 3D reconstruction is performed as follows [S1]:
1.  A ray is cast for each pixel in a virtual camera.
2.  The estimated depth $\hat{z}$ is used to place points in 3D space, generating a point cloud from the NeRF model during volumetric rendering.
3.  Poisson Surface Reconstruction is applied to the generated point cloud to reconstruct a 3D mesh [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_urban_nerf(images, lidar_depths, poses, sky_masks, network, optimizer, num_epochs=1000):
    for epoch in range(num_epochs):
        for image, lidar_depth, pose, sky_mask in zip(images, lidar_depths, poses, sky_masks):
            # 1. Cast rays for each pixel in the target virtual camera [S1]
            H, W = image.shape[:2]
            positions, directions = generate_rays(H, W, pose)
            positions = positions.reshape(-1, 3)
            directions = directions.reshape(-1, 3)

            # 2. Query network for density sigma and color c [S1]
            sigma, color = network(positions, directions)

            # 3. Compute transmittance T(t) and weights w(t) = T(t) * sigma(t) [S1]
            deltas = torch.norm(torch.diff(positions, dim=0, prepend=positions[0:1]), dim=1, keepdim=True)
            alpha = 1.0 - torch.exp(-sigma * deltas)
            T = torch.cumprod(1.0 - alpha + 1e-10, dim=0)
            T = torch.cat([torch.ones_like(T[0:1]), T[:-1]], dim=0)
            w = T * sigma

            # 4. Compute rendered color and estimated depth z_hat [S1]
            rendered_color = torch.sum(w.unsqueeze(-1) * color, dim=0).reshape(H, W, 3)
            # Estimated depth is optimized to match LiDAR depth via L_depth [S1]
            depths_along_ray = torch.sum(positions, dim=1, keepdim=True) # Simplified ray parameterization
            estimated_depth = torch.sum(w * depths_along_ray, dim=0).reshape(H, W)

            # 5. Compute auxiliary losses [S1]
            loss_depth = torch.mean((lidar_depth - estimated_depth) ** 2)

            # Dirac delta delta(z) is approximated for discrete samples [G]
            delta_approx = torch.exp(-((depths_along_ray - lidar_depth.unsqueeze(1)) ** 2) / (2 * 0.01 ** 2))
            loss_sight = torch.mean((w - delta_approx) ** 2)
            loss_seg = torch.mean(sky_mask * (w - delta_approx) ** 2)

            # Total loss includes standard photometric loss [S1]
            loss_photo = torch.mean((rendered_color - image) ** 2)
            total_loss = loss_photo + loss_depth + loss_sight + loss_seg

            optimizer.zero_grad()
            total_loss.backward()
            optimizer.step()

def extract_mesh(network, image_shape, virtual_pose):
    # 1. Cast ray for each pixel in the virtual camera [S1]
    H, W = image_shape
    positions, directions = generate_rays(H, W, virtual_pose)
    positions = positions.reshape(-1, 3)
    directions = directions.reshape(-1, 3)

    # 2. Query network and compute depth to place point cloud [S1]
    sigma, _ = network(positions, directions)
    estimated_depth = torch.sum(torch.exp(-sigma), dim=0).reshape(H, W)
    points = ray_march_to_depth(positions, directions, estimated_depth)

    # 3. Reconstruct mesh using Poisson Surface Reconstruction [S1]
    mesh = poisson_surface_reconstruction(points)
    return mesh

def generate_rays(H, W, pose):
    # Standard ray generation from camera pose
    # Returns positions (H, W, 3) and directions (H, W, 3)
    pass # Not given in sources [S1]

def ray_march_to_depth(positions, directions, depth_map):
    # Places points in 3D space using estimated depth [S1]
    # Returns point cloud (N, 3)
    pass # Not given in sources [S1]

def poisson_surface_reconstruction(points):
    # Reconstructs a 3D mesh from the generated point cloud [S1]
    pass # Not given in sources [S1]
```

**Parameters:**
*   **Sky Segmentation Model:** A pretrained model is used to identify sky pixels [S1]. Specific architecture or training data is not given in the sources [S1].
*   **Dirac Delta Approximation:** The sources define $\delta(z)$ as the Dirac delta function [S1]. Implementation details for approximating this in discrete sampling are not provided [G].
*   **Poisson Surface Reconstruction:** Used for mesh generation [S1]. Specific parameters (e.g., depth, width) are not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **LiDAR Alignment:** The depth loss requires accurate alignment between LiDAR depth and NeRF estimated depth [S1]. Misalignment may lead to incorrect geometry [S1].
*   **Sky Segmentation Errors:** Incorrect sky segmentation masks may cause the model to suppress density in non-sky regions or fail to suppress density in sky regions [S1].
*   **Unbounded Scenes:** Urban environments are unbounded, which can cause issues with standard NeRF's bounded volume assumption. The use of LiDAR helps mitigate this by providing explicit depth constraints [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Strengths:** Integrates LiDAR data for geometric accuracy; handles sparse views; addresses unbounded scene challenges [S1].
*   **Limitations:** Relies on accurate LiDAR data and sky segmentation; specific performance metrics are not provided in the sources [S1].
*   **When to Use:** Urban environments with sparse multi-view images and available LiDAR data [S1].
*   **When Not to Use:** Indoor scenes or scenes without LiDAR data where the specific losses may not be applicable [S1].

## Background
Urban Radiance Fields was proposed in November 2021 [S1]. It aims to apply NeRF-based view synthesis and 3D reconstruction to urban environments, addressing challenges such as unbounded scenes, limited camera pose variety, and large-scale scene requirements [S1]. The method supplements sparse multi-view images with LiDAR data and introduces specific losses for depth, sight, and sky segmentation [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
