# Unified Metric Aggregation

Also called: Evaluation & Metric Computation, Evaluation Reporting, Global and Per-view Metric Aggregation

## Overview
Unified Metric Aggregation processes rendered images against ground truth references to compute, aggregate, and persist evaluation metrics for radiance field models [S1, S2, unofficial]. It solves the problem of standardizing performance reporting by converting raw per-pixel errors into standardized perceptual and structural scores (PSNR, SSIM, LPIPS) and organizing them into global averages and per-view dictionaries [S1, unofficial]. Unlike [Metric Computation (PSNR, SSIM, LPIPS)](9928-metric-computation-psnr-ssim-lpips.md), which calculates raw per-pixel or per-tensor values for individual image pairs, this method aggregates those scores into summary statistics and structured output files [S1, S2, unofficial]. It builds on [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by handling the iteration, normalization, and final aggregation logic [S1, S2, unofficial].

## How it works
**Inputs:**
- A dataset of test rays or image paths containing ground truth (GT) RGB images [S1, S2, unofficial].
- A renderer capable of producing predicted RGB images for given camera poses [S2, unofficial].
- Optional parameters: `compute_extra_metrics` (boolean, default True), `N_vis` (integer, controls sampling interval) [S2, unofficial].

**Outputs:**
- Global metrics dictionary: `{"ours_{iteration}": {"PSNR": float, "SSIM": float, "LPIPS": float}}` [S1, unofficial].
- Per-view metrics dictionary: `{"ours_{iteration}": {"image_name": {"PSNR": float, "SSIM": float, "LPIPS": float}}}` [S1, unofficial].
- Saved rendered images, depth visualizations, and videos [S2, unofficial].

**Data Structures:**
- `per_view_metrics`: List of dictionaries, one per view, containing `{"PSNR": float, "SSIM": float, "LPIPS": float}` [S1, unofficial].
- `global_metrics`: Dictionary with keys `PSNR`, `SSIM`, `LPIPS` storing arithmetic means [S1, unofficial].

**Steps:**
1. **Initialize**: Create an empty list `per_view_metrics` to store per-view results [S1, unofficial].
2. **Determine Sampling Interval**: If `N_vis` is negative, evaluate all views (`img_eval_interval = 1`). Otherwise, calculate `img_eval_interval = max(total_test_rays // N_vis, 1)` [S2, unofficial].
3. **Iterate Views**: Loop through the test dataset rays or image paths using the determined interval [S2, unofficial].
4. **Render**: For each view, render the predicted RGB image. Clamp the output to the range $[0, 1]$ [S2, unofficial].
5. **Load Ground Truth**: Load the corresponding GT RGB image. Normalize to $[0, 1]$ and convert to a CHW (Channel, Height, Width) tensor format [S1, unofficial].
6. **Validate Shapes**: Ensure the predicted and GT images have identical dimensions. If not, skip or handle the mismatch [S1, unofficial].
7. **Compute MSE**: Calculate the Mean Squared Error between the predicted RGB map ($\hat{I}$) and GT ($I$):
   $$ \text{MSE} = \frac{1}{H \times W \times C} \sum_{i,j,k} (\hat{I}_{i,j,k} - I_{i,j,k})^2 $$
   where $H, W$ are height and width, and $C$ is channels [S2, unofficial].
8. **Compute PSNR**: Calculate Peak Signal-to-Noise Ratio:
   $$ \text{PSNR} = -10 \cdot \frac{\log_{10}(\text{MSE})}{1} = -10 \cdot \log_{10}(\text{MSE}) $$
   [S2, unofficial].
9. **Compute Extra Metrics**: If `compute_extra_metrics` is true, calculate SSIM and LPIPS using standard libraries (e.g., AlexNet or VGG for LPIPS) [S2, unofficial].
10. **Validate Finiteness**: Check that all computed metrics are finite numbers. Discard or flag non-finite values [S1, unofficial].
11. **Store Per-View**: Append the per-view metrics dictionary to `per_view_metrics` [S1, unofficial].
12. **Aggregate Global**: After the loop, compute the arithmetic mean of the PSNR, SSIM, and LPIPS values across all views in `per_view_metrics` [S1, unofficial].
13. **Persist**: Write the global metrics and per-view JSON structures to disk [S1, unofficial]. Save rendered images, depth maps, and optionally videos [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unified_metric_aggregation(test_dataset, renderer, compute_extra_metrics=True, N_vis=-1):
    # Step 2: Determine sampling interval
    total_rays = test_dataset.all_rays.shape[0]
    if N_vis < 0:
        img_eval_interval = 1
    else:
        img_eval_interval = max(total_rays // N_vis, 1)
    
    per_view_metrics = []
    
    # Step 3: Iterate over test rays/images
    for i in range(0, total_rays, img_eval_interval):
        rays = test_dataset.all_rays[i]
        
        # Step 4: Render
        rgb_map = renderer.render(rays)
        rgb_map = torch.clamp(rgb_map, 0.0, 1.0)
        
        # Step 5: Load GT
        gt_rgb = load_gt_image(rays.camera_id)
        gt_rgb = normalize_to_01(gt_rgb)
        gt_tensor = rgb_to_chw_tensor(gt_rgb)
        pred_tensor = rgb_to_chw_tensor(rgb_map)
        
        # Step 6: Validate shapes
        if pred_tensor.shape != gt_tensor.shape:
            continue
            
        # Step 7: Compute MSE
        mse = torch.mean((pred_tensor - gt_tensor) ** 2)
        
        # Step 8: Compute PSNR
        psnr = -10.0 * torch.log10(mse)
        
        # Step 9: Compute Extra Metrics
        ssim = 0.0
        lpips = 0.0
        if compute_extra_metrics:
            ssim = compute_ssim(pred_tensor, gt_tensor)
            lpips = compute_lpips(pred_tensor, gt_tensor)
            
        # Step 10: Validate finiteness
        if not (torch.isfinite(psnr) and torch.isfinite(ssim) and torch.isfinite(lpips)):
            continue
            
        # Step 11: Store per-view
        view_name = get_view_name(rays.camera_id)
        per_view_metrics.append({
            "PSNR": psnr.item(),
            "SSIM": ssim.item(),
            "LPIPS": lpips.item(),
            "image_name": view_name
        })
        
        # Save images/videos if needed
        save_image(rgb_map, f"{view_name}_rgb.png")
        
    # Step 12: Aggregate Global
    if not per_view_metrics:
        return {}, {}
        
    global_psnr = sum(m["PSNR"] for m in per_view_metrics) / len(per_view_metrics)
    global_ssim = sum(m["SSIM"] for m in per_view_metrics) / len(per_view_metrics)
    global_lpips = sum(m["LPIPS"] for m in per_view_metrics) / len(per_view_metrics)
    
    global_metrics = {
        "PSNR": global_psnr,
        "SSIM": global_ssim,
        "LPIPS": global_lpips
    }
    
    # Step 13: Persist
    per_view_dict = {m["image_name"]: {k: v for k, v in m.items() if k != "image_name"} for m in per_view_metrics}
    write_json("global_metrics.json", {"ours_iteration": global_metrics})
    write_json("per_view_metrics.json", {"ours_iteration": per_view_dict})
    
    return global_metrics, per_view_dict
```

**Parameters:**
- `N_vis`: Integer. If negative, evaluate all views. Otherwise, subsample to approximately `N_vis` views [S2, unofficial].
- `compute_extra_metrics`: Boolean. If True, compute SSIM and LPIPS in addition to PSNR [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Shape Mismatch**: If rendered and GT images differ in resolution, the metric computation fails. Validation is required [S1, unofficial].
- **Non-finite Values**: NaN or Inf in metrics (e.g., from MSE=0 in PSNR log) must be handled or excluded [S1, unofficial].
- **Memory**: Loading all images into memory simultaneously may cause OOM errors; chunked processing is recommended [S2, unofficial].

## Subtopics

### [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md)
The LPIPS Evaluation Protocol computes the Learned Perceptual Image Patch Similarity distance between reference and rendered images to assess perceptual quality. It is used in radiance field evaluation protocols, including those for 3D Gaussian Splatting variants.

### [Metric Computation (PSNR, SSIM, LPIPS)](9928-metric-computation-psnr-ssim-lpips.md)
Computes evaluation metrics (PSNR, SSIM, LPIPS) between the ground truth and predicted images. These metrics compare synthesized views at testing viewpoints with the ground truth. Unlike [Unified Metric Aggregation](3644-unified-metric-aggregation.md) which averages scores, this method calculates the raw per-pixel or per-tensor values for each image pair.

### Evaluation Reporting
Evaluates the approach by comparing it to previous methods in specific domains such as anatomy visualization and thermal scene representation [S3]. Reports PSNR, SSIM, and LPIPS alongside model size (in MB) and rendering speed on various platforms (standalone PC, PC-VR, mobile VR) [S3].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of test views $N$. Each view requires rendering, metric computation (PSNR is $O(HW)$, SSIM/LPIPS depend on network depth and patch size), and I/O [S1, S2, unofficial].
- **Memory Complexity**: $O(HW)$ per view for image tensors. Global aggregation is $O(1)$ if streaming means are used, or $O(N)$ if storing all per-view scores [S1, unofficial].
- **Performance**: Depends heavily on the cost of LPIPS computation, which involves forward passes through VGG/AlexNet networks [S2, unofficial].
- **Strengths**: Standardized reporting, supports multiple metrics, handles subsampling for large datasets [S1, S2, unofficial].
- **Limitations**: Does not compute raw per-pixel errors for downstream analysis (see [Metric Computation](9928-metric-computation-psnr-ssim-lpips.md)) [S1, unofficial].

## Background
Unified Metric Aggregation is a standard post-processing step in radiance field evaluation pipelines [S1, S2, unofficial]. It originated from the need to compare novel 3D reconstruction methods (like NeRF and 3DGS) against ground truth using established computer vision metrics [S1, S2, unofficial]. It is widely used in academic benchmarks and open-source implementations such as Advanced3DGS and TensoRF [S1, S2, unofficial].

## Sources
[S1] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S2] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S3] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
