# CADSplat

Also called: Sparse-View 3D Gaussian Splatting Aided by CAD Models

## Overview
CADSplat reconstructs photorealistic, geometrically accurate digital twins from sparse (< 15 views), wide-baseline posed images by regularizing 3D Gaussian Splatting (3DGS) with an explicit Computer-Aided Design (CAD) shape prior [S1, unofficial]. It differs from generative priors (e.g., pixelSplat, MVSplat) and learned priors (e.g., InstantSplat) by imposing a strict, explicit prior from the object's specific CAD model rather than a learned generative prior that may hallucinate content [S1, unofficial]. This approach provides geometric disambiguation under occlusion and scarcity, enabling applications such as markerless augmented reality registration, per-image object pose estimation, and the transfer of part labels from design to reconstruction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ posed images ($N < 15$), segmented object silhouettes for each image, and a library of CAD models (or a specific candidate CAD model).
*   **Outputs:** Optimized 3D Gaussian parameters (position, covariance, opacity, spherical harmonic coefficients), camera-to-object poses, and a non-rigid deformation field mapping the CAD mesh to the physical object.

**Data Structures:**
1.  **CAD Mesh:** A triangular mesh representing the canonical object shape.
2.  **Gaussian Primitives:** Anchored to the CAD mesh surface. Each primitive is defined by position $\mathbf{p}$, covariance matrix $\mathbf{\Sigma}$, opacity $\alpha$, and spherical harmonic (SH) coefficients $\mathbf{c}$.
3.  **Deformation Network $D_\theta$:** A neural network that outputs a displacement vector for any point on the CAD mesh.
4.  **Pose Parameters:** A global Sim(3) transformation (scale, rotation, translation) or per-camera corrections.

**Numbered Steps:**

1.  **CAD Model Retrieval and Initial Registration:**
    *   Render silhouettes from candidate CAD models in the library.
    *   Match segmented object silhouettes from input images against rendered CAD silhouettes.
    *   Select the best-matching CAD model and its associated camera-to-object poses [S1, unofficial].
    *   *Note:* This silhouette-based consensus registration is coarsely accurate (median rotation error ~6.2°, position error ~18.6% on SCO-CAD) [S1, unofficial].

2.  **Scene Initialization:**
    *   Position the selected CAD mesh at the intersection of the cameras' viewing directions.
    *   Rescale the mesh to cover the same surface area as the ground-truth masks [S1, unofficial].
    *   Anchor 3D Gaussian primitives to the surface of the retrieved CAD model [S1, unofficial].

3.  **Geometric Warmup (Mask Alignment):**
    *   Optimize pose and deformation against silhouette signals *before* appearance optimization begins.
    *   *Purpose:* Prevents texture fitting to misregistered surfaces. Without this, reconstruction collapses (PSNR 18.88, IoU 0.784) [S1, unofficial].

4.  **Joint Optimization:**
    *   Simultaneously optimize three components:
        1.  **3DGS Parameters:** Standard 3DGS optimization (position, covariance, opacity, SH coefficients).
        2.  **Camera-to-Object Registration:** Refine the initial poses using a global Sim(3) transformation (or per-camera corrections for datasets with inaccurate calibration like MVMC) [S1, unofficial].
        3.  **Non-Rigid Deformation Field $D_\theta$:** A network that "wraps" the CAD mesh onto the physical observation, accounting for shape differences due to machining tolerances, wear, or design variations [S1, unofficial].

5.  **Regularization:**
    *   **SH Neighbor Smoothness:** Pull each splat’s SH coefficients toward those of its 20 canonical-neighbor splats to remove high-frequency color changes in unobserved views [S1, unofficial].
    *   **Splat Dropout:** Ensure splats occluded in training views but visible in held-out views still receive an optimization signal [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cad_splat_training(images, silhouettes, cad_library, num_views):
    # Step 1: Retrieve CAD model and initial poses
    best_cad_model, initial_poses = match_silhouettes(images, silhouettes, cad_library)
    
    # Step 2: Initialize Scene
    # Position mesh at intersection of viewing directions, rescale to match mask area
    cad_mesh = init_cad_mesh(best_cad_model, images)
    gaussians = anchor_gaussians_to_mesh(cad_mesh)
    
    # Initialize deformation network and pose parameters
    deformation_net = init_deformation_network()
    pose_params = init_pose_params(initial_poses)
    
    # Step 3: Geometric Warmup
    # Optimize pose and deformation using only silhouette loss
    for warmup_step in range(WARMUP_ITERATIONS):
        deformed_mesh = apply_deformation(cad_mesh, deformation_net)
        rendered_silhouettes = render_silhouettes(deformed_mesh, pose_params)
        loss_mask = compute_silhouette_loss(rendered_silhouettes, silhouettes)
        optimize(deformation_net, pose_params, loss_mask)
        
    # Step 4: Joint Optimization
    for step in range(TOTAL_ITERATIONS):
        deformed_mesh = apply_deformation(cad_mesh, deformation_net)
        
        # Render Gaussians
        rendered_images, rendered_depths = render_gaussians(gaussians, deformed_mesh, pose_params)
        
        # Photometric Loss
        loss_photo = compute_photometric_loss(rendered_images, images)
        
        # Regularization: SH Neighbor Smoothness
        # For each gaussian, find 20 canonical neighbors and pull SH coefficients
        loss_sh_smooth = compute_sh_smoothness_loss(gaussians, num_neighbors=20)
        
        # Regularization: Splat Dropout
        # Randomly drop splats to ensure visibility in held-out views
        loss_dropout = compute_dropout_loss(gaussians, rendered_images, images)
        
        # Total Loss
        loss = loss_photo + lambda_sh * loss_sh_smooth + lambda_dropout * loss_dropout
        
        # Update all parameters
        optimize(gaussians, deformation_net, pose_params, loss)
        
    return gaussians, pose_params, deformation_net

def apply_deformation(mesh, net):
    # For each vertex v in mesh, compute displacement d = net(v)
    # Return new mesh with vertices v + d
    pass

def anchor_gaussians_to_mesh(mesh):
    # Sample points on mesh surface
    # Initialize Gaussian positions at these points
    # Initialize covariances based on local surface geometry
    pass
```

**Parameters:**
*   **Input View Count:** Fewer than 15 views. Performance degrades gracefully to as few as 3 views [S1, unofficial].
*   **SH Neighbors:** 20 canonical neighbors for smoothness regularization [S1, unofficial].
*   **Camera Calibration:** Wide-baseline, posed images. Poses are not necessarily pre-calibrated perfectly; they are refined during optimization [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **CAD Mismatch:** The matched CAD model rarely matches the physical object perfectly. Without the deformation field, the system treats the CAD model as rigid, causing ghosting [S1, unofficial].
*   **Misregistration:** Silhouette-based registration is only coarsely accurate. Without the geometric warmup phase, texture is fitted to a misregistered surface, and geometry never recovers [S1, unofficial].
*   **View Count Sensitivity:** CAD initialization is superior to generic sphere initialization in sparse regimes (e.g., 3 views). However, with 9-13 views, sphere initialization becomes competitive or slightly superior, as photometric signals are rich enough for the deformation field to carve the object [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   On SCO-CAD with 3 views, CAD initialization yields PSNR 20.710 vs 19.312 for sphere, and IoU 0.929 vs 0.879 [S1, unofficial].
    *   On SCO-CAD full budget, sphere initialization yields PSNR 23.535 vs 23.239 for CAD, and IoU 0.946 vs 0.942 [S1, unofficial].
    *   Deformation alone (no pose opt) achieves PSNR 22.92 and IoU 0.941, within 0.3 dB of the full method [S1, unofficial].
    *   Pose optimization alone (no deformation) achieves PSNR 22.67 [S1, unofficial].
    *   Without mask alignment, reconstruction collapses to PSNR 18.88 and IoU 0.784 [S1, unofficial].
    *   SH neighbor smoothness increases PSNR from 22.93 to 23.13 and reduces mLPIPS from 0.039 to 0.032 [S1, unofficial].
    *   On NeRS MVMC dataset, CADSplat achieves PSNR 18.7, SSIM 0.767, LPIPS 0.157, outperforming NeRS (PSNR 16.5) and CADSim (PSNR 17.7) [S1, unofficial].
*   **Scaling:** The method is designed for sparse views. The CAD prior becomes more meaningful as views become scarcer [S1, unofficial].
*   **Strengths:** Provides geometric disambiguation under occlusion and scarcity. Enables applications beyond novel-view synthesis, such as markerless AR registration and part label transfer.
*   **Limitations:** Requires a CAD model library or a specific CAD model. Performance degrades if the CAD model is a poor match for the physical object (though deformation helps).
*   **When to use:** Sparse-view reconstruction of objects with available CAD models.
*   **When not to use:** Dense-view reconstruction where generic initialization is competitive. Objects without CAD models.

## Background
CADSplat was introduced in 2026 by researchers from the Digital Future Lab, Flanders Make, Hasselt University, Belgium [S1, unofficial]. The method addresses the challenge of reconstructing digital twins from sparse, wide-baseline images by leveraging explicit CAD priors, contrasting with learned generative priors [S1, unofficial].

## Sources
[S1] CADSplat: Sparse-View 3D Gaussian Splatting Aided by CAD Models for Robust, Photorealistic Digital-Twin Reconstruction, https://arxiv.org/abs/2609.18473, fetched 2026-10-30 (unofficial)
