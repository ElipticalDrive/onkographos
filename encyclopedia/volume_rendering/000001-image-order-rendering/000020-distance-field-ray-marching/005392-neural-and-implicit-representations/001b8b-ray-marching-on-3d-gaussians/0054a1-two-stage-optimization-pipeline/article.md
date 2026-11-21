# Two-Stage Optimization Pipeline

Also called: Two-Stage Strategy, Two-Stage Refinement

## Overview
The Two-Stage Optimization Pipeline is a post-processing refinement method for 3D Gaussian Splatting (3DGS) that decouples geometric alignment from material/lighting optimization to improve reconstruction quality and enable relighting. It addresses the limitations of coarse geometry from initial regressors and accumulated pose errors by first correcting camera poses and refining geometry, then locking geometry to optimize material and lighting parameters using pre-computed visibility. It differs from standard 3DGS by introducing explicit normal vectors and a visibility pre-computation step, and from pose-free methods like [CF3DGS](1036-ray-marching-on-3d-gaussians.md) by using a dedicated pose correction stage.

## How it works
**Inputs:** A set of registered images, initial camera poses (with seed image pose fixed), and an initial 3D Gaussian point cloud (optionally with normals).
**Outputs:** Optimized camera poses, refined 3D Gaussian primitives (positions, covariances, opacities, colors/coefficients), and optimized material/lighting parameters.

**Data Structures:**
*   **3D Gaussian Primitive:** Position $\mathbf{p}$, covariance matrix $\Sigma$ (or scale/rotation), opacity $\alpha$, spherical harmonic coefficients for color/view-dependence, and an additional normal vector $\mathbf{n}$ [S2].
*   **Camera Pose:** $6$-DoF transformation matrix.
*   **Visibility Map:** Per-Gaussian visibility $V(\omega_i)$ pre-computed over the hemispherical domain defined by normal $\mathbf{n}$ [S2].

**Steps:**

1.  **Stage 1: Pose Correction and Geometry Refinement**
    1.  **Pose Optimization:** Optimize all camera poses to minimize point-to-camera ray consistency loss (Eq. 8 in [S1, unofficial]). The seed image pose is fixed. Previous training epoch poses serve as initial values [S1, unofficial].
    2.  **Geometry Optimization:** Optimize a vanilla 3D Gaussian point cloud augmented with normal vectors $\mathbf{n}$. Use normal gradient-based densification to improve geometry [S2].
    3.  **Densification and Pruning:** Apply explicit 3D Gaussian primitives densification and pruning strategies from [28] to refine scene details beyond the coarse geometry of the initial scene regressor [S1, unofficial].

2.  **Pre-computation: Visibility**
    1.  For each Gaussian $i$, pre-compute its visibility $V(\omega_i)$ across the hemispherical domain determined by its normal $\mathbf{n}$ using ray tracing [S2].

3.  **Stage 2: Material and Lighting Optimization**
    1.  **Lock Geometry:** Fix the positions, covariances, opacities, and normals of all 3D Gaussians [S2].
    2.  **Optimize Material/Lighting:** Optimize material and lighting parameters using the rendering equation integrated with the pre-computed visibility $V(\omega_i)$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_optimization(images, initial_poses, initial_gaussians):
    # Stage 1: Pose and Geometry
    # Fix seed pose (index 0)
    fixed_pose_idx = 0
    
    # Optimize poses using ray consistency loss
    # initial_poses are used as starting values
    optimized_poses = optimize_poses(
        images=images, 
        poses=initial_poses, 
        loss_fn=ray_consistency_loss, 
        fixed_idx=fixed_pose_idx
    )
    
    # Optimize geometry with normals
    # Augment Gaussians with normal vectors
    gaussians = initial_gaussians
    for gaussian in gaussians:
        gaussian.normal = compute_initial_normal(gaussian)
        
    gaussians = optimize_geometry(
        gaussians=gaussians,
        images=images,
        poses=optimized_poses,
        densification_fn=normal_gradient_densification
    )
    
    # Refine geometry with densification/pruning from [28]
    gaussians = refine_geometry(gaussians, strategy=pruning_densification_from_28)
    
    # Pre-compute visibility
    for gaussian in gaussians:
        gaussian.visibility = precompute_visibility(
            gaussian=gaussian,
            domain=hemispherical_domain(gaussian.normal)
        )
        
    # Stage 2: Material and Lighting
    # Lock geometry
    locked_gaussians = lock_geometry(gaussians)
    
    # Optimize material and lighting
    material_params, lighting_params = optimize_material_lighting(
        gaussians=locked_gaussians,
        images=images,
        poses=optimized_poses,
        visibility=gaussians.visibility,
        loss_fn=rendering_loss_with_visibility
    )
    
    return optimized_poses, gaussians, material_params, lighting_params
```

**Parameters:**
*   **Seed Image Pose:** Fixed during Stage 1 [S1, unofficial].
*   **Densification/Pruning Strategy:** From [28] (Not given in the sources).
*   **Visibility Domain:** Hemispherical domain determined by normal $\mathbf{n}$ [S2].

**Edge Cases:**
*   **Accumulated Pose Errors:** Addressed by Stage 1 pose optimization [S1, unofficial].
*   **Coarse Geometry:** Addressed by Stage 1 densification/pruning [S1, unofficial].
*   **GPU Memory Limitations:** Initial scene regressor uses low-resolution images, necessitating Stage 2 refinement [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Improves rendering quality beyond coarse geometry [S1, unofficial]. Enables realistic relighting [S2].
*   **Strengths:** Corrects accumulated pose errors, refines geometry, enables relighting [S1, unofficial][S2].
*   **Limitations:** Requires two stages, increasing complexity. Initial geometry may be coarse [S1, unofficial].
*   **When to Use:** When high-quality reconstruction and relighting are needed, and initial poses/geometry are available but imperfect [S1, unofficial][S2].
*   **When Not to Use:** When real-time performance is critical and two-stage optimization is too slow.

## Background
*   **Origin:** Proposed in [S1, unofficial] for finalizing neural scene reconstruction and in [S2] for point-based relighting.
*   **History:** Developed to address limitations of incremental registration and coarse geometry in 3DGS [S1, unofficial][S2].
*   **Usage:** Used in 3DGS-based pose-free methods and point-based relighting pipelines [S1, unofficial][S2].

## Sources
[S1] ZeroGS: Training 3D Gaussian Splatting from Unposed Images, https://doi.org/10.48550/arxiv.2411.15779, fetched 2026-10-26 (unofficial)
[S2] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
