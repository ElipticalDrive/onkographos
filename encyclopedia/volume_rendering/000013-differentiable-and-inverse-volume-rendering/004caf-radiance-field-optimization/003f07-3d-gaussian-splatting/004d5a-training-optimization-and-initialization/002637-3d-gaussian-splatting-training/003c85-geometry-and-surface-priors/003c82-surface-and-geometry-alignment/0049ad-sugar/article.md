# SuGaR

Also called: SuGaR: Surface-Aligned Gaussian Splatting for Realistic Head Avatar Reconstruction, Surface-aligned Gaussian Splatting

## Overview
SuGaR is a post-hoc method that extracts watertight 3D surfaces from a trained 3D Gaussian Splatting (3DGS) model by introducing a self-regularization loss to align Gaussian primitives with the underlying scene geometry [S1]. It distinguishes itself from implicit surface methods by enforcing alignment through a penalty term that minimizes the distance between the camera and the closest Gaussian relative to rendered depth, followed by Poisson mesh reconstruction [S1][S3, unofficial]. While it enables mesh extraction from 3DGS, it is noted that current discrete geometry methods based on 3DGS may achieve comparable or worse results compared to continuous implicit representations [S1].

## How it works
SuGaR operates in two phases: surface-aligned Gaussian optimization and mesh extraction via Poisson reconstruction [S3, unofficial].

### Inputs and Outputs
- **Input**: A pre-trained 3DGS scene representation and multi-view images with known camera poses [S1].
- **Output**: A triangular mesh representing the scene surface [S2].

### Data Structures
The method uses the standard 3DGS representation where each Gaussian is defined by center position, rotation, scale, opacity, and spherical harmonics coefficients for view-dependent color [G].

### Method Steps
1. **Render Depth and Opacity**: For each training view, render the depth map and opacity map using standard 3DGS rasterization [G]. The depth for a pixel is the weighted average of the distances of the splatted Gaussians along the ray [G].
2. **Self-Regularization Loss**: Introduce a loss term $\mathcal{L}_{align}$ to constrain the distance between the camera and the closest Gaussian to match the pixel's depth value in the rendered depth map [S1]. Let $d_{ray}$ be the rendered depth at a pixel and $d_{closest}$ be the distance from the camera center to the center of the closest Gaussian intersecting the ray. The loss is defined as:
   $$ \mathcal{L}_{align} = \sum_{v} \sum_{p \in \text{pixels}_v} | d_{ray}(p) - d_{closest}(p) | $$
   This encourages the Gaussians to align with the authentic 3D surface [S1].
3. **Flatness and Alignment Penalty**: Apply a penalty term to make Gaussians flat and aligned with the surface normal [S3, unofficial]. This regularizes the scale vector to be small in the direction perpendicular to the surface normal, effectively flattening the ellipsoids into surfel-like primitives [S3, unofficial].
4. **Optimization**: Optimize the Gaussian parameters (positions, scales, rotations) by minimizing the sum of the standard 3DGS photometric loss and the new alignment/flatness losses [G].
5. **Poisson Mesh Reconstruction**: Extract points and normals from the optimized Gaussians. Points are taken from the Gaussian centers and normals from the rotation-derived surface normals of the flattened Gaussians. Run Poisson surface reconstruction on these samples to generate a watertight mesh [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sugar_pipeline(gaussians, images, cameras, num_epochs, lr, lambda_align, lambda_flat):
    """
    Execute SuGaR surface alignment and mesh extraction.
    Args:
        gaussians: List of Gaussian objects with attributes: center, rotation, scale, opacity, sh
        images: List of target RGB images
        cameras: List of camera pose and intrinsic matrices
        num_epochs: Number of optimization iterations
        lr: Learning rate
        lambda_align: Weight for alignment loss
        lambda_flat: Weight for flatness penalty
    Returns:
        mesh: Triangular mesh from Poisson reconstruction
    """
    # Phase 1: Surface-Aligned Optimization
    for epoch in range(num_epochs):
        for image, camera in zip(images, cameras):
            # 1. Render depth and opacity using standard 3DGS alpha compositing [G]
            rendered_depth, rendered_opacity = render_depth_opacity(gaussians, camera)
            rendered_color = render_color(gaussians, camera)

            # Photometric loss [G]
            photometric_loss = mean_absolute_error(rendered_color, image)

            # 2. Alignment loss: constrain camera-to-closest-Gaussian distance vs rendered depth [S1]
            closest_dist = compute_closest_gaussian_distance(gaussians, camera)
            alignment_loss = mean_absolute_error(rendered_depth, closest_dist)

            # 3. Flatness/alignment penalty [S3, unofficial]
            flatness_loss = compute_flatness_penalty(gaussians)

            # Combine and optimize [G]
            loss = photometric_loss + lambda_align * alignment_loss + lambda_flat * flatness_loss
            apply_gradients(gaussians, loss, lr)

    # Phase 2: Mesh Extraction
    points = [g.center for g in gaussians]
    normals = [g.normal for g in gaussians]
    # Standard Poisson surface reconstruction algorithm [G]
    mesh = poisson_reconstruction(points, normals)
    return mesh

def render_depth_opacity(gaussians, camera):
    # Standard 3DGS rasterization: sort Gaussians by depth, alpha-composite along rays [G]
    depth_map = np.zeros(camera.image_shape)
    opacity_map = np.zeros(camera.image_shape)
    for g in gaussians:
        # Project Gaussian to 2D, compute contribution per pixel [G]
        pass
    return depth_map, opacity_map

def compute_closest_gaussian_distance(gaussians, camera):
    # For each pixel ray, find distance to nearest Gaussian center [G]
    dist_map = np.full(camera.image_shape, np.inf)
    for g in gaussians:
        # Project Gaussian center to camera space, compute ray distance [G]
        pass
    return dist_map

def compute_flatness_penalty(gaussians):
    # Penalty to make Gaussians flat and aligned with surface normal [S3, unofficial]
    penalty = 0.0
    for g in gaussians:
        # Flatten scale in direction perpendicular to normal
        normal = g.rotation_to_normal()
        scale = g.scale
        penalty += np.sum(np.square(scale) * (1.0 - np.dot(normal, normal)))
    return penalty

def apply_gradients(gaussians, loss, lr):
    # Standard optimizer step (e.g., Adam) [G]
    pass

def poisson_reconstruction(points, normals):
    # Standard Poisson reconstruction [G]
    pass
```

### Edge Cases and Pitfalls
- **Precision**: The alignment loss relies on accurate depth rendering. Discretization artifacts in depth maps may affect convergence [S1].
- **Material Limitations**: The method remains a surface-based representation with limited capacity in reconstructing and rendering fuzzy materials and volumetric effects, resulting in a decrease in performance compared to vanilla Gaussian Splatting [S2].

## Operating characteristics
- **Time Complexity**: Not given in the sources. Optimization is iterative, similar to 3DGS, but with additional alignment loss computation [G].
- **Memory Complexity**: Not given in the sources. Standard 3DGS storage scales linearly with the number of Gaussians [G].
- **Performance**: SuGaR produces watertight meshes suitable for downstream applications like physics simulation or DCC pipelines [S2]. However, it may not achieve the same geometric accuracy as implicit methods, especially in complex scenes [S1].
- **Strengths**: Post-hoc applicability to any 3DGS model; produces clean meshes via Poisson reconstruction [S3, unofficial].
- **Limitations**: Geometry is an emergent property of 3DGS, not a constraint; current discrete geometry methods based on 3DGS may achieve comparable or worse results compared to implicit representations [S1].

## Background
SuGaR was introduced as a pioneering work to build 3D surfaces from multi-view images using the 3DGS representation [S1]. It addresses the limitation of 3DGS's discrete nature, which makes surface extraction difficult compared to continuous implicit representations [S1]. The method uses a self-regularization loss to align Gaussians with the surface, followed by Poisson meshing [S1][S3, unofficial]. It is particularly useful for head avatar reconstruction and other applications requiring explicit mesh geometry [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
