# Training Loop Setup

Also called: Training Step Execution

## Overview
The Training Loop Setup defines the execution of a single optimization iteration for a radiance field, encompassing data loading, forward rendering, loss computation, and parameter updates [S1, unofficial]. It also covers the initialization of the training environment, including random seed fixing, data loading, log directory creation, and checkpoint reloading logic [S3, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a batch of rays defined by pixel coordinates and camera parameters, along with the current state of the radiance field representation and optimizer states [S1, unofficial]. It outputs the updated model parameters and optimizer states after one gradient step [S1, unofficial].

**Data Structures:**
The `msplat_train_step` variant uses a dense, explicit tensor layout [S1, unofficial]:
- `numPoints`: Integer count of points/gaussians.
- `means`: Float32 array of shape `(numPoints, 3)`.
- `scales`: Float32 array of shape `(numPoints, 3)`.
- `glob_scale`: Float32 scalar (default 1.0f).
- `quats`: Float32 array of shape `(numPoints, 4)` (quaternions).
- `viewMat`, `projViewMat`: Float32 matrices for camera projection.
- `fx`, `fy`, `cx`, `cy`: Float32 camera intrinsics.
- `height`, `width`: Integer image dimensions.
- `tileBounds`: Integer array defining tile boundaries for rasterization.
- `near plane`: Float32 (default 0.01f).
- `degree`, `degreesToUse`: Integers for spherical harmonic degrees.
- `cam_pos`: Float32 array `(3,)`.
- `featuresDc`: Float32 array of DC color features.
- `featuresRest`: Float32 array of SH rest coefficients.
- `opacities`: Float32 array of per-point opacities.
- `backgroundColor`: Float32 array `(3,)`.
- `gt`: Float32 array `(batch_size, height, width, 3)` ground truth images.
- `SSIM window`: Float32 array for structural similarity calculation.
- `ssimWeight`: Float32 scalar.
- `lossInvN`: Float32 scalar, normalized as $1.0f / (height \times width \times 3)$ [S1, unofficial].
- `featuresRest stride`: Integer stride for feature memory layout.
- `N_ADAM_GROUPS`: Integer for optimizer partitioning.
- `Adam params`: Float32 arrays for `exp_avg`, `exp_avg_sq`, `scaled lr`, `sqrt(bc2)`.
- `Adam beta1`, `beta2`, `eps`: Float32 scalars.
- `visCounts`, `xysGradNorm`, `max2DSize`, `invMaxDim`: Float32/Integer auxiliary buffers for gradient normalization and visibility tracking. `invMaxDim` is calculated as $1.0f / \max(height, width)$ [S1, unofficial].

**Steps:**
1. **Environment Setup:** If `args.random_seed` is not None, set `np.random.seed` and `tf.compat.v1.set_random_seed` [S3, unofficial].
2. **Data Loading:** Load image data and transformation matrices from file paths, moving tensors to the specified device [S2, unofficial].
3. **Grid Initialization:** If using a grid-based representation, generate the initial grid using `generate_grid` with parameters `gridsize`, `points_distance`, `info_size=4`, and the target device [S2, unofficial].
4. **Checkpoint Reloading:** If `args.ft_path` is provided, load specific weights; else load the latest `model_*.npy` checkpoint. Update the start step: `start = int(ft_weights[-10:-4]) + 1` [S3, unofficial]. If `model_fine` exists, load corresponding fine weights [S3, unofficial].
5. **Forward Pass:** Render the batch of rays using the current model parameters (`means`, `scales`, `quats`, `features`, etc.) and camera matrices (`viewMat`, `projViewMat`) to produce predicted pixel colors [S1, unofficial].
6. **Loss Computation:** Calculate the loss between the rendered output and the ground truth (`gt`). The loss is normalized by `lossInvN` [S1, unofficial]. Structural Similarity (SSIM) may be included using `ssimWeight` and the `SSIM window` [S1, unofficial].
7. **Backward Pass:** Compute gradients with respect to the trainable parameters [S1, unofficial].
8. **Optimizer Update:** Update the model parameters and optimizer states (e.g., Adam moments) using the computed gradients and learning rate [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_loop_setup_step(
    numPoints, means, scales, glob_scale, quats, viewMat, projViewMat,
    fx, fy, cx, cy, height, width, tileBounds, near_plane, degree,
    degreesToUse, cam_pos, featuresDc, featuresRest, opacities,
    backgroundColor, gt, ssim_window, ssimWeight, lossInvN,
    featuresRest_stride, N_ADAM_GROUPS, adam_exp_avg, adam_exp_avg_sq,
    adam_scaled_lr, adam_sqrt_bc2, adam_beta1, adam_beta2, adam_eps,
    visCounts, xysGradNorm, max2DSize, invMaxDim
):
    # 1. Forward Rendering: [G] Standard differentiable rasterization projects 3D Gaussians to 2D, sorts by depth, and alpha-composites.
    rendered_image = forward_render(
        means=means, scales=scales, glob_scale=glob_scale, quats=quats,
        viewMat=viewMat, projViewMat=projViewMat, fx=fx, fy=fy,
        cx=cx, cy=cy, height=height, width=width, tileBounds=tileBounds,
        near_plane=near_plane, degree=degree, degreesToUse=degreesToUse,
        cam_pos=cam_pos, featuresDc=featuresDc, featuresRest=featuresRest,
        opacities=opacities, backgroundColor=backgroundColor,
        featuresRest_stride=featuresRest_stride
    )

    # 2. Loss Computation: MSE normalized by lossInvN, optionally add SSIM
    mse_loss = sum(sum((rendered_image[i][j] - gt[i][j]) ** 2 for j in range(width)) for i in range(height)) / (height * width * 3) * lossInvN
    ssim_loss = 1.0 - compute_ssim(rendered_image, gt, ssim_window)
    total_loss = mse_loss + ssimWeight * ssim_loss

    # 3. Backward Pass: [G] Standard automatic differentiation computes gradients w.r.t. trainable parameters.
    # 4. Optimizer Update: Standard Adam step with adaptive scaling
    # [G] Standard Adam update: m = beta1*m + (1-beta1)*g; v = beta2*v + (1-beta2)*g^2; p = p - lr * m / (sqrt(v) + eps)
    update_adam_step(
        params=[means, scales, quats, featuresDc, featuresRest, opacities],
        grads=[p.grad for p in [means, scales, quats, featuresDc, featuresRest, opacities]],
        exp_avg=adam_exp_avg,
        exp_avg_sq=adam_exp_avg_sq,
        scaled_lr=adam_scaled_lr,
        sqrt_bc2=adam_sqrt_bc2,
        beta1=adam_beta1,
        beta2=adam_beta2,
        eps=adam_eps,
        visCounts=visCounts,
        xysGradNorm=xysGradNorm,
        max2DSize=max2DSize,
        invMaxDim=invMaxDim,
        N_ADAM_GROUPS=N_ADAM_GROUPS
    )

    return rendered_image, total_loss
```

**Edge Cases and Pitfalls:**
- **Precision:** Use float32 for standard operations; mixed precision may be used in variants like [Mixed Precision Training](2766-mixed-precision-training.md) [G].
- **Boundaries:** Ensure `tileBounds` and `near_plane` are correctly set to avoid rendering artifacts or division by zero in normalization [G].
- **Degenerate Input:** Handle cases where `numPoints` is zero or camera matrices are invalid [G].

## Subtopics

### [Training Ray Batching](2252-training-ray-batching.md)
Training Ray Batching is a data loading strategy for Neural Radiance Fields (NeRF) that selects a random subset of pixel coordinates (and their corresponding rays) from the full training image grid per optimization iteration, rather than rendering complete images. It reduces the GPU memory footprint during the backward pass of gradient descent by processing only $N_{rand}$ rays per step, which controls gradient noise and enables efficient training. It differs from [Training Loop for Refractive Index Field](4178-training-loop-for-refractive-index-field.md) and [Training Loop](4290-training-loop.md) by explicitly managing the pre-computation, shuffling, and batch-slicing of ray tensors across multiple training images, whereas those siblings focus on the network forward/backward passes or specific physical models.

### [Training Loop for Refractive Index Field](4178-training-loop-for-refractive-index-field.md)
This method defines the optimization loop for training a Multi-Layer Perceptron (MLP) to represent a 3D refractive index (IOR) field and associated emission field, using differentiable ray tracing. It distinguishes itself by combining Eikonal ray tracing with luminance integration, optimizing the IOR field via boundary regularization and luminance reconstruction loss, specifically implemented in JAX for multi-device sharding. It builds on [Training Loop Setup](1522-training-loop-setup.md) and [Training Ray Batching](2252-training-ray-batching.md).

### [Training Loop](4290-training-loop.md)
The Training Loop is the iterative optimization procedure that updates a radiance field (such as a NeRF or 3D Gaussian Splatting representation) by minimizing the difference between rendered images and ground-truth observations. It solves the problem of learning scene geometry and appearance from sparse 2D inputs by repeatedly sampling rays, rendering views, and backpropagating gradients. It sets itself apart from static rendering pipelines by integrating data fetching, loss computation, and optimizer steps into a single closed loop, building on [Training Loop Setup](1522-training-loop-setup.md) for configuration and [Training Ray Batching](2252-training-ray-batching.md) for efficient data handling.

### Training Pipeline Configuration
The main entry point for the training experiment, which loads configuration, sets up global parameters, loads the scene and sensors, renders reference images, and initializes the SDF for optimization [S4, unofficial].
- Load configuration from JSON file `config_file`. Update experiment parameters (name, scene, epochs, lr, sensor counts, batch size) and rendering parameters (integrator, resolution, spp, SDF mode) using command-line args or config defaults [S4, unofficial].
- Load the scene using `mi.load_file` with the specified scene XML and hide environment flag [S4, unofficial].
- Set sensor parameters (radius, height) based on scene name: 'hotdog' (h=0.5), 'lego' (h=0.35), 'chair' (h=0.6, r=1.5), else (h=0.4, r=1.3). Load sensors using `get_h2_sensors` [S4, unofficial].
- Initialize the SDF for optimization as a `SphereSDF` with specified resolution and radius. Enable gradients on the SDF grid tensor [S4, unofficial].
- Check if albedo, roughness, and metallic volumes are present in the scene parameters to determine optimization targets [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Highly dependent on GPU memory bandwidth and compute capabilities.
- **Strengths:** Modular design allows for easy integration of different loss functions and optimizers.
- **Limitations:** Memory usage can still be high for large scenes or high-resolution images if batching is not efficient.

## Background
The concept of a training loop setup is fundamental to differentiable rendering and neural radiance fields. The `msplat_train_step` implementation is associated with the Msplat project [S1, unofficial]. The `Plenoxels` project also utilizes similar training loop structures for voxel-based radiance fields [S2, unofficial]. The `relaxed-boundary` project demonstrates a configuration-based approach for SDF optimization [S4, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S4] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
