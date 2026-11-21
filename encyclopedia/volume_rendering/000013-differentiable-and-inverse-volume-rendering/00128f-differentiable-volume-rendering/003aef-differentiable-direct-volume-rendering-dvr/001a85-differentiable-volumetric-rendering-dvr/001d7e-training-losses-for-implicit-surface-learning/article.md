# Training Losses for Implicit Surface Learning

Also called: Photometric and Occupancy Losses

## Overview
This method defines a set of loss functions to train an occupancy network $f_\theta$ and a texture network $t_\theta$ using 2D image supervision, categorizing pixels into three distinct sets: $P_0$ (inside the object mask with finite predicted depth), $P_1$ (outside the object mask), and $P_2$ (inside the object mask but with infinite predicted depth) [S1]. It distinguishes itself from sibling methods by explicitly handling the ambiguity of empty space and missing geometry through specialized freespace and occupancy penalties, rather than relying solely on photometric consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A single image $I_k$ with corresponding camera intrinsics/extrinsics, an object mask $M_k$, and optionally ground truth depth $D_k$.
*   **Outputs:** Gradients for $\theta$ (occupancy network parameters) and texture parameters.

**Data Structures:**
*   **Point Set $\mathcal{P}_0$:** Pixels $\mathbf{u}$ inside $M_k$ where the predicted depth $\hat{d}$ is finite.
*   **Point Set $\mathcal{P}_1$:** Pixels $\mathbf{u}$ outside $M_k$.
*   **Point Set $\mathcal{P}_2$:** Pixels $\mathbf{u}$ inside $M_k$ where the predicted depth $\hat{d}$ is infinite.
*   **Ray $\mathbf{r}(d)$:** The ray originating from the camera center through pixel $\mathbf{u}$ at depth $d$.
*   **Surface Point $\hat{\mathbf{p}}$:** The point $\mathbf{r}(\hat{d})$ on the predicted surface.

**Parameters:**
*   $N_p$: Number of random pixels sampled on the image plane per training step [S1].
*   $\xi(\cdot)$: Feature extractor for RGB loss. Uses RGB values and optionally image gradients [S1].
*   $\|\cdot\|$: Robust error metric. Uses L1 loss in practice [S1].

**Steps:**
1.  Randomly sample an image $I_k$ and $N_p$ points $\mathbf{u}$ on the image plane [S1].
2.  Classify each $\mathbf{u}$ into sets $\mathcal{P}_0$, $\mathcal{P}_1$, or $\mathcal{P}_2$ based on the mask $M_k$ and the predicted surface depth $\hat{d}$ [S1].
3.  Compute the respective losses for each set (formulas below) [S1].
4.  Sum the losses and backpropagate [S1].

**Formulas:**

**RGB Loss (for $\mathcal{P}_0$):**
Photo-consistency loss using the feature extractor $\xi$ and robust metric $\|\cdot\|$ [S1]:
$$
\mathcal{L}_{\text{rgb}}(\theta) = \sum_{\mathbf{u} \in \mathcal{P}_0} \|\xi(\mathbf{I})_{\mathbf{u}} - \xi(\hat{\mathbf{I}})_{\mathbf{u}}\|
$$
In practice, $\xi$ returns RGB values and optional gradients, and $\|\cdot\|$ is the L1 norm [S1].

**Depth Loss (for $\mathcal{P}_0$, if depth $D_k$ is available):**
L1 loss on predicted vs. ground truth depth [S1]:
$$
\mathcal{L}_{\text{depth}}(\theta) = \sum_{\mathbf{u} \in \mathcal{P}_0} |d - \hat{d}|_1
$$
where $d$ is the ground truth depth and $\hat{d}$ is the predicted depth [S1].

**Freespace Loss (for $\mathcal{P}_1$):**
Binary Cross Entropy (BCE) penalizing occupancy where the surface should not exist [S1]. If a finite surface depth $\hat{d}$ is predicted for a point outside the mask, the network falsely predicts a surface point $\hat{\mathbf{p}} = \mathbf{r}(\hat{d})$. The loss is:
$$
\mathcal{L}_{\text{freespace}}(\theta) = \sum_{\mathbf{u} \in \mathcal{P}_1} \text{BCE}(f_\theta(\hat{\mathbf{p}}), 0)
$$
If no surface depth is predicted (infinite $\hat{d}$), the freespace loss is applied to a randomly sampled point on the ray [S1].

**Occupancy Loss (for $\mathcal{P}_2$):**
BCE encouraging occupancy along rays where a surface is expected but not found [S1]. Uniformly sample depth values $d_{\text{random}}$:
$$
\mathcal{L}_{\text{occupancy}}(\theta) = \sum_{\mathbf{u} \in \mathcal{P}_2} \text{BCE}(f_{\theta}(\mathbf{r}(d_{\text{random}})), 1)
$$
In single-view experiments, $d_{\text{random}}$ is replaced by the visual hull depth (first point inside all object masks) or ground truth depth if available [S1].

**Normal Loss (optional, for $\mathcal{P}_0$):**
Smoothness prior regularizing surface normals [S1]:
$$
\mathcal{L}_{\text{normal}}(\theta) = \sum_{\mathbf{u} \in \mathcal{P}_0} \|\mathbf{n}(\mathbf{p}_{\mathbf{u}}) - \mathbf{n}(\mathbf{q}_{\mathbf{u}})\|_2
$$
where $\mathbf{n}(\cdot)$ is the normal vector, $\hat{\mathbf{p}}_{\mathbf{u}}$ is the predicted surface point, and $\mathbf{q}_{\mathbf{u}}$ is a randomly sampled neighbor of $\hat{\mathbf{p}}_{\mathbf{u}}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_training_loss(image, mask, depth_gt, camera_intrinsics, camera_extrinsics, 
                          occupancy_net, texture_net, Np):
    """
    Computes the total training loss for implicit surface learning.
    
    Args:
        image: Input image I_k.
        mask: Object mask M_k.
        depth_gt: Ground truth depth D_k (optional).
        camera_intrinsics, camera_extrinsics: Camera parameters.
        occupancy_net: Network f_theta predicting occupancy.
        texture_net: Network t_theta predicting texture.
        Np: Number of pixels to sample.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # 1. Sample Np points on the image plane
    u_samples = sample_random_pixels(Np)
    
    # 2. Classify points into P0, P1, P2
    P0 = []
    P1 = []
    P2 = []
    
    for u in u_samples:
        # Project u to 3D ray r
        ray = get_ray_from_pixel(u, camera_intrinsics, camera_extrinsics)
        
        # Predict surface depth d_hat along the ray
        # Note: Implementation detail [G] - typically involves marching or root finding
        d_hat = predict_surface_depth(ray, occupancy_net)
        
        if is_inside_mask(u, mask):
            if is_finite(d_hat):
                P0.append(u)
            else:
                P2.append(u)
        else:
            P1.append(u)
            
    total_loss = 0.0
    
    # 3. Compute RGB Loss for P0
    if P0:
        # Predict image I_hat for points in P0
        I_hat = predict_image_for_points(P0, occupancy_net, texture_net, camera_intrinsics, camera_extrinsics)
        
        # Feature extraction: RGB + optional gradients
        features_gt = extract_features(image, P0)
        features_pred = extract_features(I_hat, P0)
        
        # L1 loss
        l_rgb = sum(l1_norm(f_gt - f_pred) for f_gt, f_pred in zip(features_gt, features_pred))
        total_loss += l_rgb
        
        # 4. Compute Depth Loss for P0 (if depth_gt available)
        if depth_gt is not None:
            d_gt = get_depth_at_pixels(depth_gt, P0)
            d_hat_list = [d_hat for u in P0] # d_hat was computed in step 2
            l_depth = sum(abs(d - d_h) for d, d_h in zip(d_gt, d_hat_list))
            total_loss += l_depth
            
        # 5. Compute Normal Loss for P0 (optional)
        # [G] Implementation detail: Compute normals from occupancy gradient or geometry
        # l_normal = sum(norm_dist(p_normal, q_normal) for p, q in neighbors)
        # total_loss += l_normal
        
    # 6. Compute Freespace Loss for P1
    if P1:
        l_freespace = 0.0
        for u in P1:
            ray = get_ray_from_pixel(u, camera_intrinsics, camera_extrinsics)
            # Check if finite depth was predicted (false positive)
            d_hat = predict_surface_depth(ray, occupancy_net)
            if is_finite(d_hat):
                p_hat = ray.point_at(d_hat)
                occ_val = occupancy_net(p_hat)
                l_freespace += bce(occ_val, 0)
            else:
                # No surface predicted, sample random point on ray
                d_rand = random_depth()
                p_rand = ray.point_at(d_rand)
                occ_val = occupancy_net(p_rand)
                l_freespace += bce(occ_val, 0)
        total_loss += l_freespace
        
    # 7. Compute Occupancy Loss for P2
    if P2:
        l_occupancy = 0.0
        for u in P2:
            ray = get_ray_from_pixel(u, camera_intrinsics, camera_extrinsics)
            # Sample random depth or use visual hull/ground truth
            if single_view_experiment:
                d_sample = get_visual_hull_depth(ray, mask) # or ground truth if available
            else:
                d_sample = random_depth()
                
            p_sample = ray.point_at(d_sample)
            occ_val = occupancy_net(p_sample)
            l_occupancy += bce(occ_val, 1)
        total_loss += l_occupancy
        
    return total_loss
```

## Subtopics

### Normalized Object Coordinate Space (NOCS) Volume Rendering
A method focused on novel view synthesis of objects without background, using 3D bounding boxes and object masks to define Normalized Object Coordinate Spaces for per-object volume rendering [S2]. It sets itself apart by refining shape codes, appearance codes, and bounding boxes at test time using an occupancy loss [S2]. The occupancy loss is defined as $\mathcal{L}_{\text{occ}} = \frac{1}{|W_{\text{occ}}|} \sum_{\mathbf{u} \in W_{\text{occ}}} \log(Y_{\mathbf{u}} (1/2 - \alpha) + 1/2)$, where $Y$ is the object mask and $W_{\text{occ}}$ is either the set of foreground or background pixels [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampled pixels $N_p$ and the number of points in each set $|\mathcal{P}_0|, |\mathcal{P}_1|, |\mathcal{P}_2|$ [S1].
*   **Memory Complexity:** Depends on the storage of the occupancy and texture networks and the intermediate gradients for the sampled points [S1].
*   **Strengths:** Works with as little as one image per object [S1]. Can incorporate depth information if available [S1]. The normal loss is useful for real-world data to enforce natural shapes in unconstrained areas [S1].
*   **Limitations:** Relies on accurate object masks [S1]. The freespace and occupancy losses depend on the correct classification of pixels, which can be sensitive to depth prediction errors [S1].
*   **When to use:** When training implicit neural representations with 2D image supervision, especially in single-view or sparse-view settings [S1].
*   **When not to use:** When 3D ground truth geometry is available and can be used directly for supervision, as 2D-only supervision may lead to ambiguities [S1].

## Background
This method originates from the work on learning implicit 3D representations without 3D supervision [S1]. It was developed to address the challenge of training occupancy networks and texture networks using only 2D image data, masks, and optionally depth [S1]. The approach builds on the differentiable volumetric rendering framework [Parent](775-differentiable-volumetric-rendering-dvr.md).

## Sources
[S1] Differentiable Volumetric Rendering: Learning Implicit 3D Representations without 3D Supervision, https://www.youtube.com/watch?v=lcub1KH-mmk, fetched 2026-10-27
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
