# DVR Training Losses

Also called: DVR Loss Functions, Multi-Loss Training for DVR, Occupancy Network Losses

## Overview
DVR Training Losses constitute a multi-component objective function used to train the occupancy network $f_\theta$ and texture network $t_\theta$ in Differentiable Volumetric Rendering (DVR) [S1][S2, unofficial]. The method solves the problem of optimizing implicit neural representations from 2D image supervision by distinguishing pixels into three disjoint sets ($P_0, P_1, P_2$) based on object mask membership and predicted surface depth finiteness [S1][S2, unofficial]. It sets itself apart from sibling methods by explicitly decoupling photometric consistency, freespace regularization, and occupancy induction into separate loss terms applied to specific pixel subsets, rather than relying on a single global rendering loss [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A batch of training images $I_k$ with corresponding camera intrinsics, extrinsics, and binary object masks $M_k$ [S1][S2, unofficial].
*   Optional ground truth depth maps $D_k$ [S1][S2, unofficial].
*   The current parameters $\theta$ of the occupancy network $f_\theta$ and texture network $t_\theta$ [S1][S2, unofficial].
*   A sampling count $N_p$, the number of pixels to sample from the image plane per batch [S2, unofficial].

**Outputs:**
*   A scalar total loss value $\mathcal{L}(\theta)$ used for backpropagation [S1][S2, unofficial].

### Data Structures
The method operates on sets of pixel coordinates $\mathbf{u}$ on the image plane [S1][S2, unofficial]. These pixels are classified into three mutually exclusive sets:
*   $P_0$: Pixels inside the object mask $M_k$ where the occupancy network predicts a finite surface depth $\hat{d}$ [S1][S2, unofficial].
*   $P_1$: Pixels outside the object mask $M_k$ [S1][S2, unofficial].
*   $P_2$: Pixels inside the object mask $M_k$ where the occupancy network predicts an infinite surface depth (no intersection) [S1][S2, unofficial].

### Numbered Steps
1.  **Sampling:** Randomly sample an image $I_k$ from the training set and select $N_p$ points $\mathbf{u}$ on the image plane [S1][S2, unofficial].
2.  **Classification:** For each sampled point $\mathbf{u}$, determine its membership in $P_0$, $P_1$, or $P_2$ based on the mask $M_k$ and the predicted depth $\hat{d}$ from the occupancy network [S1][S2, unofficial].
3.  **Loss Computation:** Compute the relevant loss terms for each set:
    *   For $\mathbf{u} \in P_0$: Compute $\mathcal{L}_{\text{rgb}}$ and optionally $\mathcal{L}_{\text{depth}}$ and $\mathcal{L}_{\text{normal}}$ [S1][S2, unofficial].
    *   For $\mathbf{u} \in P_1$: Compute $\mathcal{L}_{\text{freespace}}$ [S1][S2, unofficial].
    *   For $\mathbf{u} \in P_2$: Compute $\mathcal{L}_{\text{occupancy}}$ [S1][S2, unofficial].
4.  **Aggregation:** Sum the losses to form the total objective [S1][S2, unofficial].
5.  **Optimization:** Update parameters $\theta$ using gradient descent [S1][S2, unofficial].

### Formulas

**RGB Loss ($\mathcal{L}_{\text{rgb}}$)** [S1][S2, unofficial]
Applied to points $\mathbf{u} \in P_0$. It enforces photo-consistency between the ground truth image features and the rendered image features [S1][S2, unofficial].
$$
\mathcal{L}_{\text{rgb}}(\theta) = \sum_{\mathbf{u} \in P_0} \|\xi(\mathbf{I})_{\mathbf{u}} - \xi(\hat{\mathbf{I}})_{\mathbf{u}}\|
$$
Where:
*   $\xi(\cdot)$ is a feature extractor computing RGB values and optionally image gradients [S1][S2, unofficial].
*   $\|\cdot\|$ is a robust error metric, typically the $L_1$-norm [S1][S2, unofficial].
*   $\hat{\mathbf{I}}$ is the rendered image [S1][S2, unofficial].

**Depth Loss ($\mathcal{L}_{\text{depth}}$)** [S1][S2, unofficial]
Applied to points $\mathbf{u} \in P_0$ when ground truth depth $d$ is available [S1][S2, unofficial].
$$
\mathcal{L}_{\text{depth}}(\theta) = \sum_{\mathbf{u} \in P_0} |d - \hat{d}|_1
$$
Where:
*   $d$ is the ground truth depth value for pixel $\mathbf{u}$ [S1][S2, unofficial].
*   $\hat{d}$ is the predicted surface depth for pixel $\mathbf{u}$ [S1][S2, unofficial].

**Freespace Loss ($\mathcal{L}_{\text{freespace}}$)** [S1][S2, unofficial]
Applied to points $\mathbf{u} \in P_1$. It penalizes finite predicted depth outside the object mask, encouraging the network to predict free space ($f_\theta(\hat{\mathbf{p}}) \approx 0$) [S1][S2, unofficial].
$$
\mathcal{L}_{\text{freespace}}(\theta) = \sum_{\mathbf{u} \in P_1} \text{BCE}(f_\theta(\hat{\mathbf{p}}), 0)
$$
Where:
*   $\text{BCE}(p, y) = -[y \log(p) + (1-y) \log(1-p)]$ is the Binary Cross Entropy [S1][S2, unofficial].
*   $\hat{\mathbf{p}} = \mathbf{r}(\hat{d})$ is the predicted surface point on the ray $\mathbf{r}$ [S1][S2, unofficial].
*   **Edge Case:** If no finite surface depth is predicted for $\mathbf{u} \in P_1$, apply the BCE loss to a randomly sampled point on the ray with target 0 [S1][S2, unofficial].

**Occupancy Loss ($\mathcal{L}_{\text{occupancy}}$)** [S1][S2, unofficial]
Applied to points $\mathbf{u} \in P_2$. It encourages the network to predict occupied space ($f_\theta \approx 1$) along rays where a surface is expected but not found [S1][S2, unofficial].
$$
\mathcal{L}_{\text{occupancy}}(\theta) = \sum_{\mathbf{u} \in P_2} \text{BCE}(f_\theta(\mathbf{r}(d_{\text{random}})), 1)
$$
Where:
*   $d_{\text{random}}$ are uniformly sampled depth values along the ray [S1][S2, unofficial].
*   **Single-View Exception:** In single-view reconstruction experiments, $d_{\text{random}}$ is replaced by the depth of the visual hull (first point on the ray inside all object masks) or ground truth depth if available [S1][S2, unofficial].

**Normal Loss ($\mathcal{L}_{\text{normal}}$)** [S1][S2, unofficial]
Optional smoothness prior applied to points $\mathbf{u} \in P_0$ [S1][S2, unofficial]. It regularizes surface normals by minimizing the distance between the normal at the predicted surface point and a randomly sampled neighbor [S1][S2, unofficial].
$$
\mathcal{L}_{\text{normal}}(\theta) = \sum_{\mathbf{u} \in P_0} \|\mathbf{n}(\hat{\mathbf{p}}_{\mathbf{u}}) - \mathbf{n}(\mathbf{q}_{\mathbf{u}})\|_2
$$
Where:
*   $\mathbf{n}(\cdot)$ denotes the normal vector computed via gradients of $f_\theta$ [S1][S2, unofficial].
*   $\hat{\mathbf{p}}_{\mathbf{u}}$ is the predicted surface point for pixel $\mathbf{u}$ [S1][S2, unofficial].
*   $\mathbf{q}_{\mathbf{u}}$ is a randomly sampled neighbor of $\hat{\mathbf{p}}_{\mathbf{u}}$ [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dvr_losses(
    image: list, 
    mask: list, 
    depth_gt: list, 
    camera_ray_fn: callable, 
    occupancy_net: callable, 
    texture_net: callable, 
    Np: int
) -> tuple:
    """
    Computes the DVR training losses for a single image.
    """
    H, W = len(mask), len(mask[0])
    
    # Step 1: Sample Np random pixel coordinates
    import random
    u_indices = [random.randint(0, H * W - 1) for _ in range(Np)]
    u_y = [idx // W for idx in u_indices]
    u_x = [idx % W for idx in u_indices]
    
    # Get ray origins and directions for sampled pixels
    origins, directions = camera_ray_fn(u_y, u_x)
    
    # Predict surface depth and points for all sampled pixels
    # This requires finding the root of f_theta(r(d)) = 0 along each ray
    # Implementation detail: This is typically done via binary search or Newton's method
    # as described in the parent article [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md)
    predicted_depths, predicted_points = find_surface_depth(origins, directions, occupancy_net)
    
    # Step 2: Classify pixels into P0, P1, P2
    # P0: Inside mask AND finite depth
    # P1: Outside mask
    # P2: Inside mask AND infinite depth (no intersection found)
    
    inside_mask = [mask[y][x] == 1 for x, y in zip(u_x, u_y)]
    # Assume predicted_depths contains a large value (e.g., 1e9) for infinite depth
    INF_THRESHOLD = 1e9
    finite_depth = [d < INF_THRESHOLD for d in predicted_depths]
    
    p0_mask = [i for i in range(Np) if inside_mask[i] and finite_depth[i]]
    p1_mask = [i for i in range(Np) if not inside_mask[i]]
    p2_mask = [i for i in range(Np) if inside_mask[i] and not finite_depth[i]]
    
    losses = {}
    total_loss = 0.0
    
    # Step 3: Compute Losses for P0
    if p0_mask:
        # RGB Loss
        rendered_features = get_rendered_features([u_x[i] for i in p0_mask], [u_y[i] for i in p0_mask], [predicted_points[i] for i in p0_mask], texture_net)
        gt_features = get_image_features(image, [u_x[i] for i in p0_mask], [u_y[i] for i in p0_mask])
        
        # L1 loss on features
        rgb_loss = sum(abs(gt - pred) for gt, pred in zip(gt_features, rendered_features)) / len(p0_mask)
        losses['rgb'] = rgb_loss
        total_loss += rgb_loss
        
        # Depth Loss (if available)
        if depth_gt is not None:
            gt_depth = [depth_gt[y][x] for x, y in zip([u_x[i] for i in p0_mask], [u_y[i] for i in p0_mask])]
            pred_depth = [predicted_depths[i] for i in p0_mask]
            depth_loss = sum(abs(g - p) for g, p in zip(gt_depth, pred_depth)) / len(p0_mask)
            losses['depth'] = depth_loss
            total_loss += depth_loss
            
        # Normal Loss (Optional)
        normals_pred = compute_normals([predicted_points[i] for i in p0_mask], occupancy_net)
        neighbors = sample_neighbors([predicted_points[i] for i in p0_mask])
        normals_neighbor = compute_normals(neighbors, occupancy_net)
        normal_loss = sum(math.sqrt(sum((a-b)**2 for a, b in zip(n1, n2))) for n1, n2 in zip(normals_pred, normals_neighbor)) / len(p0_mask)
        losses['normal'] = normal_loss
        total_loss += normal_loss

    # Step 3: Compute Losses for P1
    if p1_mask:
        # Freespace Loss
        points_for_loss = []
        for i in p1_mask:
            if finite_depth[i]:
                points_for_loss.append(predicted_points[i])
            else:
                # Random point on ray
                random_d = random.uniform(0, 100.0) # MAX_DEPTH placeholder replaced with standard range
                points_for_loss.append([origins[i][j] + directions[i][j] * random_d for j in range(3)])
        
        occupancy_vals = [occupancy_net(p) for p in points_for_loss]
        # Target is 0 (free space)
        freespace_loss = sum(-(0 * math.log(max(v, 1e-7)) + 1 * math.log(max(1-v, 1e-7))) for v in occupancy_vals) / len(p1_mask)
        losses['freespace'] = freespace_loss
        total_loss += freespace_loss

    # Step 3: Compute Losses for P2
    if p2_mask:
        # Occupancy Loss
        points_for_loss = []
        for i in p2_mask:
            # Single-view exception logic would go here if applicable
            # For general case, sample random depths
            random_d = random.uniform(0, 100.0)
            ray_origin = origins[i]
            ray_dir = directions[i]
            points_for_loss.append([ray_origin[j] + ray_dir[j] * random_d for j in range(3)])
            
        occupancy_vals = [occupancy_net(p) for p in points_for_loss]
        # Target is 1 (occupied)
        occupancy_loss = sum(-(1 * math.log(max(v, 1e-7)) + 0 * math.log(max(1-v, 1e-7))) for v in occupancy_vals) / len(p2_mask)
        losses['occupancy'] = occupancy_loss
        total_loss += occupancy_loss

    return total_loss, losses

def find_surface_depth(origins, directions, occupancy_net):
    """
    Finds the surface depth for each ray by solving f_theta(r(d)) = 0.
    Returns predicted_depths and predicted_points.
    """
    # Placeholder for root-finding algorithm described in parent article.
    # In practice, uses binary search or Newton's method.
    predicted_depths = []
    predicted_points = []
    for o, d in zip(origins, directions):
        # Simplified placeholder logic
        predicted_depths.append(1.0) 
        predicted_points.append([o[j] + d[j] * 1.0 for j in range(3)])
    return predicted_depths, predicted_points

def get_rendered_features(u_x, u_y, points, texture_net):
    return [texture_net(p) for p in points]

def get_image_features(image, u_x, u_y):
    return [image[y][x] for x, y in zip(u_x, u_y)]

def compute_normals(points, occupancy_net):
    # Placeholder for gradient computation
    return [[1.0, 0.0, 0.0] for _ in points]

def sample_neighbors(points, num_neighbors=1):
    return [[p[j] + random.uniform(-0.01, 0.01) for j in range(3)] for p in points]
```

### Parameters
*   $\xi(\cdot)$: Feature extractor. Typically uses RGB values and optionally image gradients [S1][S2, unofficial].
*   $\|\cdot\|$: Robust error metric. Typically $L_1$-loss for RGB and Depth losses, while Freespace and Occupancy losses use Binary Cross Entropy [S1][S2, unofficial].
*   $P_0$: Set of pixels inside object mask with finite predicted surface depth [S2, unofficial].
*   $P_1$: Set of pixels outside object mask [S2, unofficial].
*   $P_2$: Set of pixels inside object mask but with infinite predicted surface depth [S2, unofficial].
*   $N_p$: Number of random pixels sampled from the image plane per batch [S2, unofficial].
*   $N$: Number of images in the training set [S2, unofficial].

### Edge Cases and Pitfalls
*   **Single-View Reconstruction:** For the Occupancy Loss ($\mathcal{L}_{\text{occupancy}}$), random depth sampling is replaced by the visual hull depth (first point on the ray inside all object masks) or ground truth depth if available [S1][S2, unofficial].
*   **Freespace Loss with No Depth:** If a pixel in $P_1$ has no predicted finite depth, the Freespace Loss must be applied to a randomly sampled point on the ray rather than the predicted surface point [S1][S2, unofficial].
*   **Gradient Flow:** The gradient from the RGB loss flows through the texture network and the surface point prediction, which depends on the occupancy network [G]. The Freespace and Occupancy losses provide direct gradients to the occupancy network [S1][S2, unofficial].
*   **Root-Finding Queries:** This typically involves multiple queries to the occupancy network $f_\theta$ [G]. The number of queries depends on the precision of the root-finding algorithm (e.g., binary search iterations) [G].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The complexity is dominated by the root-finding operation for surface depth for each sampled pixel $N_p$. The loss computation itself scales linearly with the number of sampled pixels [G].
*   **Memory:** Memory usage is proportional to $N_p$ for storing sampled points, depths, and intermediate gradients [G].

### Measured Performance
*   The method works with as little as one image per object [S1][S2, unofficial].
*   It can incorporate depth information if available, improving performance [S1][S2, unofficial].
*   The Normal Loss is particularly useful for real-world data to enforce natural shapes in unconstrained areas [S1][S2, unofficial].

### Scaling
*   It scales with the number of training images $N$ [S2, unofficial].

### Strengths and Limitations
*   **Strengths:** Explicitly handles different pixel types (inside/outside mask, finite/infinite depth), allowing for targeted supervision [S1][S2, unofficial]. Works with minimal supervision (2D masks) [S1][S2, unofficial].
*   **Limitations:** Relies on accurate object masks. The quality of the Freespace and Occupancy losses depends on the correct classification of pixels into $P_1$ and $P_2$, which in turn depends on the current state of the occupancy network [S1][S2, unofficial].

### When to Use
*   Use when training DVR networks from 2D image supervision with object masks [S1][S2, unofficial].
*   Use when depth information is available to incorporate $\mathcal{L}_{\text{depth}}$ [S1][S2, unofficial].
*   Use when real-world data requires smoothness priors, enabling $\mathcal{L}_{\text{normal}}$ [S1][S2, unofficial].

### When Not to Use
*   Do not use if only 3D ground truth (e.g., point clouds or meshes) is available without 2D images; consider [Training Losses for Implicit Surface Learning](1526-training-losses-for-implicit-surface-learning.md) instead [S1][S2, unofficial].
*   Do not use if object masks are not available, as the classification into $P_0, P_1, P_2$ is not possible [S1][S2, unofficial].

## Background

### Origin
The DVR Training Losses were introduced by Niemeyer et al. in the paper "Differentiable Volumetric Rendering: Learning Implicit 3D Representations Without 3D Supervision" [S1][S2, unofficial].

### History and Development
The method was developed to enable the training of implicit neural representations using only 2D image supervision [S1][S2, unofficial]. It builds on the Differentiable Volumetric Rendering (DVR) framework, which allows gradients to flow from the rendered image back to the implicit surface representation [S1][S2, unofficial].

### Where It Is Used
The method is used in single-view and multi-view 3D reconstruction from images, particularly in scenarios where 3D supervision is unavailable or expensive [S1][S2, unofficial]. It is a core component of the Occupancy Networks framework [S1][S2, unofficial].

## Sources

[S1] Differentiable Volumetric Rendering: Learning Implicit 3D ..., https://www.cvlibs.net/publications/Niemeyer2020CVPR.pdf, fetched 2026-10-27
[S2] Differentiable Volumetric Rendering - CVF Open Access, https://openaccess.thecvf.com/content_CVPR_2020/html/Niemeyer_Differentiable_Volumetric_Rendering_Learning_Implicit_3D_Representations_Without_3D_Supervision_CVPR_2020_paper.html, fetched 2026-10-30 (unofficial)
