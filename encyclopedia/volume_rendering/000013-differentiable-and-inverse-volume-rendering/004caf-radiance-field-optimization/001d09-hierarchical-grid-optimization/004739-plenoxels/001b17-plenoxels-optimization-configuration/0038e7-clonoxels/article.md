# Clonoxels

Also called: Cloning-Plenoxels

## Overview
Clonoxels is a voxel-based radiance field framework that automatically detects repeated objects within a scene using instance and semantic segmentation, aligns them via rigid registration, and shares voxel features between instances to improve reconstruction quality and reduce storage requirements [S1, unofficial]. It sets itself apart from standard Plenoxels by introducing intra-scene information sharing, allowing gradients to propagate to occluded regions and enabling the recovery of hidden structures by drawing information from other instances of the same object [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images with known camera parameters (intrinsics and extrinsics), instance segmentation masks, and semantic labels for the training set.
*   **Outputs:** A shared voxel feature grid, learnable rigid transformation parameters (poses) for each instance, and scalar color correction values. The final output can be rendered images or extracted 3D meshes via marching cubes [S1, unofficial].

**Data Structures:**
1.  **Baseline Grid:** A regular 3D voxel grid storing opacity values $\sigma$ and view-dependent color $c$ represented as spherical harmonic (SH) coefficients (9 per color channel), following the Plenoxels baseline [S1, unofficial].
2.  **Object Subgrid:** An axis-aligned bounding cube (AABB) of voxels assigned to a specific detected instance. Features are stored at the vertices of this subgrid.
3.  **Transformation Parameters:** For each instance, a learnable rigid transformation defined by:
    *   Rotation matrix $R \in SO(3)$, derived from a learnable quaternion.
    *   Center of rotation $o \in \mathbb{R}^3$.
    *   Translation vector $t \in \mathbb{R}^3$.
4.  **Color Correction:** A single scalar voxel feature $\Delta c$ per subgrid voxel, applied equally to all three RGB channels [S1, unofficial].

**Formulas:**
*   **Volume Rendering:** The color $\hat{C}$ of a camera ray $x(t) = x_o + t \hat{d}$ is computed as:
    $$ \hat{C}(x) = \int_{t_n}^{t_f} V(t) \sigma(x(t)) c(x(t), \hat{d}) dt $$
    where $V(t)$ is the transmittance:
    $$ V(t) = e^{-\int_0^t \sigma(x(s)) ds} $$
    [S1, unofficial]

*   **Rigid Transformation:** Each vertex $v$ of an object's feature subgrid $F$ is transformed to $v'$:
    $$ v' = R(v - o) + o + t $$
    [S1, unofficial]

*   **Color Correction:** The final color $c'$ is:
    $$ c' = c + \Delta c \cdot \mathbf{1} $$
    where $\mathbf{1} = (1, 1, 1)^T$. The result is clamped to $[0, 1]$ [S1, unofficial].

*   **Loss Functions:**
    *   Reconstruction Loss: $L_{recon} = ||C - \hat{C}||_2^2$ [S1, unofficial].
    *   Total Variation Loss: $L_{TV}$ (used for regularization) [S1, unofficial].
    *   Feature Similarity Loss: $L_{feat} = \lambda_{feat} ||F' - \hat{F}||_1$, where $F'$ is the transformed subgrid of one instance, $\hat{F}$ is the subgrid of another instance of the same class, and $\lambda_{feat}$ is a weighting coefficient. L1 is used for outlier tolerance [S1, unofficial].

**Algorithm Steps:**

1.  **Instance Detection and Ranking:**
    *   Use instance segmentation (e.g., Mask2Former or SAM2 with YOLOv10 boxes) and semantic labels on training images to extract objects.
    *   Rank semantic classes by instance count. Retain only masks corresponding to the most numerous class to prioritize repetitions [S1, unofficial].

2.  **3D Localization and Subgrid Definition:**
    *   Project the 2D masks into 3D space using multi-view consistency.
    *   Assign voxels to define axis-aligned bounding cubes (Object Subgrids) for each instance [S1, unofficial].

3.  **Initial Scene Optimization:**
    *   Optimize the entire scene's baseline features ($\sigma, c$) using $L_{recon}$ and $L_{TV}$ to build an initial representation [S1, unofficial].

4.  **Pose Estimation (Feature Sharing Setup):**
    *   **Pre-condition:** Cull low-density voxels (e.g., vertices with $\sigma > 1.0$ are kept for point cloud generation, but low-density voxels are removed before pose optimization to ensure stability) [S1, unofficial].
    *   Optimize only the transformation parameters ($R, o, t$) for each instance using $L_{feat}$ until convergence. This aligns the subgrids of repeated objects [S1, unofficial].

5.  **Joint Optimization:**
    *   Reintroduce $L_{recon}$ and $L_{TV}$.
    *   Continue updating features and transformations jointly. This allows instances to exchange information while pose refinement is informed by photometric cues [S1, unofficial].

6.  **Rendering:**
    *   Query the shared feature grid. For a given instance, apply its learned transformation to map the shared features to its local coordinate system.
    *   Apply the scalar color correction $\Delta c$ to handle lighting differences (shadows, specular highlights) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def clonoxels_optimization(scene_data, training_images, masks, semantic_labels):
    """
    Main optimization loop for Clonoxels.
    scene_data: Initial Plenoxels grid (sigma, SH_coeffs)
    training_images: List of target RGB images
    masks: Instance segmentation masks
    semantic_labels: Semantic class labels for masks
    """
    
    # 1. Instance Detection and Ranking
    # Identify repetitions using instance and semantic segmentation
    instances = detect_instances(masks, semantic_labels)
    # Rank by instance count, keep top class
    top_class_instances = rank_and_select_top_class(instances)
    
    # 2. 3D Localization
    # Project masks to 3D to define Object Subgrids
    subgrids = {}
    for instance in top_class_instances:
        subgrid = project_to_3d(instance.mask, training_images.cameras)
        subgrids[instance.id] = subgrid
        
    # 3. Initial Scene Optimization
    # Optimize baseline features with reconstruction and TV loss
    scene_data = optimize_baseline(scene_data, training_images, 
                                   loss_fn=lambda: L_recon(scene_data, training_images) + L_TV(scene_data))
    
    # 4. Pose Estimation
    # Cull low-density voxels to aid pose optimization
    scene_data = cull_low_density(scene_data, threshold=0.15) # Or use sigma > 1.0 for point cloud viz
    
    # Initialize learnable transformations for each subgrid
    transformations = {id: LearnableRigidTransform() for id in subgrids}
    
    # Optimize transformations using feature similarity loss
    for epoch in range(pose_epochs):
        loss_feat = 0
        for instance in top_class_instances:
            subgrid_F = subgrids[instance.id]
            # Apply transformation
            transformed_coords = transformations[instance.id].apply(subgrid_F.vertices)
            # Resample features from scene_data at transformed coords
            # Note: F.grid_sample is used for differentiable resampling
            F_prime = F.grid_sample(scene_data.features, transformed_coords, 
                                    mode='bilinear', padding_mode='border')
            
            # Compare with another instance of the same class (F_hat)
            # Assuming F_hat is from a reference instance of the same class
            F_hat = subgrids[reference_instance.id].features
            loss_feat += lambda_feat * torch.norm(F_prime - F_hat, p=1)
            
        loss_feat.backward()
        # Update transformation parameters only
        optimizer_transforms.step()
        
    # 5. Joint Optimization
    # Reintroduce reconstruction loss
    for epoch in range(joint_epochs):
        loss_recon = 0
        loss_feat = 0
        
        for instance in top_class_instances:
            # Render current view
            rendered_color = render_ray(scene_data, instance.camera, transformations[instance.id])
            # Apply color correction
            delta_c = get_scalar_correction(instance.id)
            final_color = clamp(rendered_color + delta_c * torch.ones(3))
            
            loss_recon += torch.norm(final_color - instance.target_image, p=2)
            
            # Feature similarity loss
            subgrid_F = subgrids[instance.id]
            transformed_coords = transformations[instance.id].apply(subgrid_F.vertices)
            F_prime = F.grid_sample(scene_data.features, transformed_coords, 
                                    mode='bilinear', padding_mode='border')
            F_hat = subgrids[reference_instance.id].features
            loss_feat += lambda_feat * torch.norm(F_prime - F_hat, p=1)
            
        total_loss = loss_recon + loss_feat
        total_loss.backward()
        # Update features and transformations
        optimizer_features.step()
        optimizer_transforms.step()
        
    return scene_data, transformations
```

**Edge Cases and Pitfalls:**
*   **Lighting Differences:** Direct feature sharing fails under different lighting (shadows, specular highlights). The scalar correction $\Delta c$ is essential. Per-channel corrections are not used to maintain memory savings [S1, unofficial].
*   **Symmetric Objects:** Rigid pose registration may struggle with highly symmetric objects, which are sensitive to pose initialization [S1, unofficial].
*   **Visual Differences:** Semantically similar objects may have significant visual differences, compromising feature sharing gains. This is a limitation of the method [S1, unofficial].
*   **Precision:** Calculations assume 32-bit floating-point precision [S1, unofficial].

## Operating characteristics
*   **Memory:** Significantly reduces memory footprint by storing only one instance's features and propagating them via learned poses and color corrections. Memory savings are reported as absolute values (e.g., 13.3 MB to 23.6 MB saved) and percentages (e.g., 66.0% to 76.4% of voxels with $\sigma > 0.15$) [S1, unofficial].
*   **Quality:** Improves reconstruction quality, particularly for occluded novel views. Clonoxels reduces MSE by 21% in occluded novel view rendering compared to the baseline by drawing information from other objects [S1, unofficial].
*   **Color Correction Impact:** Using the scalar color correction $\Delta c$ improves MSE (0.022 vs 0.039), PSNR (26.9 vs 24.2), and SSIM (0.85 vs 0.83) compared to not using it [S1, unofficial].
*   **Denoising:** The method imposes constraints on geometry and appearance, reducing noisy artifacts and producing cleaner 3D geometry suitable for mesh extraction [S1, unofficial].
*   **Limitations:** Performance degrades if repeated objects have significant visual differences or if objects are highly symmetric [S1, unofficial].

## Background
Clonoxels (Cloning-Plenoxels) was introduced by O. Camilleri, M. Volino, and A. Hilton in 2026 [S1, unofficial]. It builds on the Plenoxels framework, which optimizes voxel opacity and spherical harmonic coefficients [S1, unofficial]. The method addresses the limitations of standard voxel-based radiance fields by leveraging intra-scene repetition to enhance reconstruction and reduce storage, offering a practical path for efficient and accurate reconstruction of complex scenes [S1, unofficial].

## Sources
[S1] Clonoxels: Reduce, Reuse, and Register for Efficient Voxel-Based Radiance Fields, https://doi.org/10.2312/cgvc.20261011, fetched 2026-10-26 (unofficial)
