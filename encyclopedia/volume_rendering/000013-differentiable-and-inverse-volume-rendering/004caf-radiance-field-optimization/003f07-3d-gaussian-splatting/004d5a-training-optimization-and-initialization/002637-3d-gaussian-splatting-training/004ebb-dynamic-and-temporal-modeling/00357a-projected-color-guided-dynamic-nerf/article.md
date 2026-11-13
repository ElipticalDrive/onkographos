# Projected Color-Guided Dynamic NeRF

Also called: OD-NeRF

## Overview
Projected Color-Guided Dynamic NeRF (OD-NeRF) is an on-the-fly training method for dynamic scenes that replaces temporal deformation fields with a NeRF model conditioned on multi-view projected colors [S1]. By leveraging the observation that the projected color of a corresponding 3D point remains invariant across consecutive frames, the model implicitly tracks point correspondence, allowing it to effectively utilize radiance fields from previous frames to optimize the current frame [S1]. Unlike sibling methods such as [D-NeRF](8660-d-nerf.md) or [DaReNeRF](13782-darenerf.md) which rely on explicit temporal inputs or deformation networks that extrapolate poorly to unseen times, OD-NeRF requires the minimum number of optimization iterations per frame and exhibits superior temporal extrapolation capabilities [S1].

## How it works
The method operates in an on-the-fly setting where the radiance field is trained and rendered frame-by-frame [S1]. It builds on the spatial representation of [InstantNGP](link-to-instant-ngp.md) or [TiNeuVox](link-to-tineuvox.md) but modifies the input conditioning and sampling strategy to incorporate color statistics from previous views [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Current frame camera parameters (intrinsics, extrinsics).
    *   Current frame image.
    *   Radiance field and occupancy grid from the previous frame.
    *   Projected colors of sampled points from previous frames (stored in the occupancy grid or a buffer).
*   **Outputs:**
    *   Rendered image for the current frame.
    *   Updated radiance field (MLP weights and voxel grid) for the next frame.

### Data Structures
The core data structure is the radiance field representation, which varies by dataset type:

1.  **Synthetic Datasets (based on TiNeuVox):**
    *   **Voxel Grid:** Hash voxel grid (from InstantNGP) implemented with `tiny-cuda-nn` [S1]. This replaces the multi-scale voxel grid used in the original TiNeuVox to better capture details [S1].
    *   **MLP:** A Multi-Layer Perceptron (MLP) that takes spatial features and color statistics as input [S1].
    *   **Occupancy Grid:** Used for rejection sampling [S1].

2.  **Real-World Datasets (based on InstantNGP via NerfAcc):**
    *   **Voxel Grid:** Hash voxel grid [S1].
    *   **MLP:** Standard InstantNGP MLP [S1].
    *   **Occupancy Grid:** Used for sampling [S1].

### Numbered Steps

1.  **Transition and Update Occupancy Grid:**
    At the start of a new frame, the occupancy grid from the previous frame is transitioned to the current frame's coordinate system based on camera motion. This allows efficient sampling of points that are likely to be occupied in the current view [S1].

2.  **Ray Sampling with Rejection:**
    For each pixel, rays are cast. Points are sampled along the ray. To optimize computation, rejection sampling is applied using the transited occupancy grid $\sigma_{occ}$ [S1]. A sample $x_i$ at index $i$ along the ray is rejected if:
    $$ \text{reject } x_i \text{ if } (\sigma_{occ}(x_i) < \sigma_{min}) \land \neg(i \mod R \equiv R // 2) $$
    where $\sigma_{min}$ is a density threshold and $R$ is a fixed interval [S1]. This ensures that even in empty space, a sparse set of samples is retained to maintain continuity [S1].

3.  **Feature Query and Color Statistics Calculation:**
    For each valid sample point $x$:
    *   Query the spatial feature $f_{spatial}$ from the hash voxel grid [S1].
    *   Calculate the mean $\mu_c$ and variance $\sigma_c^2$ of the projected colors of corresponding points from previous frames. These statistics are derived from the stored projected colors associated with the spatial location or its neighborhood [S1].

4.  **NeRF MLP Input Construction:**
    Concatenate the color statistics with the spatial feature to form the input vector for the NeRF MLP:
    $$ \text{Input} = [f_{spatial}, \mu_c, \sigma_c^2] $$
    For synthetic datasets, the temporal deformation model and temporal information enhancement modules of the base TiNeuVox are removed [S1]. For real-world datasets, the spatial feature is queried from the hash voxel grid before concatenation [S1].

5.  **Density and Color Prediction:**
    Pass the input vector through the NeRF MLP to predict density $\sigma$ and color $c$ [S1].

6.  **Rendering:**
    Perform volume rendering using the predicted $\sigma$ and $c$ to generate the output image [S1].

7.  **Optimization:**
    Minimize the photometric loss between the rendered image and the ground truth image for the current frame [S1].
    *   **Synthetic Datasets:** No additional regularization is specified beyond the standard photometric loss [S1].
    *   **Real-World Datasets:** Apply a depth smoothness regularization loss $L_{depth}$ to handle large planar backgrounds with constant colors (e.g., walls, tables) [S1]. For any $3 \times 3$ patch of rays, the loss is calculated as:
        $$ L_{depth} = \frac{\text{std}(d_{far} / d_{3 \times 3})}{\text{std}(c_{3 \times 3})} $$
        where $d_{far}$ is the depth of the farthest sample, $d_{3 \times 3}$ is the set of depths in the patch, and $c_{3 \times 3}$ is the set of colors in the patch [S1].

8.  **Warm-up (Synthetic Datasets Only):**
    For the first frame of synthetic datasets, apply a 2-second warm-up duration to compensate for the slower convergence of the hashed voxel grid compared to the original multi-scale voxel [S1]. The optimization process starts with a higher learning rate or specific initialization that decays from 1 (at frame 1) to 0.05 (at frame 10) for better convergence [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_od_nerf_frame(frame_idx, camera_params, ground_truth_image, prev_occupancy_grid, prev_projected_colors):
    """
    Trains and renders one frame of OD-NeRF.
    
    Args:
        frame_idx: Current frame index.
        camera_params: Intrinsics and extrinsics for the current view.
        ground_truth_image: RGB image of the current frame.
        prev_occupancy_grid: Occupancy grid from the previous frame.
        prev_projected_colors: Projected colors from previous frames.
        
    Returns:
        rendered_image: The rendered image for the current frame.
        updated_mlp: The updated MLP weights.
        updated_occupancy_grid: The updated occupancy grid.
    """
    
    # Step 1: Transition Occupancy Grid
    # Move the occupancy grid from the previous frame to the current frame's coordinate system
    current_occupancy_grid = transition_occupancy_grid(prev_occupancy_grid, camera_params)
    
    # Step 2: Initialize MLP and Optimizer
    # If frame_idx == 0 and synthetic dataset, apply warm-up initialization
    mlp = initialize_mlp(dataset_type="synthetic" if is_synthetic else "real")
    optimizer = initialize_optimizer(mlp, warmup_duration=2.0 if (frame_idx == 0 and is_synthetic) else 0)
    
    # Step 3: Render Loop
    rendered_image = render_with_mlp(mlp, camera_params, current_occupancy_grid, prev_projected_colors)
    
    # Step 4: Compute Loss
    # Photometric loss (L1 or LPIPS)
    loss = compute_photometric_loss(rendered_image, ground_truth_image)
    
    # Depth smoothness regularization for real-world datasets
    if not is_synthetic:
        depth_loss = compute_depth_smoothness_loss(rendered_image, camera_params)
        loss = loss + depth_loss
        
    # Step 5: Backpropagate and Update
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # Step 6: Update Occupancy Grid for Next Frame
    # The occupancy grid is updated based on the current frame's rendering
    # and the transition logic for the next frame
    updated_occupancy_grid = update_occupancy_grid(current_occupancy_grid, rendered_image, camera_params)
    
    return rendered_image, mlp, updated_occupancy_grid

def render_with_mlp(mlp, camera_params, occupancy_grid, prev_projected_colors):
    """
    Renders an image using the NeRF MLP with projected color guidance.
    """
    height, width = camera_params.image_shape
    rendered_image = np.zeros((height, width, 3))
    
    for y in range(height):
        for x in range(width):
            ray = cast_ray(camera_params, x, y)
            
            # Sample points along the ray
            samples = sample_along_ray(ray, occupancy_grid)
            
            # Filter samples using rejection sampling
            valid_samples = []
            for i, sample in enumerate(samples):
                sigma_occ = query_occupancy(occupancy_grid, sample.position)
                # Rejection condition: (sigma_occ < sigma_min) AND NOT (i mod R == R // 2)
                if not (sigma_occ < sigma_min and not (i % R == R // 2)):
                    valid_samples.append(sample)
            
            # Query features and predict density/color
            densities = []
            colors = []
            
            for sample in valid_samples:
                # Query spatial feature from hash voxel grid
                f_spatial = query_hash_voxel_grid(sample.position)
                
                # Calculate mean and variance of projected colors from previous frames
                # This assumes prev_projected_colors are stored in a structure that allows lookup by position
                mu_c, sigma_c2 = compute_color_statistics(sample.position, prev_projected_colors)
                
                # Concatenate features
                input_vector = concatenate([f_spatial, mu_c, sigma_c2])
                
                # Predict density and color
                sigma, color = mlp.forward(input_vector)
                
                densities.append(sigma)
                colors.append(color)
            
            # Volume rendering
            alpha = 1.0 - torch.exp(-densities * ray_step_size)
            weights = alpha * torch.cumprod(1.0 - alpha + 1e-10, dim=0)
            rendered_color = torch.sum(weights * colors, dim=0)
            
            rendered_image[y, x] = rendered_color
            
    return rendered_image
```

### Parameters
*   **$\sigma_{min}$**: Density threshold for rejection sampling. Not given in the sources.
*   **$R$**: Fixed interval for rejection sampling. Not given in the sources.
*   **Warm-up duration**: 2 seconds for the first frame in synthetic datasets [S1].
*   **Learning rate decay**: From 1 (at frame 1) to 0.05 (at frame 10) for synthetic datasets [S1].

### Edge Cases and Pitfalls
*   **Specular Surfaces:** Specular surfaces violate the relative invariance of projected color, causing artifacts [S1].
*   **Occluded Points:** Occluded points also violate the invariance assumption, leading to artifacts [S1].
*   **Convergence on Real-World Data:** The NerfAcc implementation of InstantNGP may not converge well on forward-facing dynamic datasets with large planar backgrounds (e.g., walls, tables) due to constant colors, necessitating the depth smoothness regularization [S1].

## Operating characteristics
*   **Time Complexity:** The method achieves an interactive speed of 6 FPS for training and rendering on synthetic dynamic scenes on-the-fly [S1]. It requires the minimum number of optimization iterations for each new frame compared to methods with temporal inputs or deformation fields [S1].
*   **Memory Complexity:** Uses hash voxel grids (InstantNGP) which are memory-efficient compared to dense 3D grids. The occupancy grid is transited and updated, avoiding full recomputation [S1].
*   **Strengths:**
    *   Excellent temporal extrapolation capability [S1].
    *   Significantly faster on-the-fly training compared to state-of-the-art methods on real-world dynamic scenes [S1].
    *   Implicit point correspondence reduces the need for explicit deformation modeling [S1].
*   **Limitations:**
    *   Performance degrades on specular surfaces and occluded points [S1].
    *   Hashed voxels converge slower than multi-scale voxels, requiring warm-up strategies for synthetic datasets [S1].
*   **When to Use:** Suitable for on-the-fly training of dynamic scenes where computational efficiency is critical and scenes do not have significant specularities or complex occlusions.
*   **When Not to Use:** Not suitable for scenes with strong specular reflections or heavy occlusion without additional filtering mechanisms [S1].

## Background
OD-NeRF was introduced in the paper "OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields" [S1]. The method addresses the challenge of efficiently utilizing radiance fields estimated from previous frames for on-the-fly training, a setting where the scene is captured and rendered sequentially. It builds on the spatial representations of TiNeuVox and InstantNGP but replaces temporal conditioning with projected color guidance [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
