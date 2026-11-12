# Plenoxels Training

Also called: Plenoxels Training Implementation

## Overview
Plenoxels Training is the optimization procedure that produces the final sparse voxel grid representation from input images. It builds on [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md) by executing the gradient-based updates and persisting the learned opacity ($\sigma$) and spherical harmonic (SH) coefficients. It is distinguished from its siblings by its specific implementation artifacts, such as the fixed final resolution and training duration, and by its role as the foundational data source for downstream perception tasks like 3D classification [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of multi-view images with known camera poses.
*   **Output:** A saved voxel grid file containing the optimized sparse voxels. Each voxel stores coordinates, density values, and SH coefficients [S1, unofficial][S2].

**Data Structures:**
The output is a sparse voxel grid. The structure consists of two primary components [S2]:
1.  **Coordinates:** The 3D spatial positions of the active voxels.
2.  **Features:** The radiance field parameters for each active voxel, specifically the density ($\sigma$) and the spherical harmonic coefficients representing view-dependent color [S1, unofficial][S2].

**Steps:**
1.  **Initialization:** Initialize the voxel grid parameters ($\sigma$ and SH coefficients) based on the configuration defined in [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md).
2.  **Rendering:** Render images from the training views using differentiable volume rendering. This involves ray marching through the sparse voxel grid, accumulating opacity and color based on the stored features [S1, unofficial].
3.  **Loss Computation:** Calculate the difference between rendered images and ground truth images (typically using an $L_2$ or photometric loss).
4.  **Optimization:** Update the voxel parameters using a gradient-based optimizer (e.g., Adam, as implied by the parent configuration) to minimize the loss [S1, unofficial].
5.  **Iteration:** Repeat steps 2-4 for the specified number of iterations or until convergence.
6.  **Saving:** Upon completion, save the final optimized voxel grid to disk [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Plenoxels Training execution
# Based on [S1, unofficial] main.py implementation

def train_plenoxels(dataset, config):
    # 1. Initialize the voxel grid with sparse voxels
    # Coordinates and initial features (sigma, SH coefficients) are set up
    voxel_grid = initialize_sparse_voxel_grid(config.initial_resolution)
    
    # 2. Setup optimizer based on configuration
    optimizer = setup_optimizer(voxel_grid.parameters(), config.learning_rate)
    
    # 3. Training loop
    for iteration in range(config.total_iterations):
        # Sample a batch of camera poses and corresponding ground truth images
        images_gt, cameras = dataset.sample_batch(config.batch_size)
        
        # Render images from current voxel grid state
        # This involves ray marching and accumulating color/opacity
        images_pred = render_voxel_grid(voxel_grid, cameras)
        
        # Compute loss (e.g., MSE between predicted and ground truth)
        loss = compute_loss(images_pred, images_gt)
        
        # Backpropagate and update voxel parameters
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Optional: Log metrics or save checkpoints
        if iteration % config.save_interval == 0:
            save_checkpoint(voxel_grid, iteration)

    # 4. Save final result
    # The final grid is saved as a 128x128x128 resolution structure
    save_final_grid(voxel_grid, path="./saved_grids/")

def render_voxel_grid(voxel_grid, cameras):
    # For each camera, cast rays through the scene
    rendered_images = []
    for camera in cameras:
        rays = generate_rays(camera)
        # March along rays, querying sparse voxels for sigma and SH coefficients
        # Accumulate color and opacity using volume rendering integral
        image = volume_render_rays(rays, voxel_grid)
        rendered_images.append(image)
    return rendered_images
```

**Parameters and Typical Values:**
*   **Final Resolution:** The output voxel grid is saved at a resolution of $128 \times 128 \times 128$ [S1, unofficial].
*   **Hardware:** Training is benchmarked on a single GPU, specifically an NVIDIA RTX 3090 [S1, unofficial][S2].
*   **Duration:** The training process takes approximately 30 minutes on a single GPU [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Sparse Voxel Management:** The method relies on sparse voxel structures. Inefficient culling or management of inactive voxels can lead to memory overhead or rendering artifacts [S2].
*   **Background Handling:** Plenoxels use both sparse voxels and lumispheres for foreground and background rendering. Incorrect handling of the background component can lead to artifacts in complex scenes [S2].

## Subtopics

### 3D Classification with Spatially Sparse Convolutions
This variant uses the trained Plenoxels grid as input for 3D classification tasks. It employs 3D convolutional networks (e.g., ResNet variants) implemented with spatially sparse convolutional layers [S2]. The input to the network consists of the sparse voxel coordinates and their features (SH coefficients and density values) [S2]. Training uses the SGD optimizer with an initial learning rate of 0.1, cosine annealing for 100K iterations, and a batch size of 16 on a single RTX 3090 GPU [S2]. Data augmentation includes geometric transformations (random rotation, coordinate dropout, horizontal flip, uniform translation, scaling) and feature-level jittering [S2]. A unique augmentation involves substituting the background of a scene with backgrounds from other scenes while preserving the foreground [S2].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 30 minutes on a single GPU [S1, unofficial].
*   **Memory Complexity:** Depends on the sparsity of the voxel grid and the resolution ($128^3$ final grid) [S1, unofficial].
*   **Strengths:** Fast training time; produces a compact, sparse representation suitable for downstream tasks like classification [S1, unofficial][S2].
*   **Limitations:** Performance is tied to the specific hardware (RTX 3090) and configuration; sparse voxel rendering can be complex to implement efficiently [S1, unofficial][S2].
*   **When to Use:** When a fast, compact radiance field representation is needed for both rendering and perception tasks.
*   **When Not to Use:** When higher resolution or more complex background modeling is required without the sparse voxel constraints.

## Background
Plenoxels Training was introduced as part of the Plenoxels framework, which aims to represent radiance fields using sparse voxels and spherical harmonics to enable fast rendering and training [S1, unofficial]. The implementation details, including the 30-minute training time on a single GPU, were documented in the associated repository [S1, unofficial]. The method has been extended to support perception tasks, such as 3D classification, by leveraging the sparse voxel features directly in convolutional networks [S2].

## Sources
[S1] nissmar/Radiance-Fields, https://github.com/nissmar/Radiance-Fields, fetched 2026-10-27 (unofficial)
[S2] PeRFception: Perception using Radiance Fields, https://doi.org/10.48550/arxiv.2208.11537, fetched 2026-10-26
