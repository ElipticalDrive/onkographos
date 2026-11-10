# SuGaR

Also called: SuGaR: Surface-Aligned Gaussian Splatting for Realistic Head Avatar Reconstruction, Surface-aligned Gaussian Splatting

## Overview
SuGaR is a post-hoc method that extracts watertight 3D surfaces from a trained 3D Gaussian Splatting (3DGS) model by introducing a self-regularization loss to align Gaussian primitives with the underlying scene geometry [S1]. It distinguishes itself from implicit surface methods (e.g., NeuS) and other discrete approaches by enforcing alignment through a penalty term that minimizes the distance between the camera and the closest Gaussian relative to rendered depth, followed by Poisson mesh reconstruction [S1][S2, unofficial]. While it enables mesh extraction from 3DGS, it is noted that current discrete geometry methods based on 3DGS may achieve comparable or worse results compared to continuous implicit representations [S1].

## How it works
SuGaR operates in two phases: (1) surface-aligned Gaussian optimization, and (2) mesh extraction via Poisson reconstruction.

### Inputs and Outputs
- **Input**: A pre-trained 3DGS scene representation (set of 3D Gaussians $\mathcal{G}$) and multi-view images with known camera poses.
- **Output**: A triangular mesh representing the scene surface.

### Data Structures
The 3DGS representation consists of a set of $N$ Gaussians. Each Gaussian $i$ is defined by:
- Center position $\mathbf{t}_i \in \mathbb{R}^3$
- Rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ (or rotation matrix $\mathbf{R}_i$)
- Scale vector $\mathbf{s}_i \in \mathbb{R}^3$
- Opacity $\alpha_i \in [0, 1]$
- Spherical harmonics coefficients for view-dependent color $\mathbf{c}_i$

### Method Steps

1. **Render Depth and Opacity**:
   For each training view $v$, render the depth map $D_v$ and opacity map $\alpha_v$ using the standard 3DGS rasterizer. The depth $d_v$ for a pixel is the weighted average of the distances of the splatted Gaussians along the ray.

2. **Self-Regularization Loss**:
   Introduce a loss term $\mathcal{L}_{align}$ to constrain the distance between the camera and the closest Gaussian to match the rendered depth.
   Let $d_{ray}$ be the rendered depth at a pixel.
   Let $d_{closest}$ be the distance from the camera center to the center of the closest Gaussian intersecting the ray.
   The loss is defined as:
   $$ \mathcal{L}_{align} = \sum_{v} \sum_{p \in \text{pixels}_v} | d_{ray}(p) - d_{closest}(p) | $$
   This encourages the Gaussians to align with the authentic 3D surface [S1].

3. **Flatness and Alignment Penalty**:
   Apply a penalty term to make Gaussians flat and aligned with the surface normal. This involves regularizing the scale vector $\mathbf{s}_i$ to be small in the direction perpendicular to the surface normal, effectively flattening the ellipsoids into surfel-like primitives [S2, unofficial].

4. **Optimization**:
   Optimize the Gaussian parameters (positions, scales, rotations) by minimizing the sum of the standard 3DGS photometric loss and the new alignment/flatness losses.

5. **Poisson Mesh Reconstruction**:
   After optimization, extract points and normals from the aligned Gaussians.
   - **Points**: Use the Gaussian centers $\mathbf{t}_i$.
   - **Normals**: Use the rotation-derived normals of the flattened Gaussians.
   Run Poisson surface reconstruction on these points and normals to generate a watertight mesh [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sugar_pipeline(gaussians, images, cameras):
    """
    Execute SuGaR surface alignment and mesh extraction.
    
    Args:
        gaussians: Initial 3DGS parameters (t, q, s, alpha, SH)
        images: List of training images
        cameras: List of camera poses and intrinsics
        
    Returns:
        mesh: Triangular mesh from Poisson reconstruction
    """
    # Phase 1: Surface-Aligned Optimization
    # Initialize Gaussians with pre-trained 3DGS weights
    current_gaussians = gaussians
    
    for epoch in range(num_epochs):
        total_loss = 0
        for image, camera in zip(images, cameras):
            # Render depth and opacity
            rendered_depth, rendered_opacity, depth_weights = render_depth(current_gaussians, camera)
            
            # Compute photometric loss (standard 3DGS)
            rendered_color = render_color(current_gaussians, camera)
            photometric_loss = l1_loss(rendered_color, image)
            
            # Compute alignment loss
            # Find closest Gaussian center distance for each pixel
            closest_dist = get_closest_gaussian_distance(current_gaussians, camera)
            
            # Align rendered depth with closest Gaussian distance
            # Note: rendered_depth is typically a weighted average; 
            # SuGaR constrains the closest Gaussian to match the depth value
            alignment_loss = l1_loss(rendered_depth, closest_dist)
            
            # Compute flatness/alignment penalty
            # Encourage scales to be small in normal direction
            flatness_loss = compute_flatness_penalty(current_gaussians)
            
            # Combine losses
            loss = photometric_loss + lambda_align * alignment_loss + lambda_flat * flatness_loss
            total_loss += loss
            
        # Backpropagate and update Gaussian parameters
        update_gaussians(current_gaussians, total_loss)
        
    # Phase 2: Mesh Extraction
    # Extract points and normals from optimized Gaussians
    points = [g.center for g in current_gaussians]
    normals = [g.normal for g in current_gaussians]
    
    # Run Poisson reconstruction
    mesh = poisson_reconstruction(points, normals)
    
    return mesh
```

### Edge Cases and Pitfalls
- **Precision**: The alignment loss relies on accurate depth rendering. Discretization artifacts in depth maps may affect convergence [S1].
- **Degenerate Input**: If the scene has large transparent regions or low texture, the closest Gaussian may not align well with the true surface, leading to noisy meshes [S1].
- **Boundary**: The method assumes the Gaussians are already reasonably close to the surface; poor initial 3DGS reconstruction may require additional regularization [S1].

## Operating characteristics
- **Time Complexity**: Depends on the number of Gaussians $N$ and training views $V$. Optimization is iterative, similar to 3DGS, but with additional alignment loss computation.
- **Memory Complexity**: $O(N)$ for storing Gaussian parameters.
- **Performance**: SuGaR produces watertight meshes suitable for downstream applications like physics simulation or DCC pipelines [S2, unofficial]. However, it may not achieve the same geometric accuracy as implicit methods like NeuS, especially in complex scenes [S1].
- **Strengths**: Post-hoc applicability to any 3DGS model; produces clean meshes via Poisson reconstruction.
- **Limitations**: Geometry is an emergent property of 3DGS, not a constraint; may suffer from floating artifacts if alignment is insufficient [S2, unofficial].

## Background
SuGaR was introduced as a pioneering work to build 3D surfaces from multi-view images using the 3DGS representation [S1]. It addresses the limitation of 3DGS's discrete nature, which makes surface extraction difficult compared to continuous implicit representations [S1]. The method uses a self-regularization loss to align Gaussians with the surface, followed by Poisson meshing [S1][S2, unofficial]. It is particularly useful for head avatar reconstruction and other applications requiring explicit mesh geometry [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
