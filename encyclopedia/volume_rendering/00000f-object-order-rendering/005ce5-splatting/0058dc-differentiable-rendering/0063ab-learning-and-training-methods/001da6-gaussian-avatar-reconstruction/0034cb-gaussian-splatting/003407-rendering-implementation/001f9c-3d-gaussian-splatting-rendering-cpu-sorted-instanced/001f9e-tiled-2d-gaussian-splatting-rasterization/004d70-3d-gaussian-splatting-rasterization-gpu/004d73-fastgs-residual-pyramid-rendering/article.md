# FastGS Residual Pyramid Rendering

Also called: Laplacian pyramid accumulation, multi-scale Gaussian rendering

## Overview
FastGS Residual Pyramid Rendering is a multi-scale technique that decomposes the target image into a Laplacian pyramid of Gaussian splats [S1, unofficial]. The coarsest level renders the base image, while subsequent levels render signed residuals (differences) that are upsampled and accumulated to reconstruct the final high-resolution image [S1, unofficial]. It sets itself apart from standard single-scale rasterization by allowing independent optimization and rendering of detail at different resolutions, which facilitates efficient training and high-fidelity reconstruction of complex scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A list of Gaussian models (`gaussians_list`) organized from coarsest (L0) to finest (L_{n-1}); a list of camera views; ground-truth images; a rasterization function (`render_fn`); and a pyramid processing module (`pyramid_func`) [S1, unofficial].
*   **Outputs:** A rendered image tensor; intermediate buffers for accumulated predictions and ground-truth residuals; and metadata including visibility filters and radii [S1, unofficial].

**Data Structures:**
*   `gaussians_list`: A list of `gaussians` objects, where each object contains 3D Gaussian parameters (means, covariances, opacities, spherical harmonics coefficients) [S1, unofficial].
*   `gt_gaussian_pyr_dict`: A dictionary mapping camera IDs to lists of ground-truth Gaussian pyramid levels [S1, unofficial].
*   `pred_accumulated_dict`: A dictionary mapping camera IDs to the accumulated prediction tensor at each stage [S1, unofficial].
*   `render_pkg`: A dictionary returned by `render_fn` containing:
    *   `'render'`: `torch.Tensor`, the rendered image [S1, unofficial].
    *   `'viewspace_points'`: `torch.Tensor`, shape `[N, 4]`, requires gradient [S1, unofficial].
    *   `'visibility_filter'`: `torch.Tensor`, indices where screen-space radii are greater than 0 [S1, unofficial].
    *   `'radii'`: `torch.Tensor`, screen-space radii of splats [S1, unofficial].
    *   `'accum_metric_counts'`: `torch.Tensor`, metric accumulation buffer [S1, unofficial].

**Parameters:**
*   `pos_and_neg`: Boolean flag. When `False`, the rasterizer outputs values clamped to `[0, 1]`. When `True`, it outputs signed values in `[-1.0, 1.0]` to represent residuals [S1, unofficial].
*   `mult`: Scalar multiplier passed to `GaussianRasterizationSettings`, likely controlling opacity or scale weighting [S1, unofficial].
*   `render_size`: Target resolution for the current stage, derived from the ground-truth pyramid level shape [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Load Gaussian models from subfolders `L0` to `L_{n-1}` into `gaussians_list` [S1, unofficial].
2.  **Ground-Truth Pyramid Construction:** For each view, compute the ground-truth Gaussian pyramid using `pyramid_func.build_pyr_gaussian(gt_full.to('cuda').unsqueeze(0))` [S1, unofficial]. Store results in `gt_gaussian_pyr_dict` [S1, unofficial].
3.  **Stage Iteration:** Iterate `i_stage` from `0` to `len(gaussians_list) - 1` [S1, unofficial].
4.  **Retrieve Data:**
    *   Retrieve `gaussians = gaussians_list[-i_stage-1]` (note: the source code iterates backwards through the list, so index `-i_stage-1` accesses the coarsest level first if the list is ordered finest-to-coarsest, or vice versa depending on construction; the source implies `L0` is processed first) [S1, unofficial].
    *   Retrieve `gt_gau = gt_gaussian_pyr_dict[viewpoint_cam.uid][-i_stage-1].squeeze(0)` [S1, unofficial].
5.  **Rendering and Accumulation:**
    *   **If `i_stage == 0`:**
        *   Call `render_pkg = render_fn(view, gaussians, pipeline, background, opt.mult, render_size=gt_gau.shape[-2:], pos_and_neg=False)` [S1, unofficial].
        *   Extract `rendering = render_pkg["render"]` [S1, unofficial].
        *   Set `pred_accumulated = rendering` [S1, unofficial].
        *   Set `gt = gt_gau` [S1, unofficial].
    *   **If `i_stage > 0`:**
        *   Call `render_pkg = render_fn(view, gaussians, pipeline, background, opt.mult, render_size=gt_gau.shape[-2:], pos_and_neg=True)` [S1, unofficial].
        *   Extract `rendering = render_pkg["render"]` [S1, unofficial].
        *   Compute `upsampled = pyramid_func.get_upsampled([rendering.unsqueeze(0), pred_accumulated.unsqueeze(0)])` [S1, unofficial].
        *   Squeeze `upsampled` to remove batch dimension: `upsampled = upsampled.squeeze(0)` [S1, unofficial].
        *   Update `pred_accumulated = rendering + upsampled` [S1, unofficial].
        *   Normalize `rendering` to `[0, 1]` via `(rendering + 1.0) / 2.0` [S1, unofficial].
        *   Compute `gt = gt_gau - upsampled` [S1, unofficial].
        *   Normalize `gt` to `[0, 1]` via `(gt + 1.0) / 2.0` [S1, unofficial].
6.  **Training/Update:** Use the computed `rendering` and `gt` for loss calculation and gradient backpropagation for the current stage's Gaussians [S1, unofficial].
7.  **Inheritance/Transition:** After each stage (except the last), prepare Gaussians for the next stage. This may involve densification, pruning, or extracting a subset of Gaussians based on `pipe.inherit_state` (e.g., "keep_and_adc", "only_xyz", "with_state") [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def render_fastgs_stage(view, gaussians, pipeline, background, opt, pyramid_func, gt_gau, pred_accumulated, i_stage, render_fn):
    """
    Renders a single stage of the FastGS Residual Pyramid.
    
    Args:
        view: Camera viewpoint object.
        gaussians: Gaussian model object for the current stage.
        pipeline: Pipeline configuration.
        background: Background color tensor.
        opt: Options object containing 'mult'.
        pyramid_func: Module with build_pyr_gaussian and get_upsampled.
        gt_gau: Ground truth tensor for the current pyramid level.
        pred_accumulated: Accumulated prediction from previous stages.
        i_stage: Current stage index (0 for coarsest).
        render_fn: The rasterization function.
        
    Returns:
        rendering: Rendered image tensor.
        pred_accumulated: Updated accumulated prediction.
        gt: Normalized ground truth for loss computation.
    """
    render_size = gt_gau.shape[-2:]
    
    if i_stage == 0:
        # Coarsest level: render base image
        render_pkg = render_fn(
            view, gaussians, pipeline, background, 
            opt.mult, 
            render_size=render_size, 
            pos_and_neg=False
        )
        rendering = render_pkg["render"]
        pred_accumulated = rendering
        gt = gt_gau
    else:
        # Subsequent levels: render signed residuals
        render_pkg = render_fn(
            view, gaussians, pipeline, background, 
            opt.mult, 
            render_size=render_size, 
            pos_and_neg=True
        )
        rendering = render_pkg["render"]
        
        # Upsample the previous accumulated prediction and the current residual
        # Note: get_upsampled likely performs bilinear or nearest-neighbor upsampling
        # to match the resolution of the current stage's ground truth.
        upsampled = pyramid_func.get_upsampled([
            rendering.unsqueeze(0), 
            pred_accumulated.unsqueeze(0)
        ])
        upsampled = upsampled.squeeze(0)
        
        # Update accumulated prediction
        pred_accumulated = rendering + upsampled
        
        # Normalize rendering to [0, 1] for loss computation
        rendering_normalized = (rendering + 1.0) / 2.0
        
        # Compute residual ground truth
        gt = gt_gau - upsampled
        # Normalize ground truth to [0, 1]
        gt = (gt + 1.0) / 2.0
        
        rendering = rendering_normalized

    return rendering, pred_accumulated, gt

def fastgs_render_loop(gaussians_list, gt_full, viewpoint_cams, pipeline, background, opt, pyramid_func, render_fn):
    """
    Main loop for FastGS Residual Pyramid Rendering.
    """
    device = gt_full.device
    
    # 1. Build Ground-Truth Pyramid
    gt_gaussian_pyr_dict = {}
    for viewpoint_cam in viewpoint_cams:
        gt_full_cuda = gt_full.to(device).unsqueeze(0)
        pyr_gpu = pyramid_func.build_pyr_gaussian(gt_full_cuda)
        # Store levels, detaching and moving to device
        gt_gaussian_pyr_dict[viewpoint_cam.uid] = [pyr_gpu[0]] + [lvl.detach().to(device) for lvl in pyr_gpu[1:]]
        
    # 2. Initialize Accumulated Predictions
    pred_accumulated_dict = {}
    
    # 3. Iterate through stages
    num_stages = len(gaussians_list)
    for i_stage in range(num_stages):
        # Retrieve Gaussians for this stage
        # Note: Source code uses gaussians_list[-i_stage-1], implying list order might be finest-to-coarsest
        # or the indexing is specific to the implementation. Assuming standard L0..Ln-1 order where L0 is coarsest:
        gaussians = gaussians_list[i_stage] 
        
        for viewpoint_cam in viewpoint_cams:
            gt_gau = gt_gaussian_pyr_dict[viewpoint_cam.uid][i_stage].squeeze(0)
            
            pred_accumulated = pred_accumulated_dict.get(viewpoint_cam.uid, None)
            
            # Render current stage
            rendering, pred_accumulated, gt = render_fastgs_stage(
                viewpoint_cam, gaussians, pipeline, background, opt, 
                pyramid_func, gt_gau, pred_accumulated, i_stage, render_fn
            )
            
            pred_accumulated_dict[viewpoint_cam.uid] = pred_accumulated
            
            # Perform training step with 'rendering' and 'gt'
            # (Loss calculation and optimizer step omitted as per source structure)
            
        # Prepare Gaussians for next stage (inheritance/densification)
        if i_stage < num_stages - 1:
            # Logic for inherit_state, densification, pruning
            # See source for specific implementation details
            pass
```

**Edge Cases and Pitfalls:**
*   **PyTorch Version Compatibility:** The `viewspace_points` tensor is initialized with `requires_grad=True`, but `retain_grad()` may fail on older PyTorch versions. The code uses a `try-except` block to handle this gracefully [S1, unofficial].
*   **Resolution Mismatch:** The `render_size` must match the shape of the current ground-truth pyramid level `gt_gau.shape[-2:]` to ensure correct upsampling and accumulation [S1, unofficial].
*   **Signed Values:** When `pos_and_neg=True`, the rasterizer outputs values in `[-1.0, 1.0]`. Failure to normalize these values to `[0, 1]` before loss computation will lead to incorrect gradients [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires rendering at multiple scales. If there are $N$ stages, the total rendering time is roughly proportional to $N$ times the cost of a single rasterization pass, plus the cost of pyramid construction and upsampling.
*   **Memory Complexity:** Memory usage increases with the number of stages due to the storage of multiple Gaussian models and intermediate pyramid levels.
*   **Strengths:** Allows for efficient training by starting with coarse details and refining them. Reduces artifacts associated with single-scale rendering at high resolutions.
*   **Limitations:** Requires careful management of Gaussian inheritance and densification between stages. More complex implementation than single-scale rasterization.
*   **When to use:** When high-fidelity reconstruction of complex scenes is required, and training time/quality trade-offs favor multi-scale optimization.
*   **When not to use:** For simple scenes or real-time applications where the overhead of multi-scale rendering is prohibitive.

## Background
FastGS Residual Pyramid Rendering was introduced in the context of 3D Gaussian Splatting to improve rendering quality and training efficiency. It builds on the principles of Laplacian pyramids, applying them to the Gaussian splatting framework. The method was developed as part of the "Laplacian-GS" project [S1, unofficial].

## Sources
[S1] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
