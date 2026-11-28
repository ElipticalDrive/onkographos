# CADSplat

Also called: Sparse-View 3D Gaussian Splatting Aided by CAD Models

## Overview
CADSplat reconstructs photorealistic, geometrically accurate digital twins from sparse (< 15 views), wide-baseline posed images by regularizing 3D Gaussian Splatting (3DGS) with an explicit Computer-Aided Design (CAD) shape prior [S1, unofficial]. Unlike methods that rely on learned generative priors or large-scale data priors (e.g., InstantSplat, pixelSplat), CADSplat imposes a strict, explicit geometric constraint derived from the object's specific CAD model, providing geometric disambiguation under severe occlusion and view scarcity [S1, unofficial]. It differs from sibling geometry-constrained methods by anchoring Gaussians to a parametric CAD surface and jointly optimizing a non-rigid deformation field, rather than relying solely on surface normal or SDF constraints [S1, unofficial].

## How it works
**Inputs:**
1.  A set of $N$ input images ($N < 15$), assumed to be wide-baseline and posed (relative geometry approximately known).
2.  A library of CAD models (meshes).
3.  Segmented object silhouettes for each input image [S1, unofficial].

**Outputs:**
1.  A set of 3D Gaussians $\{G_i\}$ anchored to a deformed CAD surface.
2.  Optimized camera-to-object poses.
3.  A non-rigid deformation field mapping the canonical CAD shape to the physical object [S1, unofficial].

**Data Structures:**
*   **CAD Mesh:** A triangular mesh $\mathcal{M}$ with vertices $\mathbf{V}$ and faces $\mathcal{F}$.
*   **Gaussian Primitives:** A fixed set of Gaussians $\{G_i\}_{i=1}^M$ initialized on the surface of $\mathcal{M}$. Each $G_i$ retains standard 3DGS parameters: mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$ (rotation $\mathbf{R}_i$, scale $\mathbf{S}_i$), opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$ for color [S1, unofficial].
*   **Deformation Network:** A neural network $D_\theta$ that takes a point on the CAD surface and returns a displacement vector [S1, unofficial].
*   **Pose Parameters:** A global Sim(3) transformation (scale, rotation, translation) or per-camera corrections [S1, unofficial].

**Algorithm Steps:**

1.  **CAD Retrieval and Silhouette Registration:**
    *   Render silhouettes from all CAD models in the library from various viewpoints.
    *   Match the segmented silhouettes of the input images against the rendered CAD silhouettes.
    *   Select the best-matching CAD model and its associated camera-to-object poses as the initial registration [S1, unofficial].
    *   *Note:* This step provides only coarse accuracy (median rotation error ~6.2°, position error ~18.6% on SCO-CAD) [S1, unofficial].

2.  **Initialization:**
    *   Position the selected CAD mesh at the intersection of the cameras' viewing directions.
    *   Rescale the mesh to cover the same surface area as the ground-truth silhouettes [S1, unofficial].
    *   Anchor the initial set of 3D Gaussians to the surface of this CAD mesh [S1, unofficial].

3.  **Geometric Warmup (Mask Alignment):**
    *   Optimize the camera-to-object pose and the deformation field $D_\theta$ using only silhouette (mask) alignment loss.
    *   *Purpose:* Prevents the appearance optimization from fitting texture to a misregistered surface, which causes reconstruction collapse [S1, unofficial].

4.  **Joint Optimization:**
    *   Optimize three components simultaneously:
        1.  **3DGS Parameters:** $\boldsymbol{\mu}_i, \mathbf{R}_i, \mathbf{S}_i, \alpha_i, \mathbf{c}_i$.
        2.  **Camera-to-Object Registration:** Refine the initial poses using a global Sim(3) transformation (or per-camera corrections for datasets with inaccurate calibration like MVMC) [S1, unofficial].
        3.  **Deformation Field $D_\theta$:** A non-rigid field that "wraps" the CAD mesh onto the physical observation, accounting for machining tolerances, wear, or design differences [S1, unofficial].

5.  **Regularization:**
    *   **SH Neighbor Smoothness:** Pull each splat's spherical harmonic coefficients toward those of its 20 canonical-neighbor splats to remove high-frequency color changes in unobserved views [S1, unofficial].
    *   **Splat Dropout:** Randomly drop splats during training to ensure that splats occluded in training views but visible in held-out views still receive an optimization signal [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CADSplat_Training(InputImages, Silhouettes, CADLibrary):
    # 1. Retrieval
    best_cad_model, initial_poses = Match_Silhouettes(InputImages, Silhouettes, CADLibrary)
    
    # 2. Initialization
    cad_mesh = best_cad_model
    cad_mesh = Rescale_To_Mask_Area(cad_mesh, Silhouettes)
    cad_mesh = Position_At_Center(cad_mesh, InputImages)
    
    # Anchor Gaussians to mesh surface
    gaussians = Anchor_Gaussians_To_Mesh(cad_mesh)
    
    # Initialize deformation network and pose parameters
    deformation_net = Initialize_Network()
    pose_params = Initialize_Pose(initial_poses)
    
    # 3. Geometric Warmup
    # Optimize pose and deformation using silhouette loss only
    for step in Warmup_Steps:
        deformed_mesh = Deform_Mesh(cad_mesh, deformation_net)
        rendered_silhouettes = Render_Silhouettes(deformed_mesh, pose_params)
        loss_mask = Compute_Silhouette_Loss(rendered_silhouettes, Silhouettes)
        Update(deformation_net, pose_params, loss_mask)
        
    # 4. Joint Optimization
    for step in Total_Steps:
        # Apply deformation to get physical surface
        deformed_mesh = Deform_Mesh(cad_mesh, deformation_net)
        
        # Sample Gaussians from deformed surface
        # Note: Gaussians are anchored to the surface, so their means move with deformation
        current_gaussians = Update_Gaussian_Positions(gaussians, deformed_mesh)
        
        # Render image
        rendered_image = Render_3DGS(current_gaussians, pose_params)
        
        # Compute photometric loss
        loss_photo = Compute_Photo_Loss(rendered_image, InputImages)
        
        # Regularization: SH Neighbor Smoothness
        # Find 20 canonical neighbors for each splat
        neighbors = Find_Canonical_Neighbors(current_gaussians, k=20)
        loss_sh_smooth = Compute_SH_Smoothness_Loss(current_gaussians, neighbors)
        
        # Regularization: Splat Dropout
        # Randomly mask out a subset of splats for this step
        active_gaussians = Apply_Splat_Dropout(current_gaussians)
        
        # Joint Loss
        loss = loss_photo + lambda_sh * loss_sh_smooth
        # Note: Splat dropout is handled by the rendering pass on active_gaussians
        
        # Update all parameters
        Update(gaussians, pose_params, deformation_net, loss)
        
    return gaussians, pose_params, deformation_net

def Deform_Mesh(mesh, net):
    # For each vertex v in mesh, compute displacement d = net(v)
    # New position v' = v + d
    # Return new mesh
    pass

def Anchor_Gaussians_To_Mesh(mesh):
    # Sample points on mesh surface
    # Initialize Gaussian means at these points
    # Initialize covariances based on local surface normal and tangent plane
    pass
```

**Parameters:**
*   **Input View Count:** $< 15$ views. Performance degrades gracefully to as few as 3 views [S1, unofficial].
*   **SH Neighbor Count:** 20 canonical neighbors [S1, unofficial].
*   **Camera Calibration:** Wide-baseline, posed. Not necessarily perfectly pre-calibrated [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **CAD Mismatch:** The matched CAD model rarely matches the physical object perfectly due to tolerances or wear. Without the deformation field, this causes ghosting [S1, unofficial].
*   **Misregistration:** Silhouette-based registration is coarse. Without the geometric warmup phase, texture fitting to a misregistered surface causes reconstruction collapse (PSNR drops to ~18.88) [S1, unofficial].
*   **Pose Accuracy:** For datasets with inaccurate calibration (e.g., MVMC), per-camera pose corrections are necessary in addition to global Sim(3) [S1, unofficial].
*   **View Count Dependency:** CAD initialization is superior to sphere initialization in sparse regimes (3 views). However, with 9-13 views, sphere initialization becomes competitive or slightly superior, as photometric signals are rich enough for the deformation field to carve the object [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:**
    *   On SCO-CAD with 3 views, CAD initialization yields PSNR 20.710 vs 19.312 for sphere, and IoU 0.929 vs 0.879 [S1, unofficial].
    *   On SCO-CAD full budget, sphere initialization yields PSNR 23.535 vs 23.239 for CAD, and IoU 0.946 vs 0.942 [S1, unofficial].
    *   Deformation alone (no pose opt) achieves PSNR 22.92 and IoU 0.941, within 0.3 dB of the full method [S1, unofficial].
    *   Pose optimization alone (no deformation) achieves PSNR 22.67 [S1, unofficial].
    *   Without mask alignment, reconstruction collapses to PSNR 18.88 and IoU 0.784 [S1, unofficial].
    *   SH neighbor smoothness increases PSNR from 22.93 to 23.13 and reduces mLPIPS from 0.039 to 0.032 [S1, unofficial].
    *   On NeRS MVMC dataset, CADSplat achieves PSNR 18.7, SSIM 0.767, LPIPS 0.157, outperforming NeRS (PSNR 16.5) and CADSim (PSNR 17.7) [S1, unofficial].
*   **Scaling:** The method is designed for sparse views. It degrades gracefully to 3 views [S1, unofficial].
*   **Strengths:** Provides geometric disambiguation under occlusion; enables applications like markerless AR registration and part label transfer; robust to sparse inputs.
*   **Limitations:** Requires access to a CAD model library; relies on silhouette segmentation; deformation field may struggle if the CAD model is categorically wrong (e.g., a bottle instead of a fire hydrant), though it can bridge category-level gaps [S1, unofficial].

## Background
CADSplat was introduced in 2026 by researchers from the Digital Future Lab, Flanders Make, and Hasselt University [S1, unofficial]. It addresses the problem of reconstructing digital twins from sparse, wide-baseline images where traditional 3DGS fails due to lack of geometric constraints. It builds on [Scene Representation and Geometry Constraints](15421-scene-representation-and-geometry-constraints.md) by introducing an explicit CAD prior. It differs from generative priors (pixelSplat, MVSplat) and learned priors (InstantSplat) by using an explicit, object-specific CAD model [S1, unofficial].

## Sources
[S1] CADSplat: Sparse-View 3D Gaussian Splatting Aided by CAD Models for Robust, Photorealistic Digital-Twin Reconstruction, https://arxiv.org/abs/2609.18473, fetched 2026-10-30 (unofficial)
