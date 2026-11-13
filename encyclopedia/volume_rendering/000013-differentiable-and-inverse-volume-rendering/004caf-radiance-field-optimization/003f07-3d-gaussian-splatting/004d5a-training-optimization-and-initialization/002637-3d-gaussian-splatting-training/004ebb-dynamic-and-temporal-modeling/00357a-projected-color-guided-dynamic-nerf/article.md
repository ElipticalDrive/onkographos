# Projected Color-Guided Dynamic NeRF

Also called: OD-NeRF

## Overview
Projected Color-Guided Dynamic NeRF (OD-NeRF) is an on-the-fly training method for dynamic scenes that replaces temporal deformation fields with a NeRF model conditioned on multi-view projected colors [S1]. By leveraging the invariance of projected color for corresponding 3D points across consecutive frames, the model implicitly tracks correspondence, allowing it to utilize radiance fields from previous frames to render the current frame with minimal optimization iterations [S1]. Unlike sibling methods that use explicit temporal inputs or deformation networks which extrapolate poorly to unseen times, OD-NeRF requires the minimum number of optimization iterations per new frame [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of multi-view images representing a dynamic scene. For synthetic datasets, the method builds upon the TiNeuVox architecture; for real-world datasets, it builds upon InstantNGP (via NerfAcc) [S1].
*   **Outputs:** A radiance field representation that can render novel views for any frame in the sequence, updated incrementally frame-by-frame [S1].

**Data Structures:**
1.  **Occupancy Grid:** A spatial structure used for efficient sampling. In the synthetic implementation, this is a transited and updated occupancy grid derived from the previous frame [S1]. In the real-world implementation, this is the hash voxel grid from InstantNGP [S1].
2.  **NeRF MLP:** A Multi-Layer Perceptron that maps spatial features and projected color statistics to density and color [S1].
3.  **Spatial Feature:** The feature vector queried from the voxel grid (multi-scale for synthetic, hash for real-world) at a sampled point's location [S1].

**Parameters:**
*   **$\sigma_{min}$:** Density threshold for rejection sampling. Not given in the sources.
*   **$R$:** Fixed interval for rejection sampling. Not given in the sources.
*   **Warm-up Duration:** 2 seconds for the first frame in synthetic datasets to compensate for the slower convergence of hashed voxels compared to multi-scale voxels [S1].
*   **Optimization Schedule:** The optimization process intensity is scaled from 1 (at frame 1) to 0.05 (at frame 10) to aid convergence [S1].

**Numbered Steps:**

1.  **Initialization:** For the first frame, initialize the radiance field. If using the synthetic dataset configuration, apply a 2-second warm-up period [S1].
2.  **Sampling:** Sample points along rays.
    *   *Synthetic:* Use rejection sampling based on the transited occupancy grid from the previous frame. A sample $x_i$ is rejected if its occupancy value $\sigma_{occ}(x_i)$ is below $\sigma_{min}$ AND it is not on a fixed interval $R$ [S1]:
        $$ \text{reject } x_i \text{ if } (\sigma_{occ}(x_i) < \sigma_{min}) \land \neg(i \mod R \equiv R//2) $$
    *   *Real-world:* Use the standard sampling strategy of InstantNGP [S1].
3.  **Projected Color Computation:** For each sampled point, compute the mean and variance of its projected colors from the training views. This provides the "projected color statistics" [S1].
4.  **Feature Concatenation:** Concatenate the mean and variance of the projected colors with the spatial feature queried from the voxel grid at the point's location [S1].
5.  **NeRF Query:** Input the concatenated feature vector into the NeRF MLP to predict density and color [S1].
6.  **Rendering:** Render the image using standard volume rendering integration [S1].
7.  **Optimization:** Minimize the photometric loss between the rendered image and the ground truth image.
    *   *Real-world Specific:* If using real-world datasets, add a depth smoothness regularization loss $L_{depth}$ calculated over any $3 \times 3$ patch of rays:
        $$ L_{depth} = \frac{\text{std}(d_{far} / d_{3 \times 3})}{\text{std}(c_{3 \times 3})} $$
        where $d_{far}$ is the depth of the farthest sample, $d_{3 \times 3}$ is the mean depth of the $3 \times 3$ patch, and $c_{3 \times 3}$ is the mean color of the patch [S1].
8.  **Transition and Update:** Update the occupancy grid for the next frame based on the current frame's results to guide sampling for the subsequent frame [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_od_nerf_frame_by_frame(video_sequence, config):
    """
    Trains OD-NeRF on a sequence of frames.
    config: dict containing 'dataset_type' ('synthetic' or 'real'), 
            'sigma_min', 'R', 'warmup_duration'.
    """
    # Initialize model and occupancy grid
    if config['dataset_type'] == 'synthetic':
        model = TiNeuVoxModel()
        occupancy_grid = initialize_occupancy_grid()
    else:
        model = InstantNGPModel()
        occupancy_grid = model.hash_voxel_grid

    for frame_idx, frame_images in enumerate(video_sequence):
        # Warm-up for first frame in synthetic datasets
        if frame_idx == 0 and config['dataset_type'] == 'synthetic':
            warmup(duration=config['warmup_duration'])

        # Optimization schedule scaling
        opt_scale = 1.0 - (frame_idx / 10.0) * 0.95 # From 1.0 to 0.05
        
        # Render and Optimize
        rendered_images = []
        for camera in frame_images.cameras:
            rays = generate_rays(camera)
            samples = []
            
            for ray in rays:
                # Sampling step
                if config['dataset_type'] == 'synthetic':
                    # Rejection sampling based on occupancy grid
                    uniform_samples = sample_uniform(ray)
                    for i, x_i in enumerate(uniform_samples):
                        sigma_occ = occupancy_grid.query(x_i)
                        if not (sigma_occ < config['sigma_min'] and not (i % config['R'] == config['R'] // 2)):
                            samples.append(x_i)
                else:
                    # InstantNGP sampling
                    samples = sample_instantngp(ray, occupancy_grid)

                # Projected Color Computation
                # Compute mean and variance of projected colors for the sampled point
                # from all training views. This assumes access to previous frame's 
                # rendered colors or a cache of projected colors.
                projected_colors = get_projected_colors(samples, frame_images)
                mean_color, var_color = compute_stats(projected_colors)

                # Feature Concatenation
                spatial_feature = query_spatial_feature(samples, occupancy_grid)
                input_feature = concatenate(spatial_feature, mean_color, var_color)

                # NeRF Query
                density, color = model.forward(input_feature)
                
                # Volume Rendering
                rendered_color = volume_render(ray, samples, density, color)
                rendered_images.append(rendered_color)

            # Loss Calculation
            loss = photometric_loss(rendered_images, frame_images.gt_images)
            
            # Real-world specific depth regularization
            if config['dataset_type'] == 'real':
                depth_loss = compute_depth_smoothness_loss(rendered_images, frame_images)
                loss += depth_loss

            # Backpropagation
            model.backward(loss * opt_scale)

        # Transition and Update Occupancy Grid
        update_occupancy_grid(occupancy_grid, rendered_images, frame_images)
```

**Edge Cases and Pitfalls:**
*   **Specular Surfaces:** Specular highlights violate the assumption of projected color invariance, leading to artifacts [S1].
*   **Occlusions:** Occluded points also violate relative invariance, causing artifacts [S1]. Filtering these outliers may incur significant computational costs [S1].
*   **Convergence Speed:** Hashed voxels converge slower than multi-scale voxels, necessitating the warm-up period for synthetic datasets [S1].
*   **Planar Backgrounds:** Real-world datasets with large planar backgrounds (e.g., walls, tables) may fail to converge well even for the first frame, requiring the depth smoothness regularization [S1].

## Operating characteristics
*   **Speed:** Achieves an interactive speed of 6 FPS for both training and rendering on synthetic dynamic scenes on-the-fly [S1]. It provides a significant speed-up compared to state-of-the-art methods on real-world dynamic scenes [S1].
*   **Optimization Efficiency:** Requires the minimum number of optimization iterations for each new frame compared to models with temporal input dimensions or deformation fields [S1].
*   **Extrapolation:** Exhibits excellent temporal extrapolation capability, unlike deformation-based models that extrapolate poorly to unseen times [S1].
*   **Memory:** Uses hash voxel grids (InstantNGP) or multi-scale voxel grids (TiNeuVox), which are generally more memory-efficient than dense 3D grids [S1].
*   **Limitations:** Performance degrades on scenes with strong specularities or significant occlusions due to the reliance on projected color invariance [S1].

## Background
OD-NeRF was introduced in the paper "OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields" [S1]. It addresses the challenge of efficient on-the-fly training, where the radiance field is trained and rendered frame-by-frame, effectively utilizing the radiance field estimated from previous frames [S1]. The method was implemented on top of TiNeuVox for synthetic datasets and InstantNGP (via NerfAcc) for real-world datasets [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
