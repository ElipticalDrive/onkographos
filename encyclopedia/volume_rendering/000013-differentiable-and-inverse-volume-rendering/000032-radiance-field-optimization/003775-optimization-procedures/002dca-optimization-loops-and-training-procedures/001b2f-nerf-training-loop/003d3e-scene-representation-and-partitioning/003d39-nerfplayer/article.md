# NeRFPlayer

Also called: Decomposed Neural Radiance Fields, NeRFPlayer Decomposed Neural Radiance Fields

## Overview
NeRFPlayer is a dynamic scene representation method that decomposes the spatiotemporal domain into stationary, deforming, and new area types to enable efficient streaming and rendering of dynamic scenes [S1]. It sets itself apart from siblings like MegaNeRF and BlockNeRF by explicitly modeling temporal dynamics through a decomposition field that assigns probabilities to area types, allowing for selective skipping of expensive field queries and adaptive deformation handling, whereas siblings typically partition space statically without explicit temporal type classification [S1].

## How it works
NeRFPlayer builds on [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by introducing a temporal decomposition mechanism. The method represents a dynamic scene using three distinct neural fields: a decomposition field, a deformation field, and a radiance field (which includes a stationary component).

### Inputs and Outputs
*   **Inputs:** Camera pose $(c, \theta, \phi)$, time $t$, and a set of sampled 3D points $\mathbf{x}$ along the ray.
*   **Outputs:** Color $C$ and opacity $\alpha$ for the ray, computed via volume rendering.

### Data Structures and Layouts
The representation consists of four main components:

1.  **Decomposition Field ($f$):**
    *   **Structure:** Explicitly cached features ($V_f$) and a small MLP decoder ($D_f$).
    *   **Function:** Takes a 3D position $\mathbf{x}$ and outputs a probability distribution over three area types: stationary, deforming, and new.
    *   **Output:** A vector of probabilities $P = [p_{stat}, p_{def}, p_{new}]$.

2.  **Deformation Field ($d$):**
    *   **Structure:** A 4-layer Multi-Layer Perceptron (MLP).
    *   **Width:** 256 units per layer.
    *   **Function:** Maps a 3D position $\mathbf{x}$ and time $t$ to a deformation vector $\mathbf{v}$.

3.  **Stationary Field ($s$):**
    *   **Structure:** A 2-layer MLP.
    *   **Width:** 64 units per layer.
    *   **Function:** Represents the radiance and density of static parts of the scene.

4.  **Radiance Field ($r$):**
    *   **Structure:** A 4-layer MLP.
    *   **Width:** 64 units per layer.
    *   **Function:** Represents the radiance and density of dynamic parts. It shares the same structure as the decoder in the chosen backbone (InstantNGP or TensoRF).

5.  **Backbone (Hybrid Representation):**
    *   The method supports two backbones: InstantNGP and TensoRF.
    *   **InstantNGP:** 8 levels, 4 feature dimensions per entry.
    *   **TensoRF:** Follows original TensoRF settings for forward-facing datasets (LLFF).
    *   **Streaming Channels ($k$):** Set to 1 for both backbones.

### Parameters
*   **Threshold ($\tau$):** Used to skip field forwarding. Default: $0.001$ [S1].
*   **Loss Balancing ($\lambda$):** Balances reconstruction and regularization loss. Default: $0.1$ [S1].
*   **Deformation Penalty ($\alpha$):** Penalizes deforming areas during training. Default: $0.01$ [S1].
*   **Streaming Channels ($k$):** Number of new channels needed for rendering a new frame. Default: $1$ [S1].

### Numbered Steps

1.  **Sample Points:** For a given camera pose and time $t$, sample points $\mathbf{x}_i$ along the ray.
2.  **Query Decomposition Field:** Forward each point $\mathbf{x}_i$ through the decomposition field $f$ to obtain probabilities $P_i = [p_{stat}, p_{def}, p_{new}]$ for each area type [S1].
3.  **Skip Optimization (Efficiency):** If the maximum probability $P^*_i < \tau$ (where $\tau = 0.001$), set the deformation velocity $v^*_i = 0$ and skip the forwarding process of the deformation and radiance fields for this point [S1].
4.  **Compute Deformation:** For points where $P^*_i \ge \tau$, query the deformation field $d$ to get the deformation vector $\mathbf{v}_i = d(\mathbf{x}_i, t)$ [S1].
5.  **Query Radiance Fields:**
    *   If the point is classified as stationary (high $p_{stat}$), query the stationary field $s$.
    *   If the point is classified as deforming or new (high $p_{def}$ or $p_{new}$), query the radiance field $r$ using the deformed position $\mathbf{x}_i + \mathbf{v}_i$ [S1].
6.  **Volume Rendering:** Composite the color and opacity from the sampled points using standard volume rendering equations [S1].
7.  **Loss Calculation:** Compute the total loss $\mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg}$, where $\mathcal{L}_{rec}$ is the reconstruction loss and $\mathcal{L}_{reg}$ is the regularization loss [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerfplayer_render(camera_pose, time_t, ray_samples):
    """
    Renders a single ray using the NeRFPlayer representation.
    
    Args:
        camera_pose: Camera extrinsics and intrinsics.
        time_t: Current time step.
        ray_samples: List of 3D points sampled along the ray.
        
    Returns:
        Rendered color and opacity.
    """
    # Constants
    THRESHOLD = 0.001
    LAMBDA = 0.1
    ALPHA = 0.01
    
    # Initialize accumulators
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    transmittance = 1.0
    
    for point in ray_samples:
        # Step 1: Query Decomposition Field
        # f consists of cached features Vf and MLP decoder Df
        probs = decomposition_field(point) # Returns [p_stat, p_def, p_new]
        max_prob = max(probs)
        
        # Step 2: Skip Optimization
        if max_prob < THRESHOLD:
            # Set velocity to 0 and skip field forwarding
            velocity = [0.0, 0.0, 0.0]
            # Assume static properties for skipped points or use default
            # In practice, this might imply querying a default static field
            # or skipping entirely if probability is negligible.
            # Based on source: "skip the field"
            continue
            
        # Step 3: Compute Deformation
        # d is a 4-layer MLP with width 256
        velocity = deformation_field(point, time_t)
        
        # Step 4: Determine Field to Query
        # Identify dominant type
        if probs[0] > probs[1] and probs[0] > probs[2]:
            # Stationary
            # s is a 2-layer MLP with width 64
            density, color = stationary_field(point)
        else:
            # Deforming or New
            # r is a 4-layer MLP with width 64
            # Use deformed position
            deformed_point = point + velocity
            density, color = radiance_field(deformed_point)
            
        # Step 5: Volume Rendering
        alpha = 1.0 - exp(-density * step_size)
        weight = transmittance * alpha
        accumulated_color += weight * color
        accumulated_alpha += weight
        transmittance *= (1.0 - alpha)
        
    return accumulated_color, accumulated_alpha

def train_step(batch):
    """
    Training step for NeRFPlayer.
    """
    # Forward pass
    rendered_colors, _ = nerfplayer_render(...)
    
    # Loss Calculation
    # L = L_rec + lambda * L_reg
    loss_rec = reconstruction_loss(rendered_colors, ground_truth)
    loss_reg = regularization_loss(...) # Specific reg terms not detailed in notes
    total_loss = loss_rec + LAMBDA * loss_reg
    
    # Backpropagation
    total_loss.backward()
    
    # Optimization
    optimizer.step()
    
    # Adaptive Alpha Schedule
    # Start with large alpha, gradually decrease
    current_alpha = ALPHA * schedule_factor(step)
    
    return total_loss
```

### Edge Cases and Pitfalls
*   **High FPS Multi-Camera Datasets:** Deformation is not used by default for efficiency because modeling every dynamic area as new areas leads to good temporal interpolation performance [S1].
*   **Alpha Sensitivity:** A large $\alpha$ (e.g., 1) over-suppresses deforming areas, leading to blurry moving objects. A small $\alpha$ (e.g., 0.005) under-suppresses deforming areas, leading to a noisy scene [S1]. Recommended practice is to start with a relatively large $\alpha$ and gradually decrease it [S1].
*   **Streaming Bitrate:** The bitrate depends on the number of streaming channels $k$. TensoRF-CP achieves low bitrate, while TensoRF-VM has higher cost but better performance [S1].

## Subtopics

### InstantNGP Backbone
Uses 8 levels and 4 feature dimensions per entry. Offers faster rendering but slightly lower SSIM compared to TensoRF-VM on some metrics [S1].

### TensoRF Backbone
Follows original TensoRF settings. TensoRF-VM achieves better performance on fast-moving objects compared to InstantNGP [S1].

## Operating characteristics
*   **Training Time:** On Plenoptic Video, NeRFPlayer (InstantNGP) takes 22.797 GPU hours, and (TensoRF-VM) takes 23.238 GPU hours, compared to 28.448 (Neural Volumes), 29.580 (NeRF-T), and 30.293 (DyNeRF) [S1].
*   **Rendering Time:** On Plenoptic Video, NeRFPlayer (InstantNGP) takes 5.5 s/img, and (TensoRF-VM) takes 6 s/img, compared to 10.8 s/img (Neural Volumes), 22.1 s/img (NeRF-T), and 30.692 s/img (DyNeRF) [S1]. NeRFPlayer with InstantNGP renders images in 12% of the time required by DyNeRF [S1].
*   **Quality (PSNR/SSIM):** On HyperNeRF-vrig, NeRFPlayer (InstantNGP) achieves a mean PSNR of 22.4 and SSIM of 0.814, comparable to HyperNeRF (PSNR 22.4, SSIM 0.814) but with more accurate and less sharp images [S1].
*   **Streaming:** The method supports streaming by tuning the number of new channels $k$. For TensoRF-CP, the first frame requires 96 channels, while TensoRF-VM requires 4 [S1].
*   **Strengths:** High rendering speed, efficient training, good temporal interpolation for high FPS data, and support for streaming.
*   **Limitations:** May produce less sharp images compared to HyperNeRF; deformation modeling can be sensitive to $\alpha$; high FPS datasets may not benefit from deformation field [S1].

## Background
NeRFPlayer was introduced in the paper "NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields" [S1]. It addresses the challenges of dynamic scene reconstruction and rendering by decomposing the scene into stationary, deforming, and new areas, allowing for efficient handling of temporal changes. The method is designed to be general and can be integrated with various hybrid representations like InstantNGP and TensoRF [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
