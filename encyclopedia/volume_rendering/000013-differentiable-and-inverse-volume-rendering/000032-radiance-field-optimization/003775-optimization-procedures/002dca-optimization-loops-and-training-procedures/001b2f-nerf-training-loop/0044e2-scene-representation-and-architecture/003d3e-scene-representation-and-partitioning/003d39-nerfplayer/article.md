# NeRFPlayer

Also called: Decomposed Neural Radiance Fields, NeRFPlayer Decomposed Neural Radiance Fields

## Overview
NeRFPlayer is a dynamic scene representation method that decomposes the scene into stationary, deforming, and new areas to enable efficient streaming and high-quality rendering of dynamic environments [S1]. It solves the problem of temporal coherence and storage efficiency in dynamic NeRFs by explicitly modeling the temporal evolution of scene components rather than treating all dynamic changes as new geometry [S1]. Unlike sibling methods like MegaNeRF or Spatial Partitioning for Distributed NeRF which focus on static or distributed static scenes, NeRFPlayer specifically addresses the spatiotemporal decomposition required for video sequences, achieving higher PSNR than HyperNeRF and significantly faster rendering than DyNeRF [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A sequence of RGB images of a dynamic scene from known camera poses [S1].
*   Backbone network configuration (InstantNGP or TensoRF) [S1].
*   Hyperparameters: $\lambda = 0.1$, $\alpha = 0.01$, threshold $\tau = 0.001$, streaming channels $k=1$ [S1].

**Outputs:**
*   A decomposed neural radiance field consisting of a decomposition field, deformation field, stationary field, and radiance field [S1].
*   Rendered images for arbitrary camera poses and timestamps [S1].

### Data Structures
The representation consists of four main components [S1]:

1.  **Decomposition Field ($f$):**
    *   **Structure:** Explicitly cached features ($V_f$) and a small MLP decoder ($D_f$) [S1].
    *   **Function:** Maps 3D spatial coordinates to probabilities for three area types: stationary, deforming, and new [S1].

2.  **Deformation Field ($d$):**
    *   **Structure:** A 4-layer MLP with width 256 [S1].
    *   **Function:** Computes the velocity vector $\mathbf{v}^*$ for deforming points [S1].

3.  **Stationary Field ($s$):**
    *   **Structure:** A 2-layer MLP with width 64 [S1].
    *   **Function:** Represents the static background components [S1].

4.  **Radiance Field ($r$):**
    *   **Structure:** A 4-layer MLP with width 64, identical in structure to the decoder in the chosen backbone (InstantNGP or TensoRF) [S1].
    *   **Function:** Computes the color $\mathbf{c}$ and density $\sigma$ for the radiance contribution [S1].

5.  **Backbone Features:**
    *   **InstantNGP:** 8 levels, 4 feature dimensions per entry [S1].
    *   **TensoRF:** Follows original TensoRF settings for LLFF datasets [S1].
    *   **Streaming Channels ($k$):** Set to 1 for both backbones [S1].

### Numbered Steps

1.  **Sampling:** Sample points $\mathbf{x}$ along rays for the current camera pose and timestamp [S1].
2.  **Decomposition Probability:** Forward $\mathbf{x}$ through the decomposition field $f$ to obtain probabilities for each area type (stationary, deforming, new) [S1]. Let $P^*$ be the probability associated with the deforming area [S1].
3.  **Velocity Calculation (Conditional):**
    *   If $P^* < \tau$ (where $\tau = 0.001$), set velocity $\mathbf{v}^* = \mathbf{0}$ and skip the deformation field forwarding [S1].
    *   Otherwise, forward $\mathbf{x}$ through the deformation field $d$ to obtain velocity $\mathbf{v}^*$ [S1].
4.  **Field Forwarding:**
    *   Based on the dominant area type probability, forward $\mathbf{x}$ through the appropriate field ($s$ for stationary, $d$ for deforming, or new area handling) [S1].
    *   For deforming areas, the position is adjusted by the velocity $\mathbf{v}^*$ before being passed to the radiance field [S1].
5.  **Radiance Computation:** Forward the processed point through the radiance field $r$ to obtain density $\sigma$ and color $\mathbf{c}$ [S1].
6.  **Rendering:** Compute the final color $C(\mathbf{r})$ using the standard volumetric rendering integral, aggregating contributions from all sampled points [S1, Parent].
7.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ [S1]:
    $$
    \mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg}
    $$
    where $\mathcal{L}_{rec}$ is the reconstruction loss (e.g., MSE between rendered and ground truth RGB), $\mathcal{L}_{reg}$ is a regularization term, and $\lambda = 0.1$ [S1].
8.  **Optimization:** Update network parameters using gradient descent [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerfplayer_render(ray_origin, ray_direction, timestamp, decomposition_field, deformation_field, 
                      stationary_field, radiance_field, backbone_decoder, tau=0.001):
    """
    Renders a single ray using the NeRFPlayer decomposition.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        timestamp: float, time index for the dynamic scene.
        decomposition_field: Function mapping 3D pos to probabilities.
        deformation_field: 4-layer MLP (width 256) mapping 3D pos to velocity.
        stationary_field: 2-layer MLP (width 64) for static background.
        radiance_field: 4-layer MLP (width 64) for color/density.
        backbone_decoder: Decoder for the backbone (InstantNGP/TensoRF).
        tau: Threshold for skipping deformation field.
        
    Returns:
        color: 3D vector, rendered color.
        depth: float, accumulated depth.
    """
    # Standard ray marching setup
    points = sample_points_along_ray(ray_origin, ray_direction)
    T = 1.0  # Transmittance
    color = [0.0, 0.0, 0.0]
    depth = 0.0
    
    for point in points:
        # Step 1: Get decomposition probabilities
        probs = decomposition_field(point)
        p_deforming = probs['deforming']
        
        # Step 2: Determine velocity
        if p_deforming < tau:
            velocity = [0.0, 0.0, 0.0]
            # Skip deformation field forwarding [S1]
        else:
            velocity = deformation_field(point)
            
        # Step 3: Determine field to use for radiance
        # Note: The specific logic for 'new' areas vs 'stationary' vs 'deforming'
        # is implied by the decomposition probabilities. 
        # For deforming areas, the point is displaced by velocity.
        if p_deforming > probs['stationary'] and p_deforming > probs['new']:
            # Deforming area
            displaced_point = point + velocity * (1.0 / fps) # Assuming discrete time steps
            sigma, c = radiance_field(displaced_point)
        elif probs['stationary'] > probs['new']:
            # Stationary area
            sigma, c = stationary_field(point) # Or radiance_field depending on implementation detail [S1]
        else:
            # New area handling (streaming channels)
            sigma, c = radiance_field(point) # Simplified; actual implementation uses streaming channels k [S1]
            
        # Step 4: Volumetric Rendering contribution
        # alpha = 1 - exp(-sigma * delta)
        alpha = 1.0 - math.exp(-sigma * point.delta)
        weight = T * alpha
        color += weight * c
        depth += weight * point.distance
        T *= (1.0 - alpha)
        
        if T < 1e-4:
            break
            
    return color, depth

def nerfplayer_train_step(batch, decomposition_field, deformation_field, stationary_field, 
                          radiance_field, backbone_decoder, lambda_reg=0.1):
    """
    Performs one training step.
    """
    rendered_colors = []
    gt_colors = []
    
    for ray in batch.rays:
        # Render using the forward function defined above
        pred_color, _ = nerfplayer_render(ray.origin, ray.direction, ray.timestamp, 
                                          decomposition_field, deformation_field, 
                                          stationary_field, radiance_field, backbone_decoder)
        rendered_colors.append(pred_color)
        gt_colors.append(ray.color)
        
    # Loss calculation [S1]
    loss_rec = mse(rendered_colors, gt_colors)
    loss_reg = regularization(decomposition_field, deformation_field, stationary_field, radiance_field)
    total_loss = loss_rec + lambda_reg * loss_reg
    
    # Backpropagate and update
    total_loss.backward()
    optimizer.step()
    
    return total_loss
```

### Parameters
*   **Threshold $\tau$:** 0.001. If the probability of a point being in a deforming area is below this, the velocity is set to 0 and the deformation field is skipped [S1].
*   **Deformation Network:** 4-layer MLP, width 256 [S1].
*   **Stationary Field:** 2-layer MLP, width 64 [S1].
*   **Radiance Field:** 4-layer MLP, width 64 [S1].
*   **InstantNGP Backbone:** 8 levels, 4 feature dimensions per entry [S1].
*   **TensoRF Backbone:** Follows original TensoRF settings for LLFF [S1].
*   **Streaming Channels $k$:** 1 [S1].
*   **Loss Hyperparameters:** $\lambda = 0.1$, $\alpha = 0.01$ [S1].

### Edge Cases and Pitfalls
*   **High FPS Multi-Camera Datasets:** Deformation is not used by default for efficiency because modeling every dynamic area as new areas leads to good temporal interpolation performance [S1].
*   **Alpha ($\alpha$) Sensitivity:** 
    *   Large $\alpha$ (e.g., 1) over-suppresses deforming areas, leading to blurry moving objects [S1].
    *   Small $\alpha$ (e.g., 0.005) under-suppresses deforming areas, leading to a noisy scene [S1].
    *   Recommended practice: Start with a relatively large $\alpha$ to penalize deforming areas, then gradually decrease $\alpha$ to allow areas to deform [S1].
*   **Misalignment:** Incorrect deformation modeling can cause misalignment of moving objects, as seen in HyperNeRF [S1].

## Subtopics

### InstantNGP Backbone
Uses hash grid encoding for fast feature lookup. Chosen for its speed and ability to handle high-resolution details. [Link](11368-scene-representation-and-architecture.md)

### TensoRF Backbone
Uses tensor decomposition for compact representation. Chosen for lower bitrate in streaming scenarios. [Link](11368-scene-representation-and-architecture.md)

## Operating characteristics
*   **Training Time (Plenoptic Video):**
    *   Ours-InstantNGP: 22.797 GPU hours [S1].
    *   Ours-TensoRF-VM: 23.238 GPU hours [S1].
    *   Neural Volumes: 28.448 GPU hours [S1].
    *   NeRF-T: 29.580 GPU hours [S1].
    *   DyNeRF: 30.293 GPU hours [S1] [S1].
*   **Rendering Time (Plenoptic Video):**
    *   Ours-InstantNGP: 5.5 s/img [S1].
    *   Ours-TensoRF-VM: 6 s/img [S1].
    *   Neural Volumes: 10.8 s/img [S1].
    *   NeRF-T: 22.1 s/img [S1].
    *   DyNeRF: 30.692 s/img [S1] [S1].
*   **Quality (HyperNeRF-vrig Mean):**
    *   PSNR: 22.4 (Ours-InstantNGP), 22.4 (HyperNeRF), 22.2 (Nerfies), 26.3 (NSFF), 16.9 (NV), 20.1 (NeRF) [S1].
    *   SSIM: 0.814 (Ours-InstantNGP), 0.814 (HyperNeRF), 0.803 (Nerfies), 0.916 (NSFF), 0.571 (NV), 0.745 (NeRF) [S1].
*   **Streaming Bitrate:**
    *   TensoRF-CP achieves low bitrate.
    *   TensoRF-VM has higher cost but obvious performance gain [S1].
    *   First frame requires 96 channels for TensoRF-CP and 4 for TensoRF-VM [S1].
*   **Strengths:** Higher PSNR than HyperNeRF, significantly faster rendering than DyNeRF (12% of the time), better handling of fast-moving objects with TensoRF-VM [S1].
*   **Limitations:** Slightly lower SSIM than HyperNeRF, generating more accurate but less sharp images [S1].

## Background
NeRFPlayer was introduced in the paper "NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields" [S1]. It builds upon the principles of decomposed neural radiance fields to handle dynamic scenes efficiently. The method was developed to address the limitations of existing dynamic NeRF methods in terms of training time, rendering speed, and streaming efficiency [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
