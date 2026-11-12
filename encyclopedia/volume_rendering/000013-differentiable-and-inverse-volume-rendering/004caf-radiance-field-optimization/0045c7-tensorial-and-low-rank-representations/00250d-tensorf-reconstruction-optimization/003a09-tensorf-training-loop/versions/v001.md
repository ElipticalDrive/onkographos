# TensoRF Training Loop

Also called: TensoRF Optimization

## Overview
The TensoRF Training Loop is the iterative optimization procedure used to minimize the L2 rendering loss of a tensorial radiance field, using ground truth pixel colors as supervision [S1, unofficial]. It differs from standard NeRF training by explicitly managing orthogonal, L1, and Total Variation (TV) regularizations on the decomposed tensor factors, and by supporting dynamic resolution updates via an alpha mask [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Training dataset containing rays and ground truth RGB values.
*   Model parameters: `density_n_comp`, `appearance_n_comp`, `app_dim`, `near_far`, `shadingMode`, `alphaMask_thres`, `density_shift`, `distance_scale`, `pos_pe`, `view_pe`, `fea_pe`, `featureC`, `step_ratio`, `fea2denseAct` [S1, unofficial].
*   Hyperparameters: `N_voxel_init` (initial voxel count), `lr_decay_target_ratio`, `Ortho_reg_weight`, `L1_reg_weight`, `TV_weight_density`, `TV_weight_app` [S1, S2, unofficial].

**Outputs:**
*   Optimized tensor factors representing the radiance field.
*   Rendered RGB maps, alphas, depths, weights, and uncertainties for the sampled rays [S1, unofficial].

### Data Structures
The model `tensorf` stores decomposed tensor factors for density and appearance. The exact layout is defined by the initialization parameters `density_n_comp`, `appearance_n_comp`, etc., which determine the dimensions of the CP (Canonical Polyadic) or TT (Tensor-Train) decomposition components [S1, unofficial].

### Steps
1.  **Initialization:** Load train/test data. Compute `aabb`, `near_far`, `white_bg`, and `ndc_ray` flags [S1, unofficial]. Initialize the `tensorf` model with the specified parameters [S1, unofficial]. Calculate learning rate decay factor:
    $$ lr\_factor = args.lr\_decay\_target\_ratio^{(1/args.lr\_decay\_iters)} $$
    [S1, unofficial]. Filter rays if not using NDC: `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs, bbox_only=True)` [S1, unofficial]. Initialize `SimpleSampler` for ray batching [S1, unofficial].

2.  **Iteration Loop:**
    a.  **Sampling:** Get ray indices: `ray_idx = trainingSampler.nextids()`. Fetch rays and RGBs: `rays_train, rgb_train = allrays[ray_idx], allrgbs[ray_idx].to(device)` [S1, unofficial].
    b.  **Rendering:** Compute number of samples per ray:
        $$ nSamples = \min(args.nSamples, cal\_n\_samples(reso\_cur, args.step\_ratio)) $$
        [S1, unofficial]. Render the scene:
        ```python
        rgb_map, alphas_map, depth_map, weights, uncertainty = renderer(
            rays_train, tensorf, chunk=args.batch_size, N_samples=nSamples,
            white_bg=white_bg, ndc_ray=ndc_ray, device=device, is_train=True
        )
        ```
        [S1, unofficial].
    c.  **Loss Computation:**
        i.  **MSE Loss:**
            $$ loss = \text{mean}((rgb\_map - rgb\_train)^2) $$
            [S1, unofficial].
        ii. **Orthogonal Regularization:** If `Ortho_reg_weight > 0`:
            $$ loss\_reg = tensorf.vector\_comp\_diffs() $$
            $$ total\_loss = loss + Ortho\_reg\_weight \times loss\_reg $$
            [S1, unofficial].
        iii. **L1 Density Regularization:** If `L1_reg_weight > 0`:
            $$ loss\_reg\_L1 = tensorf.density\_L1() $$
            $$ total\_loss += L1\_reg\_weight \times loss\_reg\_L1 $$
            [S1, unofficial].
        iv. **TV Density Regularization:** If `TV_weight_density > 0`:
            $$ TV\_weight\_density = TV\_weight\_density \times lr\_factor $$
            $$ loss\_tv = tensorf.TV\_loss\_density(tvreg) \times TV\_weight\_density $$
            $$ total\_loss += loss\_tv $$
            [S1, unofficial].
        v.  **TV Appearance Regularization:** If `TV_weight_app > 0`:
            $$ TV\_weight\_app = TV\_weight\_app \times lr\_factor $$
            $$ loss\_tv = tensorf.TV\_loss\_app(tvreg) \times TV\_weight\_app $$
            $$ total\_loss += loss\_tv $$
            [S1, unofficial].

    d.  **Optimization Step:**
        ```python
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        ```
        [S1, unofficial]. Update learning rate for all parameter groups:
        ```python
        for param_group in optimizer.param_groups:
            param_group['lr'] = param_group['lr'] * lr_factor
        ```
        [S1, unofficial].

3.  **Evaluation and Update:** Periodically evaluate test PSNR and log metrics to TensorBoard [S1, unofficial]. If the current iteration is in `update_AlphaMask_list` and the current voxel count is less than $256^3$, update the volume resolution [S1, unofficial].

### Parameters
*   `N_voxel_init`: Initial number of voxels [S1, unofficial].
*   `nSamples`: Number of samples per ray, calculated as `min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))` [S1, unofficial].
*   `lr_decay_target_ratio`: Target learning rate ratio for decay [S1, unofficial].
*   `Ortho_weight`: Weight for orthogonal regularization [S1, unofficial].
*   `L1_weight_inital`: Initial weight for L1 density regularization [S1, unofficial].
*   `TV_weight_density`: Initial weight for TV density regularization [S1, unofficial].
*   `TV_weight_app`: Initial weight for TV appearance regularization [S1, unofficial].

### Edge Cases and Pitfalls
*   **Depth Supervision Ineffectiveness:** In low-data scenarios, depth supervision yields minimal PSNR improvement (0.01) because COLMAP key points are orders of magnitude fewer than color pixels, causing depth loss gradients to affect only a small portion of the voxel space [S2, unofficial].
*   **Joint Pose Optimization Failure:** Naively applying joint optimization on voxel-based NeRFs leads to failure as premature high-frequency signals in the voxel volume cause camera poses to get stuck in local minima [S3, unofficial].

## Subtopics

### [NeuManifold](9771-neumanifold.md)
A two- (or three-) stage pipeline for neural watertight manifold reconstruction with efficient rendering support. Stage 1 initializes geometry and appearance networks via volume rendering. Stage 2 optimizes topology, geometry, and appearance using differentiable rasterization with a proposed differentiable marching cubes (DiffMC) iso-surface extraction. Stage 3 optionally fine-tunes mesh vertex positions and appearance to improve rendering quality [9771-neumanifold.md].

### TensoRF Downsampling
A variant where both train and test data are downsampled by a factor of 4. Training iterations are reduced to 2,500 (vs. 30,000 in the original TensoRF paper) due to memory and time constraints. Tested on the synthetic Lego scene from the NeRF dataset with 100 training views and 200 test views [S2, unofficial].

### Depth Supervised TensoRF
Applies depth supervision loss during training. As noted in the pitfall section, this yields minimal improvement in low-data scenarios due to the sparsity of COLMAP key points compared to color pixels [S2, unofficial].

### Separable Component-Wise Convolution for Joint Pose Optimization
A technique to control the spectrum of the radiance field by performing separable component-wise convolution of Gaussian filters on the decomposed tensor and 2D supervision images, preventing premature high-frequency signals from causing camera poses to get stuck in local minima during joint optimization [S3, unofficial]. Steps include applying Gaussian filters to the 3D tensor volume and 2D supervision images, then using the filtered radiance field for joint optimization [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of iterations (typically 30,000 for full data, 2,500 for downsampled variants) and the number of samples per ray [S1, S2, unofficial].
*   **Memory Complexity:** Determined by `N_voxel_init` and the tensor decomposition dimensions [S1, unofficial].
*   **Strengths:** Supports dynamic resolution updates via alpha masks; includes multiple regularization terms (Orthogonal, L1, TV) to stabilize training [S1, unofficial].
*   **Limitations:** Depth supervision is ineffective in low-data regimes due to sparse keypoint coverage [S2, unofficial]. Joint pose optimization requires spectral filtering to avoid local minima [S3, unofficial].

## Background
The TensoRF method was introduced in the paper "Tensorial Radiance Fields" (ECCV 2022) [S1, unofficial]. It optimizes a tensorial radiance field per scene via gradient descent to minimize L2 rendering loss [3453-tensorf-reconstruction-optimization.md].

## Sources
[S1] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S3] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
