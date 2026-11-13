# Unified Metric Aggregation

Also called: Evaluation & Metric Computation, Evaluation Reporting, Global and Per-view Metric Aggregation

## Overview
Unified Metric Aggregation processes rendered images against ground-truth data to compute, aggregate, and persist photorealistic and perceptual quality metrics [S1, S2, unofficial][S3]. It differs from sibling methods by focusing on the post-processing stage: aggregating per-view scores into global statistics and structured reports, rather than performing the ray marching or the individual metric calculations themselves [S1, S2, unofficial]. It serves as the final quantitative step in the NeRF evaluation pipeline, ensuring that results from [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md) and [Masked Evaluation](7531-masked-evaluation.md) are standardized for comparison [S1, unofficial][S3].

## How it works
**Inputs:**
- Paired lists of file paths or tensors for rendered images and ground-truth (GT) images [S1, S2, unofficial].
- Optional configuration flags: `compute_extra_metrics` (boolean for SSIM/LPIPS) [S2, unofficial], `N_vis` (integer controlling evaluation interval) [S2, unofficial].
- GT images must exist for PSNR/SSIM/LPIPS computation; depth maps may be generated for visualization [S2, unofficial].

**Outputs:**
- Global metrics dictionary: `{"ours_{iteration}": {"PSNR": float, "SSIM": float, "LPIPS": float}}` [S1, unofficial].
- Per-view metrics dictionary: `{"ours_{iteration}": {"image_name": {"PSNR": float, "SSIM": float, "LPIPS": float}}}` [S1, unofficial].
- Saved RGB and depth images/videos [S2, unofficial].
- Text/JSON files containing the metrics [S1, S2, unofficial].

**Data Structures:**
- `per_view_metrics`: List or dictionary storing per-image results [S1, S2, unofficial].
- `global_metrics`: Dictionary storing arithmetic means [S1, S2, unofficial].

**Steps:**
1. **Initialize Evaluation Interval:** If `N_vis < 0`, set `img_eval_interval = 1` (evaluate all rays/images). Otherwise, calculate `img_eval_interval = max(total_test_rays // N_vis, 1)` [S2, unofficial].
2. **Iterate Over Test Set:** Loop through the test dataset rays/images using the calculated interval (e.g., `test_dataset.all_rays[0::img_eval_interval]`) [S2, unofficial].
3. **Render and Prepare:**
   - Reshape rays to image dimensions ($H \times W$) [S2, unofficial].
   - Render the view using a chunked renderer [S2, unofficial].
   - Clamp the resulting RGB map to $[0, 1]$ [S2, unofficial].
   - Generate depth visualization if required [S2, unofficial].
4. **Load Ground Truth:**
   - Load the corresponding GT RGB image [S1, unofficial].
   - Normalize GT to $[0, 1]$ [S1, unofficial].
   - Convert both rendered and GT images to CHW (Channel, Height, Width) tensor format [S1, unofficial].
5. **Validate Shapes:** Check that rendered and GT tensors have identical dimensions; skip or error if mismatched [S1, unofficial].
6. **Compute Metrics:**
   - **MSE:** Calculate Mean Squared Error: $\text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (R_i - G_i)^2$, where $R_i$ is rendered pixel, $G_i$ is GT pixel, $N$ is total pixels [S2, unofficial].
   - **PSNR:** Calculate Peak Signal-to-Noise Ratio: $\text{PSNR} = -10 \cdot \frac{\log_{10}(\text{MSE})}{1}$ [S2, unofficial].
   - **Extra Metrics:** If `compute_extra_metrics` is true, compute SSIM and LPIPS (using AlexNet or VGG backbone) [S2, unofficial].
7. **Validate Results:** Ensure all computed metric values are finite (not NaN or Inf) [S1, unofficial].
8. **Store Per-View:** Append the per-image metrics to the `per_view_metrics` structure, keyed by image name [S1, S2, unofficial].
9. **Save Visuals:** Save the rendered RGB and depth images to disk [S2, unofficial].
10. **Aggregate Global Metrics:** After the loop, compute the arithmetic mean of all values in `per_view_metrics` for each metric type (PSNR, SSIM, LPIPS) [S1, S2, unofficial].
11. **Persist Results:** Write global metrics and per-view dictionaries to JSON/text files [S1, S2, unofficial]. Optionally compile videos from saved frames [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_unified_metrics(rendered_images, gt_images, compute_extra_metrics=False, N_vis=-1):
    """
    Compute and aggregate PSNR, SSIM, and LPIPS metrics.
    rendered_images: List of tensors or paths, shape (N, C, H, W) or iterable
    gt_images: List of tensors or paths, shape (N, C, H, W) or iterable
    compute_extra_metrics: bool, whether to compute SSIM and LPIPS
    N_vis: int, number of images to evaluate. -1 means all.
    """
    # Step 1: Determine evaluation count
    total_images = len(rendered_images)
    if N_vis < 0:
        eval_count = total_images
        step = 1
    else:
        eval_count = N_vis
        step = max(total_images // N_vis, 1)
    
    per_view_results = {}
    psnr_list = []
    ssim_list = []
    lpips_list = []
    
    # Step 2: Iterate
    for i in range(0, total_images, step):
        if i >= total_images:
            break
            
        # Step 3 & 4: Load and Normalize
        # Assume rendered_images[i] and gt_images[i] are loaded and normalized to [0,1]
        # Convert to CHW tensor if not already
        render_tensor = normalize_to_chw(rendered_images[i])
        gt_tensor = normalize_to_chw(gt_images[i])
        
        # Step 5: Validate
        if render_tensor.shape != gt_tensor.shape:
            continue # Skip mismatched
            
        # Step 6: Compute Metrics
        # MSE
        mse = torch.mean((render_tensor - gt_tensor) ** 2)
        
        # PSNR
        psnr_val = -10.0 * torch.log10(mse).item()
        psnr_list.append(psnr_val)
        
        # Extra Metrics
        ssim_val = None
        lpips_val = None
        if compute_extra_metrics:
            ssim_val = compute_ssim(render_tensor, gt_tensor)
            ssim_list.append(ssim_val)
            lpips_val = compute_lpips(render_tensor, gt_tensor)
            lpips_list.append(lpips_val)
            
        # Step 7: Validate Finiteness
        if not torch.isfinite(torch.tensor(psnr_val)):
            continue
            
        # Step 8: Store Per-View
        image_name = f"image_{i}"
        per_view_results[image_name] = {
            "PSNR": psnr_val,
            "SSIM": ssim_val if ssim_val is not None else "N/A",
            "LPIPS": lpips_val if lpips_val is not None else "N/A"
        }
        
        # Step 9: Save Visuals (Implementation detail omitted as per source structure)
        save_image(render_tensor, image_name)

    # Step 10: Aggregate Global
    global_psnr = sum(psnr_list) / len(psnr_list) if psnr_list else 0.0
    global_ssim = sum(ssim_list) / len(ssim_list) if ssim_list else 0.0
    global_lpips = sum(lpips_list) / len(lpips_list) if lpips_list else 0.0
    
    global_metrics = {
        "PSNR": global_psnr,
        "SSIM": global_ssim,
        "LPIPS": global_lpips
    }
    
    # Step 11: Persist
    write_json(global_metrics, "global_metrics.json")
    write_json(per_view_results, "per_view_metrics.json")
    
    return global_metrics, per_view_results
```

**Parameters:**
- `N_vis`: Controls the subset of test views evaluated. Default often implies all views if negative [S2, unofficial].
- `compute_extra_metrics`: Boolean flag to enable SSIM/LPIPS. Default False in some implementations to save compute [S2, unofficial].

**Edge Cases:**
- **Shape Mismatch:** Rendered and GT images must have identical dimensions; otherwise, the pair is skipped [S1, unofficial].
- **NaN/Inf:** Metric values must be validated for finiteness before aggregation [S1, unofficial].
- **Missing GT:** If GT images are unavailable, PSNR/SSIM/LPIPS cannot be computed; only visualization or relative metrics might be saved [S2, unofficial].

## Subtopics

### [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md)
LPIPS is a full-reference perceptual quality metric that measures image similarity by comparing deep feature representations extracted from a pretrained convolutional neural network. Unlike pixel-wise metrics such as PSNR or MSE, which often correlate poorly with human perception, LPIPS aligns closely with human judgments of image similarity by leveraging semantic features. It is primarily used in volume rendering and NeRF optimization to evaluate reconstruction fidelity or as a perceptual loss function to guide training, capturing nuances like texture and structure that standard metrics miss.

### [Metric Computation (PSNR, SSIM, LPIPS)](9928-metric-computation-psnr-ssim-lpips.md)
This method computes standard photorealistic evaluation metrics—Peak Signal-to-Noise Ratio (PSNR), Structural Similarity Index (SSIM), and Learned Perceptual Image Patch Similarity (LPIPS)—to quantify the fidelity of synthesized novel views against ground-truth images. It differs from depth-specific metrics by focusing exclusively on RGB pixel-space and perceptual feature-space comparisons, serving as the final quantitative step in the NeRF evaluation pipeline after ray marching and integration.

### Evaluation Reporting
This variant extends standard metric aggregation by including system-level and performance benchmarks alongside visual quality metrics. It reports PSNR, SSIM, and LPIPS, but also includes the size of the resulting model/splats in MB and rendering speeds on specific hardware platforms (standalone PC, PC-VR, mobile VR) [S3]. This is used to evaluate approaches in distinct domains like anatomy visualization and thermal scene representation, highlighting both rendering quality and memory/efficiency trade-offs [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of evaluated images $N$. The dominant cost is the metric computation (especially LPIPS, which involves forward passes through CNNs) [S2, unofficial][S3].
- **Memory Complexity:** Linear with respect to the number of images stored in the per-view dictionary and the temporary tensors for metric calculation [S1, S2, unofficial].
- **Performance:** LPIPS computation is significantly slower than PSNR/SSIM due to deep network inference [S2, unofficial]. Using `N_vis` to subsample test views reduces total evaluation time [S2, unofficial].
- **Strengths:** Provides a comprehensive view of quality (photometric + perceptual) and system performance (speed/size) [S3]. Standardized JSON output facilitates automated tracking [S1, unofficial].
- **Limitations:** Requires ground-truth images for PSNR/SSIM/LPIPS [S2, unofficial]. LPIPS is computationally expensive [S2, unofficial]. Does not evaluate geometric accuracy (depth) directly in the metric values, though depth images are saved [S2, unofficial].

## Background
Unified Metric Aggregation is a standard post-processing step in NeRF and Gaussian Splatting evaluation pipelines. It originated from the need to standardize reporting across different radiance field methods, moving from simple MSE to perceptual metrics like LPIPS [S1, S2, unofficial][S3]. It is widely used in academic papers and open-source implementations (e.g., TensoRF, Advanced3DGS) to compare novel view synthesis quality [S1, S2, unofficial][S3].

## Sources
[S1] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S2] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S3] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
