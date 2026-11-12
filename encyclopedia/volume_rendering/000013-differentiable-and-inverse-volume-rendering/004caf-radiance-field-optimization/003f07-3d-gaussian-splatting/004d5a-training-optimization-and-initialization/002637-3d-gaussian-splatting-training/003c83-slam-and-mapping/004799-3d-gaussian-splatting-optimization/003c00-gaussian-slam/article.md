# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that represents the scene as a collection of 3D Gaussians and employs sub-map fusion to maintain consistency [S2, unofficial]. It distinguishes itself from sibling methods like SplaTAM, which uses online tracking and densification, by decoupling the mapping process into localized sub-maps that are fused, allowing for more stable long-term reconstruction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of RGB-D images (or RGB with estimated depth) and their corresponding initial camera poses.
*   **Outputs:** A global map of 3D Gaussians, refined camera poses, and the associated sub-map structures.

**Data Structures:**
The scene is represented as a collection of 3D Gaussians $\mathcal{G}$. Each Gaussian $i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$.
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Color $c_i$ (view-dependent or independent).

The method maintains a set of **sub-maps**, each containing a subset of Gaussians associated with a specific keyframe or local region.

**Optimization Objective:**
The total loss $L$ combines photometric, depth, and regularization terms:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where:
*   $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures color difference.
*   $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures depth difference.
*   $L_{reg}$ includes scale regularization to encourage isotropic Gaussians, stabilizing localization and mapping [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the first sub-map using Gaussians derived from the first frame's depth or point cloud.
2.  **Tracking:** For each new frame, estimate the camera pose $\mathbf{T}_t$ relative to the current global map (or active sub-map) using photometric optimization or ICP.
3.  **Keyframe Selection:** Determine if the current frame is sufficiently different from existing keyframes to warrant a new sub-map.
4.  **Sub-map Creation/Fusion:**
    *   If a new keyframe is selected, create a new local sub-map.
    *   Fuse the new sub-map with the existing global map, resolving overlaps and ensuring cross-view coherence [S2, unofficial].
5.  **Densification:** Add new Gaussians in areas with high reconstruction error or poor coverage. Prune redundant or low-opacity Gaussians.
6.  **Optimization:** Update the parameters of the Gaussians (position, scale, rotation, opacity, color) and camera poses using gradient descent to minimize the loss $L$. Specifically, apply extra scale regularization to prevent degenerate anisotropic Gaussians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_slam(frames):
    global_map = SubMap()
    keyframes = []
    
    for frame in frames:
        # 1. Track camera pose
        pose = track_pose(frame, global_map)
        
        # 2. Check if keyframe
        if is_new_keyframe(frame, keyframes):
            # 3. Create local sub-map
            local_submap = create_submap(frame, pose)
            
            # 4. Fuse sub-map into global map
            global_map = fuse_submaps(global_map, local_submap)
            keyframes.append(frame)
            
        # 5. Densify and Optimize
        densify_gaussians(global_map, frame, pose)
        optimize_gaussians(global_map, frame, pose)
        
    return global_map, keyframes

def optimize_gaussians(map, frame, pose):
    # Render current view
    rendered_color, rendered_depth = render(map, pose)
    
    # Compute losses
    l_photo = l1_loss(rendered_color, frame.color)
    l_depth = l1_loss(rendered_depth, frame.depth)
    
    # Scale regularization for isotropy
    l_reg = sum(gaussian.scale_variance() for gaussian in map.gaussians)
    
    # Total loss
    loss = l_photo + lambda_depth * l_depth + lambda_reg * l_reg
    
    # Gradient descent update
    update_parameters(map.gaussians, pose, loss)
```

**Parameters:**
*   $\lambda_{depth}$: Weight for depth loss. Not given in the sources.
*   $\lambda_{reg}$: Weight for scale regularization. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degeneracy:** Gaussians may become overly anisotropic, leading to poor reconstruction. Mitigated by scale regularization [S1].
*   **Map Drift:** Accumulation of pose errors over time. Sub-map fusion helps mitigate this by maintaining local consistency [S2, unofficial].
*   **Duplicate Densification:** Avoiding redundant Gaussians in well-reconstructed areas. Requires careful densification criteria [S1].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method with online optimization using Gaussian Splatting [S2, unofficial]. It differs from Gaussian-SLAM by not requiring depth inputs and using online densification rather than keyframe-based sub-map fusion. Choose it when depth sensors are unavailable.

### Vox-Fusion
A hybrid SLAM system combining a voxel-based neural implicit surface representation with an octree structure for dynamic scene expansion [S3]. It encodes and optimizes each scene voxel implicitly, supporting incremental mapping without prior environment knowledge and leveraging a multi-process framework for speed [S3]. Choose it for dynamic scenes requiring implicit surface representations.

### Neural SLAM Pipeline
An end-to-end simultaneous localization and mapping approach that produces dense, photorealistic maps while tracking camera pose, replacing sparse classical SLAM outputs [S4, unofficial]. It tracks the current frame against the existing map, updates the map with new views by inserting/refining Gaussians, and optionally performs loop closure [S4, unofficial]. It is denser and more photorealistic than classical SLAM but requires heavier compute [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and frames. Sub-map fusion adds overhead compared to purely online methods but improves long-term stability.
*   **Memory Complexity:** Scales with the number of Gaussians in the global map. Sub-maps allow for localized memory management.
*   **Measured Performance:** On the Replica dataset, Gaussian-SLAM achieves PSNR 37.50, SSIM 0.96, and LPIPS 0.07 [S1].
*   **Strengths:** Stable long-term mapping via sub-map fusion; isotropic Gaussians improve localization stability.
*   **Limitations:** Keyframe-based approach may be less responsive to rapid scene changes compared to online methods; sub-map fusion adds computational complexity.
*   **When to use:** Long-duration SLAM tasks requiring consistent global maps.
*   **When not to use:** Real-time applications with strict latency constraints where online methods like SplaTAM are preferred.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It builds on the principles of 3D Gaussian Splatting and adapts them for SLAM by introducing a keyframe-based approach with sub-map fusion, addressing issues of map drift and degeneracy found in earlier neural SLAM methods.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
