# Training Loss Functions for Implicit 3D Representation

Also called: Multi-task Loss for Implicit Surfaces, RGB, Depth, Freespace, and Occupancy Losses

## Overview
This method defines the multi-task loss function used to train implicit 3D representation networks (specifically the combined network $f_\theta$ and texture network $t_\theta$) using 2D image supervision. It solves the problem of reconstructing 3D shape and texture from limited input (single or multi-view images) by partitioning image pixels into three disjoint sets ($P_0, P_1, P_2$) based on object masks and predicted surface depths, applying specific photometric, depth, occupancy, and normal losses to each set. It sets itself apart from sibling methods by explicitly decoupling the optimization of surface presence (occupancy) from surface appearance (RGB/depth) through targeted losses for empty space (freespace) and missing geometry (occupancy), rather than relying solely on volume rendering integrals.

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Input image $I_k$ and its binary object mask $M_k$.
    *   Optional ground truth depth map $D_k$.
    *   Camera parameters defining rays $\mathbf{r}$.
    *   Network parameters $\theta$ for the occupancy/texture network.
*   **Outputs:**
    *   Scalar loss value $\mathcal{L}(\theta)$ for backpropagation.
    *   Predicted surface depth $\hat{d}$ for each pixel.
    *   Predicted surface point $\hat{\mathbf{p}} = \mathbf{r}(\hat{d})$.
    *   Occupancy probability $f_\theta(\mathbf{p})$.
    *   Texture/color prediction.

### Data Structures and Network Layout
The network outputs a 4-dimensional vector at each queried point $\mathbf{p}$:
1.  One dimension for occupancy probability $f_\theta(\mathbf{p}) \in [0, 1]$.
2.  Three dimensions for texture/color (RGB).

The combined network consists of 5 fully-connected ResNet blocks with ReLU activation. For single-view reconstruction, an encoder network (ResNet-18) produces a 256-dimensional latent code $z$ from the input image.

### Pixel Set Partitioning
Pixels $\mathbf{u}$ on the image plane are categorized into three sets:
1.  **$P_0$ (Inside Mask, Finite Depth):** Pixels $\mathbf{u} \in M_k$ where the occupancy network predicts a finite surface depth $\hat{d}$.
2.  **$P_1$ (Outside Mask):** Pixels $\mathbf{u} \notin M_k$.
3.  **$P_2$ (Inside Mask, Infinite Depth):** Pixels $\mathbf{u} \in M_k$ where the occupancy network predicts no surface (infinite depth).

### Loss Formulas

**1. RGB Loss ($\mathcal{L}_{\text{rgb}}$)**
Applied to pixels in $P_0$. It enforces photometric consistency between the input image features and the rendered/predicted features at the predicted surface point.
$$ \mathcal{L}_{\text{rgb}}(\theta) = \sum_{\mathbf{u} \in P_0} \|\xi(\mathbf{I})_{\mathbf{u}} - \xi(\hat{\mathbf{I}})_{\mathbf{u}}\| $$
*   $\xi(\cdot)$: Feature extractor. Typically uses RGB values and optionally image gradients [S1, unofficial].
*   $\|\cdot\|$: Robust error metric. Typically L1 loss [S1, unofficial].
*   $\hat{\mathbf{I}}_{\mathbf{u}}$: Predicted image feature at the surface point $\hat{\mathbf{p}}$ corresponding to pixel $\mathbf{u}$.

**2. Depth Loss ($\mathcal{L}_{\text{depth}}$)**
Applied to pixels in $P_0$ if ground truth depth $d$ is available.
$$ \mathcal{L}_{\text{depth}}(\theta) = \sum_{\mathbf{u} \in P_0} |d - \hat{d}|_1 $$
*   $d$: Ground truth depth value for pixel $\mathbf{u}$.
*   $\hat{d}$: Predicted surface depth for pixel $\mathbf{u}$.

**3. Freespace Loss ($\mathcal{L}_{\text{freespace}}$)**
Applied to pixels in $P_1$. It penalizes the prediction of geometry in empty space.
$$ \mathcal{L}_{\text{freespace}}(\theta) = \sum_{\mathbf{u} \in P_1} \text{BCE}(f_\theta(\hat{\mathbf{p}}), 0) $$
*   $\text{BCE}(p, y)$: Binary Cross Entropy between prediction $p$ and target $y$.
*   $\hat{\mathbf{p}}$: The predicted surface point $\mathbf{r}(\hat{d})$ if $\hat{d}$ is finite.
*   **Edge Case:** If no surface depth is predicted ($\hat{d}$ is infinite), apply the BCE loss to a randomly sampled point on the ray $\mathbf{r}$ instead [S1, unofficial].

**4. Occupancy Loss ($\mathcal{L}_{\text{occupancy}}$)**
Applied to pixels in $P_2$. It encourages the network to predict occupied space along rays that should contain geometry.
$$ \mathcal{L}_{\text{occupancy}}(\theta) = \sum_{\mathbf{u} \in P_2} \text{BCE}(f_\theta(\mathbf{r}(d_{\text{random}})), 1) $$
*   $d_{\text{random}}$: Uniformly sampled depth values along the ray $\mathbf{r}$ [S1, unofficial].
*   **Single-View/Multi-View Variants:**
    *   In single-view experiments, use the first point on the ray inside all object masks (visual hull depth) or ground truth depth instead of random sampling [S1, unofficial].
    *   If additional depth supervision is available, use the ground truth depth [S1, unofficial].

**5. Normal Loss ($\mathcal{L}_{\text{normal}}$)**
Optional smoothness prior applied to pixels in $P_0$.
$$ \mathcal{L}_{\text{normal}}(\theta) = \sum_{\mathbf{u} \in P_0} \|\mathbf{n}(\hat{\mathbf{p}}_{\mathbf{u}}) - \mathbf{n}(\mathbf{q}_{\mathbf{u}})\|_2 $$
*   $\mathbf{n}(\cdot)$: Normal vector at the queried point.
*   $\hat{\mathbf{p}}_{\mathbf{u}}$: Predicted surface point for pixel $\mathbf{u}$.
*   $\mathbf{q}_{\mathbf{u}}$: A randomly sampled neighbor of $\hat{\mathbf{p}}_{\mathbf{u}}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_training_loss(
    input_image: Tensor,
    input_mask: Tensor,
    ground_truth_depth: Optional[Tensor],
    camera_rays: Tensor,  # Shape [H, W, 3] or similar
    occupancy_network: Module,
    texture_network: Module,
    num_samples: int = 128,
    tau: float = 0.5
) -> Tensor:
    """
    Computes the multi-task loss for implicit 3D representation training.
    
    Args:
        input_image: RGB image tensor.
        input_mask: Binary mask tensor (1 for object, 0 for background).
        ground_truth_depth: Optional depth tensor.
        camera_rays: Ray origins and directions for each pixel.
        occupancy_network: Network f_theta outputting occupancy prob.
        texture_network: Network t_theta outputting color.
        num_samples: Number of depth samples per ray.
        tau: Threshold for occupancy probability.
        
    Returns:
        Total loss scalar.
    """
    H, W = input_image.shape[1], input_image.shape[2]
    total_loss = 0.0
    
    # 1. Sample points along rays and query networks
    # Sample depths uniformly or according to sampling schedule
    depths = sample_depths(camera_rays, num_samples) # Shape [H, W, N]
    points = ray_march(camera_rays, depths) # Shape [H, W, N, 3]
    
    # Query occupancy and texture
    # Note: In practice, this might be batched or use importance sampling
    occupancy_probs = occupancy_network(points) # Shape [H, W, N, 1]
    colors = texture_network(points) # Shape [H, W, N, 3]
    
    # 2. Predict surface depth for each pixel
    # Find the first point where occupancy > tau
    # This is a simplified heuristic; actual implementation may use weighted average or specific root-finding
    surface_indices = find_first_above_threshold(occupancy_probs, tau) # Shape [H, W]
    surface_points = gather(points, surface_indices) # Shape [H, W, 3]
    surface_depths = gather(depths, surface_indices) # Shape [H, W]
    
    # 3. Partition pixels into P0, P1, P2
    inside_mask = (input_mask > 0.5)
    finite_depth = (surface_depths < INF)
    
    P0 = inside_mask & finite_depth
    P1 = ~inside_mask
    P2 = inside_mask & ~finite_depth
    
    # 4. Compute RGB Loss for P0
    if P0.any():
        # Render/predict color at surface points
        predicted_colors = interpolate_texture(colors, surface_points)
        # Extract features (RGB + optional gradients)
        feat_input = extract_features(input_image)
        feat_pred = extract_features(predicted_colors)
        
        # L1 loss on features
        rgb_loss = torch.sum(torch.abs(feat_input[P0] - feat_pred[P0]))
        total_loss += rgb_loss
        
        # 5. Compute Depth Loss for P0 (if available)
        if ground_truth_depth is not None:
            gt_depth = ground_truth_depth[P0]
            pred_depth = surface_depths[P0]
            depth_loss = torch.sum(torch.abs(gt_depth - pred_depth))
            total_loss += depth_loss
            
            # 6. Compute Normal Loss for P0 (optional)
            # Compute normals at surface points and neighbors
            normal_p = compute_normals(surface_points[P0])
            # Sample neighbors q_u
            neighbor_offsets = sample_neighbor_offsets(surface_points[P0])
            neighbor_points = surface_points[P0] + neighbor_offsets
            normal_q = compute_normals(neighbor_points)
            normal_loss = torch.sum(torch.norm(normal_p - normal_q, dim=1))
            total_loss += normal_loss

    # 7. Compute Freespace Loss for P1
    if P1.any():
        # For pixels outside mask, penalize occupancy
        # If finite depth predicted, use that point
        # If infinite, use random sample on ray
        freespace_points = torch.zeros_like(surface_points)
        has_finite = finite_depth[P1]
        
        # Points with finite depth
        if has_finite.any():
            idx_finite = torch.where(has_finite)[0]
            # Map back to global indices if necessary, assuming P1 is a mask
            # Simplified: gather from surface_points where P1 is true
            pts_finite = surface_points[P1][has_finite]
            freespace_points[P1][has_finite] = pts_finite
            
        # Points with infinite depth: sample random point on ray
        if (~has_finite).any():
            # Sample a random depth along the ray for these pixels
            random_depths = torch.rand((~has_finite).sum(), 1) * max_depth
            random_points = ray_march_single(camera_rays[P1][~has_finite], random_depths)
            freespace_points[P1][~has_finite] = random_points
            
        # Query occupancy at these points
        # Note: occupancy_network might need to handle batched points
        occ_vals = occupancy_network(freespace_points[P1])
        # Target is 0 (empty space)
        freespace_loss = binary_cross_entropy(occ_vals, torch.zeros_like(occ_vals))
        total_loss += freespace_loss

    # 8. Compute Occupancy Loss for P2
    if P2.any():
        # Encourage occupancy (target 1)
        # Sample random depths or use visual hull/GT depth
        occ_targets = torch.ones((P2.sum(), 1))
        
        # Determine query points
        # In single-view: use visual hull depth or GT depth
        # In general: uniform random depth
        query_points = sample_occupancy_query_points(camera_rays[P2], ground_truth_depth, input_mask)
        
        occ_vals = occupancy_network(query_points)
        occupancy_loss = binary_cross_entropy(occ_vals, occ_targets)
        total_loss += occupancy_loss
        
    return total_loss

def binary_cross_entropy(predictions: Tensor, targets: Tensor) -> Tensor:
    # Standard BCE implementation
    eps = 1e-7
    predictions = torch.clamp(predictions, eps, 1 - eps)
    return -torch.mean(targets * torch.log(predictions) + (1 - targets) * torch.log(1 - predictions))

def find_first_above_threshold(probs: Tensor, threshold: float) -> Tensor:
    # Returns index of first sample > threshold along the ray dimension
    # Shape [H, W]
    above = (probs > threshold).float()
    # Cumulative sum to find first occurrence
    cumsum = torch.cumsum(above, dim=2)
    # Find where cumsum transitions from 0 to 1
    # This is a simplified logic; robust implementation uses argmax on cumsum
    indices = torch.argmax(cumsum, dim=2)
    return indices
```

### Parameters and Settings
*   **Batch Size:** 64 images [S1, unofficial].
*   **Pixels per Image:** 1024 random pixels sampled for loss computation [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **Learning Rate:** $\gamma = 10^{-4}$, decreased by a factor of 5 after 750 and 1000 epochs [S1, unofficial].
*   **Sampling Accuracy:** Starts at $n=16$, increases to $n=128$ by doubling after 50, 150, and 250 thousand iterations [S1, unofficial].
*   **Occupancy Threshold ($\tau$):** 0.5 [S1, unofficial].
*   **Hardware:** Single NVIDIA V100 GPU [S1, unofficial].

### Edge Cases and Pitfalls
*   **Infinite Depth Prediction:** If the network predicts no surface (infinite depth) for a pixel in $P_0$, the RGB and Depth losses cannot be computed directly. The method relies on $P_2$ (Occupancy Loss) to eventually pull the surface into existence, but this requires careful balancing of loss weights [S1, unofficial].
*   **Freespace Ambiguity:** If a pixel is outside the mask ($P_1$) but the network predicts a finite depth, the Freespace Loss is applied to that predicted point. If no depth is predicted, a random point on the ray is used. This ensures empty space is penalized regardless of prediction state [S1, unofficial].
*   **Single-View vs Multi-View:** The definition of query points for Occupancy Loss changes. Single-view uses visual hull or GT depth; multi-view may use random sampling. Using the wrong strategy can lead to poor convergence [S1, unofficial].
*   **Gradient Computation:** The Normal Loss requires differentiable normal estimation, which depends on the differentiability of the occupancy network's output with respect to spatial coordinates [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** Linear with respect to the number of sampled pixels ($N_p$) and the number of depth samples per ray ($n$). The iterative increase of $n$ from 16 to 128 increases computational cost over training time [S1, unofficial].
*   **Memory:** Depends on the batch size (64 images) and the number of samples per pixel. The network size is moderate (5 ResNet blocks, 256-dim latent code for encoder) [S1, unofficial].

### Measured Performance
*   Trained on a single NVIDIA V100 GPU [S1, unofficial].
*   Effective with as little as one image per object [S1, unofficial].
*   Can incorporate depth information if available [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Explicitly handles empty space (Freespace Loss) and missing geometry (Occupancy Loss), providing stable training for implicit surfaces. Works with 2D/2.5D supervision only.
*   **Limitations:** Relies on accurate object masks for $P_0$ and $P_1$. Performance degrades if masks are noisy. The decoupling of occupancy and texture requires careful tuning of loss weights (not specified in sources) [S1, unofficial].

### When to Use
*   When training implicit 3D representations from 2D images without 3D ground truth.
*   When dealing with single-view or sparse multi-view inputs.
*   When depth supervision is optional but beneficial.

### When Not to Use
*   When 3D ground truth (meshes, point clouds) is available and direct 3D losses are preferred.
*   When computational resources are extremely limited and the overhead of multiple loss terms and ray sampling is prohibitive.

## Background

### Origin
The method originates from the paper "Learning Implicit 3D Representations without 3D Supervision" [S1, unofficial]. It was developed to enable the reconstruction of 3D shape and texture from 2D images by learning an implicit occupancy field and a texture field jointly.

### History and Development
The approach builds on the concept of implicit surfaces and differentiable rendering. It specifically addresses the challenge of training such networks without explicit 3D labels by leveraging 2D image consistency and occupancy priors. The use of distinct losses for different pixel regions ($P_0, P_1, P_2$) is a key innovation to stabilize training [S1, unofficial].

### Usage
Used in single-view and multi-view 3D reconstruction tasks, particularly in scenarios with limited supervision. It is a core component of the training setup for the DVR Network Architecture described in the parent article [S1, unofficial].

## Sources

[S1] Learning Implicit 3D Representations without 3D Supervision, https://arxiv.org/abs/1912.07372, fetched 2026-10-27 (unofficial)
