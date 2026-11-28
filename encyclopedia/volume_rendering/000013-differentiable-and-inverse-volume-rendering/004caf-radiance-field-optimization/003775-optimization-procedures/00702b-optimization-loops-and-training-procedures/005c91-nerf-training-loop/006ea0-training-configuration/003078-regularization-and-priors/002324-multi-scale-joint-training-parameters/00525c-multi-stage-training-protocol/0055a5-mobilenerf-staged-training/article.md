# MobileNeRF Staged Training

Also called: MobileNeRF Training Stages

## Overview
MobileNeRF Staged Training is a three-stage optimization protocol that sequentially trains the geometry, acceleration structures, and appearance networks of the MobileNeRF representation [S1]. It decouples joint optimization into phases with adaptive ray marching to stabilize convergence [S1].

## How it works
The method optimizes the MobileNeRF representation through three distinct training stages, interleaved with specific acceleration grid optimization steps [S1]. The representation consists of a regular-grid mesh $\mathbf{M}$ defined by vertices $\mathbf{V}$, an acceleration grid $\mathbf{G}$, and three Multi-Layer Perceptrons (MLPs): an opacity MLP with parameters $\theta_{OpacityMLP}$ (also denoted $\theta_A$), a features MLP with parameters $\theta_{FeaturesMLP}$, and a shader MLP with parameters $\theta_{ShaderMLP}$ [S1].

### Inputs and Outputs
*   **Inputs:** Training images, camera poses, initial vertex positions $\mathbf{V}$, initial acceleration grid $\mathbf{G}$, initial MLP parameters, and scene type (synthetic 360°, forward-facing, or unbounded 360°) [S1].
*   **Outputs:** Optimized vertices $\mathbf{V}$, acceleration grid $\mathbf{G}$, and MLP parameters $\theta_{OpacityMLP}$, $\theta_{FeaturesMLP}$, $\theta_{ShaderMLP}$ [S1].

### Data Structures
*   **Vertices ($\mathbf{V}$):** Defines the geometry of the regular-grid mesh $\mathbf{M}$ [S1].
*   **Acceleration Grid ($\mathbf{G}$):** A 3D grid of size $P \times P \times P$ used to prune voxels during ray marching [S1].
*   **MLPs:** Three separate networks for opacity, features, and shading [S1].

### Parameters
*   $w_d$: Weight for the distance loss $\mathcal{L}_{dist}$. Synthetic 360° scenes: $0.0$ [S1]. Forward-facing scenes: $0.01$ [S1]. Unbounded 360° scenes: $0.001$ [S1].
*   $w_{g1}, w_{g2}$: Weights for sparse and smooth grid losses. Value is $10^{-5}$ for both [S1].
*   $\tau_G$: Threshold for pruning voxels in the acceleration grid. Value is $0.1$ [S1].
*   $P$: Grid dimension for the $P \times P \times P$ acceleration grid and mesh intersections [S1].

### Numbered Steps

1.  **Initialization:** Initialize vertices $\mathbf{V}$, acceleration grid $\mathbf{G}$, and MLP parameters $\theta_{OpacityMLP}$, $\theta_{FeaturesMLP}$, $\theta_{ShaderMLP}$ using standard initialization [G].

2.  **Stage 1 (First Quarter):** Optimize $\mathbf{V}$, $\theta_{OpacityMLP}$, $\theta_{FeaturesMLP}$, and $\theta_{ShaderMLP}$ to minimize $\mathcal{L}_{Color} + w_d \mathcal{L}_{dist} + \mathcal{L}_{v}$ [S1]. Concurrently optimize $\mathbf{G}$ to minimize $\mathcal{L}_{Grid}^{bnd} + w_{g1} \mathcal{L}_{Grid}^{sparse} + w_{g2} \mathcal{L}_{Grid}^{smooth}$ [S1]. During ray marching, $\mathbf{G}$ is inaccurate, so the algorithm keeps all $3P$ intersected voxels and intersection points regardless of $\tau_G$ [S1].

3.  **Stage 2 (Next Quarter):** Optimize $\mathbf{V}$, $\theta_{OpacityMLP}$, $\theta_{FeaturesMLP}$, and $\theta_{ShaderMLP}$ to minimize $\mathcal{L}_{Color}^{stage2} + w_d \mathcal{L}_{dist} + \mathcal{L}_{v}$ [S1]. Optimize $\mathbf{G}$ using the same grid loss as Stage 1 [S1]. During ray marching, use $\mathbf{G}$ with threshold $\tau_G=0.1$ to prune empty voxels, keeping at most $3P/2$ non-empty voxels and the $3P/2$ intersection points closest to the camera [S1]. Double the training batch size [S1].

4.  **Stage 3 (Rest of Training):** When the loss converges, fix $\mathbf{V}$, $\theta_{OpacityMLP}$ ($\theta_A$), and $\mathbf{G}$ [S1]. Optimize only $\theta_{FeaturesMLP}$ and $\theta_{ShaderMLP}$ to minimize $\mathcal{L}_{Color}^{bin}$ [S1]. During ray marching, keep at most $3P/4$ non-empty voxels and intersection points [S1]. Double the training batch size again [S1]. For unbounded 360° scenes, compute intersections for concentric boxes and keep all of them throughout the process [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mobile_nrf_staged_training(images, poses, P, tau_G, scene_type):
    # Initialize parameters using standard initialization (e.g., small random values) [G]
    V = ...
    G = ...
    theta_OpacityMLP = ...
    theta_FeaturesMLP = ...
    theta_ShaderMLP = ...

    # Set distance loss weight based on scene type [S1]
    if scene_type == "synthetic_360":
        w_d = 0.0
    elif scene_type == "forward_facing":
        w_d = 0.01
    elif scene_type == "unbounded_360":
        w_d = 0.001
    else:
        w_d = 0.0

    w_g1 = 1e-5
    w_g2 = 1e-5

    total_iterations = ... # Not given in the sources.
    quarter = total_iterations // 4
    batch_size = ... # Not given in the sources.

    for iteration in range(total_iterations):
        # Determine ray marching strategy based on training progress [S1]
        if iteration < quarter:
            # Stage 1: First quarter. G is inaccurate.
            max_voxels = 3 * P
            max_intersections = 3 * P
            use_grid_pruning = False
        elif iteration < 2 * quarter:
            # Stage 2: Next quarter. Use G to prune.
            max_voxels = 3 * P // 2
            max_intersections = 3 * P // 2
            use_grid_pruning = True
            batch_size = batch_size * 2
        else:
            # Stage 3: Rest of training.
            max_voxels = 3 * P // 4
            max_intersections = 3 * P // 4
            use_grid_pruning = True
            batch_size = batch_size * 2

        # Unbounded scenes exception: keep all intersections [S1]
        if scene_type == "unbounded_360":
            max_voxels = None
            max_intersections = None
            use_grid_pruning = False

        # Sample rays from images and poses for the current batch [G]
        rays = ... # Standard ray sampling from camera poses
        # Render colors and depths using MobileNeRF forward pass [G]
        colors, depths = ... # Standard volume rendering or rasterization forward pass

        # Compute losses based on stage [S1]
        if iteration < 2 * quarter:
            if iteration < quarter:
                # Stage 1 loss [S1]
                loss = ... # Standard color loss (e.g., MSE) + w_d * distance loss + volume loss
            else:
                # Stage 2 loss [S1]
                loss = ... # Modified color loss + w_d * distance loss + volume loss

            # Update all parameters using gradient descent [G]
            V, theta_OpacityMLP, theta_FeaturesMLP, theta_ShaderMLP = ... # Standard optimizer step

            # Optimize Grid G [S1]
            loss_grid = ... # Boundary loss + w_g1 * sparse loss + w_g2 * smooth loss
            G = ... # Standard optimizer step for G
        else:
            # Stage 3: Post-convergence. Fix V, theta_OpacityMLP, G [S1]
            loss = ... # Binarized color loss
            # Update only features and shader MLPs using gradient descent [G]
            theta_FeaturesMLP, theta_ShaderMLP = ... # Standard optimizer step

    return V, G, theta_OpacityMLP, theta_FeaturesMLP, theta_ShaderMLP
```

### Edge Cases and Pitfalls
*   **Inaccurate Grid Early On:** In the first quarter, the acceleration grid $\mathbf{G}$ is not reliable [S1]. Failing to disable pruning or keep all $3P$ intersections will lead to missing geometry and poor convergence [S1].
*   **Batch Size Scaling:** The batch size must be doubled when the number of kept intersections is halved to maintain stable gradient statistics [S1].
*   **Unbounded Scenes:** Standard pruning logic fails for concentric boxes in unbounded 360° scenes; all intersections must be retained [S1].
*   **Texture Constraints:** Mobile hardware requires texture sizes to be powers of 2 and at most $4096 \times 4096$ [S1]. Large texture images must be split into multiple tiles [S1].

## Subtopics

### MobileNeRF Training Objective
The core training objective minimizes $\mathcal{L}_{Color} + w_d \mathcal{L}_{dist} + \mathcal{L}_{v}$ with respect to vertices $\mathbf{V}$, opacity MLP parameters $\theta_{OpacityMLP}$, features MLP parameters $\theta_{FeaturesMLP}$, and shader MLP parameters $\theta_{ShaderMLP}$ [S1]. This objective is used in the first stage [S1]. The distance loss weight $w_d$ varies by scene type, and the loss terms include color, distance, and volume components [S1].

## Operating characteristics
*   **Time Complexity:** The regular-grid mesh $\mathbf{M}$ provides an efficient way for computing intersections between a ray and the mesh of size $P \times P \times P$ in $O(P)$ complexity [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Stabilizes training by decoupling geometry/acceleration from appearance refinement [S1]. Adapts ray marching efficiency based on training progress [S1].
*   **Limitations:** Requires careful tuning of stage transitions and batch size scaling [S1]. The acceleration grid optimization adds complexity [S1].
*   **When to Use:** When training MobileNeRF and standard joint optimization fails to converge due to geometry/acceleration instability [S1].

## Background
MobileNeRF was introduced in the MobileNeRF paper [S1]. The staged training protocol is part of its optimization procedure [S1].

## Sources
[S1] MobileNeRF: Exploiting the Polygon Rasterization Pipeline for Efficient Neural Field Rendering on Mobile Architectures, https://doi.org/10.1109/cvpr52729.2023.01590, fetched 2026-10-26
