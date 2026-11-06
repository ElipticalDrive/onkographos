# Training Loop

Also called: NeRF Training Pipeline, Training Loop & Data Fetching

## Overview
The Training Loop orchestrates the iterative optimization of a radiance field (NeRF or Gaussian Splatting) by fetching data, rendering predictions, computing loss, and updating parameters. It differs from [Training Ray Batching](2252-training-ray-batching.md) by managing the high-level control flow, iterator reset logic, and dataset-specific initialization, while [Training Loop Setup](1522-training-loop-setup.md) handles the static configuration and model creation.

## How it works
**Inputs:** Training dataset (LLFF, Blender, LINEMOD, DeepVoxels), hyperparameters (e.g., `N_rand`, `no_batching`), and model configuration.
**Outputs:** Optimized radiance field parameters and rendered images.

**Data Structures:**
*   `train_loader`: PyTorch DataLoader or custom iterator yielding batches of images, poses, and metadata.
*   `render_kwargs`: Dictionary containing model-specific rendering parameters, including `near` and `far` clipping planes.
*   `optimizer`: Optimization state (e.g., Adam) holding gradients and momentum buffers.

**Steps:**
1.  **Fetch Batch:** Retrieve the next batch from `train_iterator`. If `StopIteration` is raised, reset the iterator to `iter(train_loader)` and fetch again [S1, unofficial].
2.  **Load Data:** Load dataset-specific assets:
    *   **LLFF:** Images, poses, bounds (`bds`), and test indices [S2, unofficial].
    *   **Blender:** Images, poses, and split indices (`i_train`, `i_val`, `i_test`) [S2, unofficial].
    *   **LINEMOD:** Intrinsics matrix $K$ and bounds [S1, unofficial].
    *   **DeepVoxels:** Calculate near/far bounds based on hemisphere radius $R_{hemi}$ [S2, unofficial].
3.  **Device Transfer:** Move ground truth images, camera matrices, and optional masks to the CUDA device [S1, unofficial].
4.  **Set Clipping Planes:** Configure `near` and `far` planes based on dataset type [S2, unofficial]:
    *   **LLFF:** If `no_ndc` is false, $near = 0$, $far = 1$. If true, $near = \min(bds) \times 0.9$, $far = \max(bds) \times 1.0$.
    *   **Blender:** $near = 2.0$, $far = 6.0$.
    *   **DeepVoxels:** $near = R_{hemi} - 1.0$, $far = R_{hemi} + 1.0$.
    *   Update `render_kwargs` with these bounds [S2, unofficial].
5.  **Render:**
    *   **NeRF:** Call the NeRF model (created via `create_nerf`) to render rays. The model computes volume rendering using the composited color formula:
        $$ \tilde{\mathbf{C}} = \sum_{i=1}^{N} \left(\alpha_i \boldsymbol{c}_i \prod_{j=1}^{i-1} \left(1 - \alpha_j\right)\right) $$
        where $\alpha_i$ is transmittance/opacity and $\boldsymbol{c}_i$ is color [S2, unofficial].
    *   **Gaussian Splatting:** Render scene using Gaussian primitives with background color $(0.0, 0.0, 0.0)$ or $(1.0, 1.0, 1.0)$ depending on implementation, image size $(128, 128)$, and per-splat settings [S1, unofficial].
6.  **Optimize:**
    *   Compute loss (e.g., MSE between rendered and ground truth).
    *   Call `loss.backward()` to compute gradients.
    *   Call `optimizer.step()` to update parameters.
    *   Call `optimizer.zero_grad()` to clear gradients for the next step [S1, unofficial].
7.  **Batching (Optional):** If `use_batching` is enabled (`not args.no_batching`), prepare ray batches by stacking rays from training images, concatenating with RGB values, transposing, and reshaping into a flat array of rays with corresponding colors [S2, unofficial].

**Parameters:**
*   `N_rand`: Number of random rays sampled per batch (from `args.N_rand`) [S2, unofficial].
*   `use_batching`: Boolean flag controlling ray batching [S2, unofficial].
*   `img_size`: Resolution for rendering, typically $(128, 128)$ [S1, unofficial].
*   `bg_colour`: Background color, typically $(0.0, 0.0, 0.0)$ or $(1.0, 1.0, 1.0)$ [S1, unofficial].

**Edge Cases:**
*   **Iterator Exhaustion:** Must handle `StopIteration` by resetting the iterator to ensure continuous training [S1, unofficial].
*   **Dataset Specific Bounds:** Incorrect near/far calculation leads to clipping artifacts or missing geometry [S2, unofficial].
*   **Precision:** Floating point precision in volume rendering accumulation can affect gradient stability [S2, unofficial].

## Subtopics

### [VCCNF Training Loop](4736-vccnf-training-loop.md)
A training procedure for a Voxelized Continuous Coordinate Normalizing Flow (VCCNF) that learns to map Gaussian noise to shape latent codes conditioned on voxel data, using MSE loss [4736-vccnf-training-loop.md].

### [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md)
Provides gradient guidance for 3D optimization by encoding RGB images and camera poses into a multi-view diffusion model, predicting noise, and stepping the DDIM scheduler [6345-mvdream-multi-view-diffusion-guidance.md].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and layers in the NeRF/Gaussian Splatting pipeline per iteration.
*   **Memory Complexity:** Proportional to the number of parameters in the radiance field and the batch size of rays.
*   **Strengths:** Flexible dataset handling; supports both NeRF and Gaussian Splatting backends.
*   **Limitations:** Performance heavily dependent on `N_rand` and hardware acceleration for ray marching.
*   **When to use:** Standard training for implicit or explicit 3D representations.
*   **When not to use:** When using specialized guidance like diffusion models (see MVDream) or normalizing flows (see VCCNF).

## Background
Originated from the NeRF paper (Mildenhall et al., 2020) and extended by Gaussian Splatting (Kerbl et al., 2023). The loop structure described here is based on common implementations in open-source repositories [S1, S2, unofficial].

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S3] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
