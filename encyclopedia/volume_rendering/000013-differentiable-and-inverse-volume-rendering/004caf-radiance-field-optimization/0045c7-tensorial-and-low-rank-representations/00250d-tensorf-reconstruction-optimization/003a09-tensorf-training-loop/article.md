# TensoRF Training Loop

Also called: TensoRF Optimization

## Overview
The TensoRF Training Loop is the iterative optimization procedure used to minimize the L2 rendering loss of a tensorial radiance field, using ground truth pixel colors as supervision [S1, unofficial]. It differs from standard NeRF training by incorporating specific regularization terms (orthogonal, L1, and Total Variation) tailored to the low-rank tensor decomposition structure, alongside a dynamic resolution update mechanism for the feature grid [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Training dataset (images, poses, intrinsics) [S1, unofficial].
    *   Model parameters: `density_n_comp`, `appearance_n_comp`, `app_dim`, `near_far`, `shadingMode`, `alphaMask_thres`, `density_shift`, `distance_scale`, `pos_pe`, `view_pe`, `fea_pe`, `featureC`, `step_ratio`, `fea2denseAct` [S1, unofficial].
    *   Hyperparameters: `N_voxel_init` (initial voxel count), `nSamples` (samples per ray), `lr_decay_target_ratio`, `Ortho_weight`, `L1_weight_inital`, `TV_weight_density`, `TV_weight_app` [S1, unofficial].
*   **Outputs:**
    *   Optimized tensorial radiance field parameters [S1, unofficial].
    *   Rendered RGB maps, alphas, depths, and weights for loss computation [S1, unofficial].

### Data Structures
*   **Rays:** 3D arrays containing ray origins and directions, filtered by bounding box if not using NDC [S1, unofficial].
*   **RGBs:** 3D arrays containing ground truth pixel colors [S1, unofficial].
*   **TensorF Model:** Contains the decomposed tensor components for density and color fields [S1, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Load train/test data and compute `aabb`, `near_far`, `white_bg`, and `ndc_ray` flags [S1, unofficial].
    *   Initialize the `tensorf` model with the specified parameters [S1, unofficial].
    *   Calculate learning rate decay factor:
        $$ lr\_factor = args.lr\_decay\_target\_ratio^{(1/args.lr\_decay\_iters)} $$
    *   If not using NDC, filter rays to the bounding box:
        ```python
        allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs, bbox_only=True)
        ```
    *   Initialize `SimpleSampler` for ray batching [S1, unofficial].

2.  **Iteration Loop:**
    *   For each iteration:
        a.  **Ray Sampling:** Get ray indices via `ray_idx = trainingSampler.nextids()`. Fetch rays and RGBs:
            ```python
            rays_train, rgb_train = allrays[ray_idx], allrgbs[ray_idx].to(device)
            ```
        b.  **Rendering:** Render the scene using the `renderer` function:
            ```python
            rgb_map, alphas_map, depth_map, weights, uncertainty = renderer(
                rays_train, tensorf, chunk=args.batch_size, N_samples=nSamples, 
                white_bg=white_bg, ndc_ray=ndc_ray, device=device, is_train=True
            )
            ```
            *Note: `nSamples` is calculated as `min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))` [S1, unofficial].*
        c.  **Loss Computation:**
            i.  **MSE Loss:**
                $$ loss = \text{mean}((rgb\_map - rgb\_train)^2) $$
            ii. **Orthogonal Regularization:** If `Ortho_reg_weight > 0`:
                $$ loss\_reg = tensorf.vector\_comp\_diffs() $$
                $$ total\_loss += Ortho\_reg\_weight \times loss\_reg $$
            iii. **L1 Density Regularization:** If `L1_reg_weight > 0`:
                $$ loss\_reg\_L1 = tensorf.density\_L1() $$
                $$ total\_loss += L1\_reg\_weight \times loss\_reg\_L1 $$
            iv. **TV Density Regularization:** If `TV_weight_density > 0`:
                Decay weight by `lr_factor`. Compute:
                $$ loss\_tv = tensorf.TV\_loss\_density(tvreg) \times TV\_weight\_density $$
                $$ total\_loss += loss\_tv $$
            v.  **TV Appearance Regularization:** If `TV_weight_app > 0`:
                Decay weight by `lr_factor`. Compute:
                $$ loss\_tv = tensorf.TV\_loss\_app(tvreg) \times TV\_weight\_app $$
                $$ total\_loss += loss\_tv $$
        d.  **Optimization Step:**
            ```python
            optimizer.zero_grad()
            total_loss.backward()
            optimizer.step()
            ```
        e.  **Learning Rate Update:**
            For all parameter groups:
            ```python
            param_group['lr'] = param_group['lr'] * lr_factor
            ```
        f.  **Evaluation:** Periodically evaluate test PSNR and log metrics to TensorBoard [S1, unofficial].
        g.  **Alpha Mask Update:** If the current iteration is in `update_AlphaMask_list` and the current voxel count is less than $256^3$, update the volume resolution [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_tensorf(dataset, model, optimizer, args):
    # 1. Initialization
    allrays, allrgbs = dataset.get_all_rays_and_rgbs()
    aabb, near_far, white_bg, ndc_ray = dataset.compute_bounds_and_focal()
    
    model.init_parameters(
        density_n_comp=args.density_n_comp,
        appearance_n_comp=args.appearance_n_comp,
        app_dim=args.app_dim,
        near_far=near_far,
        shadingMode=args.shadingMode,
        alphaMask_thres=args.alphaMask_thres,
        density_shift=args.density_shift,
        distance_scale=args.distance_scale,
        pos_pe=args.pos_pe,
        view_pe=args.view_pe,
        fea_pe=args.fea_pe,
        featureC=args.featureC,
        step_ratio=args.step_ratio,
        fea2denseAct=args.fea2denseAct
    )
    
    lr_factor = args.lr_decay_target_ratio ** (1.0 / args.lr_decay_iters)
    
    if not ndc_ray:
        allrays, allrgbs = model.filtering_rays(allrays, allrgbs, bbox_only=True)
        
    sampler = SimpleSampler(allrays.shape[0])
    
    # 2. Iteration Loop
    for iteration in range(args.max_iters):
        # a. Ray Sampling
        ray_idx = sampler.nextids()
        rays_train = allrays[ray_idx].to(device)
        rgb_train = allrgbs[ray_idx].to(device)
        
        # b. Rendering
        n_samples = min(args.nSamples, cal_n_samples(model.reso_cur, args.step_ratio))
        rgb_map, alphas_map, depth_map, weights, uncertainty = model.renderer(
            rays_train, model, chunk=args.batch_size, N_samples=n_samples,
            white_bg=white_bg, ndc_ray=ndc_ray, device=device, is_train=True
        )
        
        # c. Loss Computation
        loss = torch.mean((rgb_map - rgb_train) ** 2)
        total_loss = loss
        
        # Orthogonal Regularization
        if args.Ortho_reg_weight > 0:
            loss_reg = model.vector_comp_diffs()
            total_loss += args.Ortho_reg_weight * loss_reg
            
        # L1 Density Regularization
        if args.L1_reg_weight > 0:
            loss_reg_L1 = model.density_L1()
            total_loss += args.L1_reg_weight * loss_reg_L1
            
        # TV Density Regularization
        if args.TV_weight_density > 0:
            tv_weight = args.TV_weight_density * lr_factor
            loss_tv = model.TV_loss_density(tvreg=1) * tv_weight
            total_loss += loss_tv
            
        # TV Appearance Regularization
        if args.TV_weight_app > 0:
            tv_weight = args.TV_weight_app * lr_factor
            loss_tv = model.TV_loss_app(tvreg=1) * tv_weight
            total_loss += loss_tv
            
        # d. Optimization Step
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
        # e. Learning Rate Update
        for param_group in optimizer.param_groups:
            param_group['lr'] = param_group['lr'] * lr_factor
            
        # f. Evaluation
        if iteration % args.eval_interval == 0:
            evaluate_test_psnr(dataset, model)
            log_to_tensorboard(iteration, total_loss)
            
        # g. Alpha Mask Update
        if iteration in args.update_AlphaMask_list:
            if model.current_voxel_count < 256**3:
                model.update_volume_resolution()
```

### Parameters with Typical Values
*   `N_voxel_init`: Initial number of voxels. Not given in the sources.
*   `nSamples`: Number of samples per ray. Calculated dynamically [S1, unofficial].
*   `lr_decay_target_ratio`: Target learning rate ratio for decay. Not given in the sources.
*   `Ortho_weight`: Weight for orthogonal regularization. Not given in the sources.
*   `L1_weight_inital`: Initial weight for L1 density regularization. Not given in the sources.
*   `TV_weight_density`: Initial weight for TV density regularization. Not given in the sources.
*   `TV_weight_app`: Initial weight for TV appearance regularization. Not given in the sources.

### Edge Cases and Pitfalls
*   **Depth Supervision Ineffectiveness:** When using depth supervision with low data, the improvement in PSNR is minimal (0.01 PSNR) because COLMAP key points are orders of magnitude fewer than color pixel comparisons, causing depth loss gradients to affect only a small portion of the voxel space [S2, unofficial].
*   **Resolution Update Condition:** The volume resolution is only updated if the current voxel count is less than $256^3$ [S1, unofficial].

## Subtopics

### [NeuManifold](9771-neumanifold.md)
NeuManifold is a three-stage pipeline for reconstructing neural watertight manifolds with efficient, high-quality rendering support. It solves the problem of generating high-fidelity 3D meshes from 2D images by combining the robust initialization of volume rendering with the geometric precision of differentiable rasterization. Unlike pure volumetric methods, it produces explicit triangle meshes; unlike pure mesh-based methods, it avoids the artifacts caused by linear interpolation in non-linear density fields by using a differentiable marching cubes algorithm. [NeuManifold](9771-neumanifold.md)

### [Separable Component-Wise Convolution for Joint Pose Optimization](10576-separable-component-wise-convolution-for-joint-pose-optimiza.md)
This method enables robust joint optimization of camera poses and decomposed low-rank tensorial radiance fields by controlling the frequency spectrum of the scene representation. It prevents premature high-frequency signals in the voxel volume from causing camera poses to get stuck in local minima, a failure mode common in naive joint optimization. It sets itself apart from hash-encoding-based methods by specifically leveraging the separability of the input low-rank tensor signal, making it more efficient than traditional separable convolution kernels. [Separable Component-Wise Convolution for Joint Pose Optimization](10576-separable-component-wise-convolution-for-joint-pose-optimiza.md)

### [StyleTRF](11815-styletrf.md)
StyleTRF is a method for stylized view generation that decouples style adaptation from view capture by fine-tuning the appearance component of a pre-optimized Tensorial Radiance Field (TensoRF) using sparse stylized priors. It sets itself apart from joint optimization methods like SNeRF by freezing the geometric density parameters of the scene representation, thereby preserving crisp geometry while adapting appearance, and by using a deterministic CNN-based stylization module (Johnson et al.) to ensure 3D consistency across viewpoints. [StyleTRF](11815-styletrf.md)

### [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md)
This family defines the tensor decomposition strategies used to factorize the radiance field density and color fields in TensoRF, balancing storage efficiency against rendering quality. It differs from sibling methods like [NeuManifold](9771-neumanifold.md) or [StyleTRF](11815-styletrf.md) by explicitly structuring the low-rank approximation into discrete rank groups (rank-residual learning) and offering specific variants (CP, HY, HY-S) to control the ratio of vector- to matrix-based components. [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md)

### [Coarse-to-fine feature grid](15277-coarse-to-fine-feature-grid.md)
A training optimization scheme that reduces initial training cost and regularizes feature representations by initializing a 3D feature grid at a coarse resolution and progressively upscaling it to higher resolutions. It improves model convergence and makes camera refinement more effective compared to training at full resolution from the start. Unlike static grid methods, this approach dynamically adapts the spatial detail of the feature representation during the optimization process. [Coarse-to-fine feature grid](15277-coarse-to-fine-feature-grid.md)

### [Codec-Adaptive TriPlane Radiance Fields](17898-codec-adaptive-triplane-radiance-fields.md)
Codec-Adaptive TriPlane Radiance Fields (CATRF) is a compression framework for plane-factorized radiance fields that optimizes neural features directly in the decoded space of a standard video or image codec. It solves the problem of severe rendering degradation at low bitrates by integrating a real codec round-trip (JPEG, VP9, HEVC, or AV1) into the training loop via a Straight-Through Estimator (STE). Unlike Codec-Agnostic (CA) methods that compress features after training, or Learned-Codec-in-the-Loop (LCL) methods that require shipping scene-specific neural decoder parameters, CATRF adapts features to real codec distortions without introducing any learned codec parameters or entropy models. [Codec-Adaptive TriPlane Radiance Fields](17898-codec-adaptive-triplane-radiance-fields.md)

### TensoRF Downsampling
Downsamples both train and test data by a factor of 4. Training iterations are reduced to 2,500 (vs. 30,000 in original TensoRF paper) due to memory and time constraints. Tested on the synthetic Lego scene from the NeRF dataset with 100 training views and 200 test views [S2, unofficial].

### Depth Supervised TensoRF
Applies depth supervision loss during training. However, it suffers from low PSNR with low data and minimal improvement (0.01 PSNR) over non-depth supervised approaches because COLMAP key points are orders of magnitude fewer than color pixel comparisons, causing depth loss gradients to affect only a small portion of voxel space [S2, unofficial].

### TensoRF with PATA
Uses a spike-based color decoder. TensoRF T: Maximum time step set to 8. Initial membrane time constant $\tau = 2.0$. Firing threshold = 0.5. TensoRF Input Dimensions: Spike-based color decoder receives 150-dimensional input on Synthetic-NeRF and 30-dimensional input on LLFF [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:** The use of orthogonal, L1, and TV regularizations helps stabilize the optimization of low-rank tensors. However, depth supervision can be ineffective if the number of depth points is significantly lower than the number of color pixels [S2, unofficial].
*   **When to use it:** When optimizing a tensorial radiance field for high-quality novel view synthesis with efficient memory usage.
*   **When not to use it:** When depth supervision is critical and high-precision depth data is unavailable [S2, unofficial].

## Background
The TensoRF Training Loop originates from the TensoRF paper (ECCV 2022), which introduced tensorial radiance fields as a compact representation for neural radiance fields [S1, unofficial]. The method builds on the principles of differentiable volume rendering and low-rank tensor decomposition.

## Sources
[S1] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S3] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
