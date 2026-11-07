# NeRFPlayer

Also called: Decomposed Neural Radiance Fields, NeRFPlayer Decomposed Neural Radiance Fields

## Overview
NeRFPlayer is a dynamic scene representation method that decomposes a spatiotemporal volume into stationary, deforming, and new (streaming) areas to enable efficient training and streaming of dynamic radiance fields [S1]. It distinguishes itself from siblings like MegaNeRF and BlockNeRF by explicitly modeling temporal dynamics through a decomposition field that probabilistically assigns points to static, deforming, or new categories, allowing for sparse deformation modeling and low-bitrate streaming [S1]. Unlike methods that treat all dynamic regions uniformly, NeRFPlayer separates static backgrounds from moving objects, reducing flickering and misalignment artifacts common in hyper-neural approaches [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** RGB images of a dynamic scene from varying camera poses and timestamps. No special capturing settings or prior knowledge (e.g., optical flow, depth) are required [S1].
- **Outputs:** A decomposed neural radiance field representation allowing for novel view synthesis and temporal interpolation of dynamic scenes [S1].

**Data Structures:**
The representation consists of four main components:
1.  **Decomposition Field ($f$):** Composed of explicitly cached features ($V_f$) and a small MLP decoder ($D_f$). It outputs probabilities for each area type (stationary, deforming, new) for a given spatial-temporal point [S1].
2.  **Deformation Field ($d$):** A 4-layer MLP with width 256. It predicts velocity vectors for deforming points [S1].
3.  **Stationary Field ($s$):** A 2-layer MLP with width 64. It encodes the static background [S1].
4.  **Radiance Field ($r$):** A 4-layer MLP with width 64. It has the same structure as the decoder in the chosen backbone (InstantNGP or TensoRF) and predicts color and density [S1].

**Backbone Integration:**
NeRFPlayer is a framework that can be built on top of static scene backbones. Two implementations are provided:
-   **InstantNGP Backbone:** Uses 8 levels of hash grids with 4 feature dimensions per entry [S1].
-   **TensoRF Backbone:** Follows settings from original TensoRF experiments on real forward-facing datasets (LLFF) [S1].
-   **Streaming Channels ($k$):** Set to 1 for both backbones, representing the number of new channels needed to render a new frame [S1].

**Algorithm Steps:**
1.  **Sampling:** Sample points along rays for the target camera pose and timestamp [S1].
2.  **Decomposition Probability:** Forward sampled points through the decomposition field ($f$) to obtain probabilities for each area type: stationary ($P_{stat}$), deforming ($P_{def}$), and new ($P_{new}$) [S1].
3.  **Field Selection and Deformation:**
    -   If the point is classified as **stationary**, use the stationary field ($s$) directly [S1].
    -   If the point is classified as **deforming**:
        -   Check if probability $P^* < \tau$ (threshold). If true, set velocity $v^* = 0$ and skip the deformation field forwarding for efficiency [S1].
        -   If $P^* \ge \tau$, forward the point through the deformation field ($d$) to obtain velocity $v^*$ [S1].
        -   Apply velocity to warp the point to its deformed position [S1].
        -   Forward the deformed point through the radiance field ($r$) [S1].
    -   If the point is classified as **new** (streaming), use the radiance field ($r$) with updated streaming channels [S1].
4.  **Rendering:** Aggregate color and density from the selected fields using standard volume rendering integration [S1].

**Loss Function:**
The training loss is defined as:
$$ L = L_{rec} + \lambda L_{reg} $$
where $L_{rec}$ is the reconstruction loss (e.g., MSE) and $L_{reg}$ is a regularization term [S1].

**Parameters:**
-   **Threshold ($\tau$):** 0.001. Used to skip field forwarding if probability is low [S1].
-   **Deformation Network:** 4-layer MLP, width 256 [S1].
-   **Stationary Field:** 2-layer MLP, width 64 [S1].
-   **Radiance Field:** 4-layer MLP, width 64 [S1].
-   **InstantNGP Levels:** 8 [S1].
-   **InstantNGP Feature Dimensions:** 4 [S1].
-   **Streaming Channels ($k$):** 1 [S1].
-   **Loss Hyper-parameter ($\lambda$):** 0.1 [S1].
-   **Deformation Penalty ($\alpha$):** 0.01. Used to penalize deforming areas during training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, timestamp, decomposition_field, deformation_field, stationary_field, radiance_field, tau=0.001):
    """
    Renders a single ray using the NeRFPlayer decomposition.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        timestamp: float, time index for the dynamic scene.
        decomposition_field: Function returning probabilities [P_stat, P_def, P_new].
        deformation_field: Function returning velocity vector.
        stationary_field: Function returning color/density for static points.
        radiance_field: Function returning color/density for deformed/new points.
        tau: float, threshold for skipping deformation.
        
    Returns:
        color: 3D vector, rendered color.
        depth: float, accumulated depth.
    """
    # 1. Sample points along the ray
    points = sample_points_along_ray(ray_origin, ray_direction)
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    accumulated_depth = 0.0
    
    for point in points:
        # 2. Get decomposition probabilities
        probs = decomposition_field(point, timestamp)
        P_stat, P_def, P_new = probs
        
        # 3. Determine field and deformation
        if P_stat > P_def and P_stat > P_new:
            # Stationary area
            color, density = stationary_field(point)
        elif P_def > P_stat and P_def > P_new:
            # Deforming area
            # Check threshold to skip deformation for efficiency
            if P_def < tau:
                velocity = [0.0, 0.0, 0.0]
            else:
                velocity = deformation_field(point, timestamp)
            
            # Warp point based on velocity
            deformed_point = point + velocity
            color, density = radiance_field(deformed_point)
        else:
            # New/Streaming area
            color, density = radiance_field(point)
            
        # 4. Volume rendering integration (standard alpha compositing)
        # Note: Standard NeRF integration assumes density is sigma
        alpha = 1.0 - exp(-density * dist) # dist is distance between samples
        weight = alpha * (1.0 - accumulated_alpha)
        
        accumulated_color += weight * color
        accumulated_alpha += weight
        accumulated_depth += weight * point.z # Example depth metric
        
        if accumulated_alpha > 0.99:
            break
            
    return accumulated_color, accumulated_depth
```

**Edge Cases and Pitfalls:**
-   **High FPS Multi-Camera Datasets:** Deformation is not used by default for efficiency because modeling every dynamic area as new areas leads to good temporal interpolation performance [S1].
-   **Alpha ($\alpha$) Tuning:** A large $\alpha$ (e.g., 1) over-suppresses deforming areas, leading to blurry moving objects. A small $\alpha$ (e.g., 0.005) under-suppresses deforming areas, leading to a noisy scene [S1]. Recommended practice is to start with a relatively large $\alpha$ to penalize deforming areas, then gradually decrease $\alpha$ to allow areas to deform [S1].
-   **HyperNeRF Misalignment:** Unlike HyperNeRF, which suffers from misalignment of moving objects due to incorrect deformation modeling (treating all points as deforming), NeRFPlayer's decomposition reduces this issue [S1].

## Subtopics

### InstantNGP Backbone
Uses hash-based grid encoding for fast lookup. Chosen for high rendering speed and lower training time compared to TensoRF. [Link](8580-meganerf.md) (Note: InstantNGP is a backbone, not a sibling in the partitioning sense, but relevant for implementation choice).

### TensoRF Backbone
Uses tensor decomposition for compact representation. Chosen for lower bitrate in streaming scenarios, though with higher computational cost than InstantNGP. [Link](8581-blocknerf.md)

## Operating characteristics
**Time and Memory Complexity:**
-   **Training Time:** NeRFPlayer with InstantNGP backbone requires 22.797 GPU hours on Plenoptic Video, compared to 28.448 (Neural Volumes), 29.580 (NeRF-T), and 30.293 (DyNeRF) [S1].
-   **Rendering Time:** NeRFPlayer with InstantNGP renders at 5.5 s/img on Plenoptic Video, compared to 6 s/img (TensoRF-VM), 10.8 s/img (Neural Volumes), 22.1 s/img (NeRF-T), and 30.692 s/img (DyNeRF) [S1].
-   **Streaming Bitrate:** For TensoRF-CP, the first frame requires 96 channels, and subsequent frames require $k=1$ new channel. For TensoRF-VM, the first frame requires 4 channels, and subsequent frames require $k=1$ new channel. TensoRF-CP achieves lower bitrate, while TensoRF-VM has higher cost but better performance [S1].

**Measured Performance:**
-   **Plenoptic Video (Multi-Camera):**
    -   Ours-InstantNGP: 22.797h training, 5.5s/img rendering.
    -   Ours-TensoRF-VM: 23.238h training, 6s/img rendering.
-   **HyperNeRF-vrig (Single-Camera):**
    -   Ours-InstantNGP: PSNR 22.4, SSIM 0.814 [S1].
    -   HyperNeRF: PSNR 22.4, SSIM 0.814 [S1].
    -   NSFF: PSNR 26.3, SSIM 0.916 [S1].
    -   Neural Volumes (NV): PSNR 16.9, SSIM 0.571 [S1].
    -   NeRF: PSNR 20.1, SSIM 0.745 [S1].
-   **Ablation Studies (HyperNeRF-interp):**
    -   Full decomposition PSNR: 29.2.
    -   Without deforming: PSNR 28.2.
    -   Without new: PSNR 28.2 [S1].
-   **Ablation Studies (Immersive Video):**
    -   Horse (w/o static): PSNR 27.0; (w/ static): PSNR 27.4 [S1].
    -   Welder (every 8 frames, w/o deforming): PSNR 26.1; (w/ deforming): PSNR 25.2 [S1].

**Strengths and Limitations:**
-   **Strengths:** Significantly faster training and rendering than DyNeRF (12% of the time) with comparable quality [S1]. Better performance on fast-moving objects with TensoRF-VM backbone [S1]. Lower bitrate for streaming with TensoRF-CP [S1].
-   **Limitations:** Slightly lower SSIM than HyperNeRF, generating more accurate but less sharp images [S1]. High FPS multi-camera datasets may not benefit from deformation modeling if treated as new areas [S1].

**When to Use:**
-   Use when efficient training and rendering of dynamic scenes are required.
-   Use when streaming bandwidth is a constraint (especially with TensoRF-CP).
-   Use when dealing with fast-moving objects where deformation modeling is critical (TensoRF-VM).

**When Not to Use:**
-   Do not use for high FPS multi-camera datasets if deformation modeling is not needed for temporal interpolation, as it adds unnecessary complexity [S1].
-   Do not use if sharpness is prioritized over accuracy and computational efficiency (HyperNeRF may be preferred for SSIM) [S1].

## Background
NeRFPlayer was introduced in the paper "NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields" [S1]. It builds on the concept of decomposed neural radiance fields to handle dynamic scenes by separating static and dynamic components. The method addresses limitations of previous dynamic NeRF methods, such as high computational cost and artifacts in dynamic regions, by introducing a decomposition field that probabilistically assigns points to stationary, deforming, or new categories [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
