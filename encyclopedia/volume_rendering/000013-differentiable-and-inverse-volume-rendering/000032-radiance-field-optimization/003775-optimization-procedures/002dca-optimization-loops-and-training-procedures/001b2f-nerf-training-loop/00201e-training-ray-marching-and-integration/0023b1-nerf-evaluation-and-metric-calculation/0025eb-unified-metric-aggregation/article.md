# Unified Metric Aggregation

Also called: Evaluation Reporting, Global and Per-view Metric Aggregation

## Overview
Unified Metric Aggregation consolidates per-view rendering metrics (PSNR, SSIM, LPIPS) into global averages and structured per-view dictionaries for evaluation reporting [S1, unofficial]. It distinguishes itself by standardizing the output format for both summary statistics and granular image-level results, enabling consistent tracking across training iterations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Pairs of predicted and ground truth (GT) image file paths; iteration identifier.
- **Outputs:** Global metric dictionary (JSON-serializable); per-view metric dictionary (JSON-serializable).

**Data Structures:**
- `global_metrics`: Dictionary with key `ours_{iteration}` mapping to a dictionary of floats: `{"PSNR": float, "SSIM": float, "LPIPS": float}`.
- `per_view_metrics`: Dictionary with key `ours_{iteration}` mapping to a nested dictionary where keys are `image_name` and values are `{"PSNR": float, "SSIM": float, "LPIPS": float}`.

**Steps:**
1. Initialize empty lists: `psnr_list`, `ssim_list`, `lpips_list`.
2. Iterate over each pair of `(pred_path, gt_path)` in the test set.
3. Load `pred_img` and `gt_img` from paths.
4. Normalize pixel values to $[0, 1]$.
5. Convert images to CHW tensor format (Channels, Height, Width).
6. Verify `pred_img` and `gt_img` have identical shapes. If not, raise error or skip [G].
7. Compute PSNR, SSIM, and LPIPS scores for the pair.
8. Validate that all computed scores are finite numbers. If not, handle as error [G].
9. Append scores to respective lists. Store individual scores in `per_view_metrics` under `image_name`.
10. After loop, compute arithmetic mean for each list:
    $$ \text{Global PSNR} = \frac{1}{N} \sum_{i=1}^{N} \text{PSNR}_i $$
    $$ \text{Global SSIM} = \frac{1}{N} \sum_{i=1}^{N} \text{SSIM}_i $$
    $$ \text{Global LPIPS} = \frac{1}{N} \sum_{i=1}^{N} \text{LPIPS}_i $$
11. Populate `global_metrics` with the computed means.
12. Write `global_metrics` and `per_view_metrics` to disk as JSON files.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aggregate_metrics(pred_paths, gt_paths, iteration_id):
    psnr_list = []
    ssim_list = []
    lpips_list = []
    per_view = {}
    
    for pred_path, gt_path in zip(pred_paths, gt_paths):
        pred_img = load_image(pred_path)
        gt_img = load_image(gt_path)
        
        # Normalize to [0, 1]
        pred_img = pred_img / 255.0
        gt_img = gt_img / 255.0
        
        # Convert to CHW tensor
        pred_tensor = to_chw_tensor(pred_img)
        gt_tensor = to_chw_tensor(gt_img)
        
        # Check shape match
        assert pred_tensor.shape == gt_tensor.shape, "Shape mismatch"
        
        # Compute metrics
        psnr = compute_psnr(pred_tensor, gt_tensor)
        ssim = compute_ssim(pred_tensor, gt_tensor)
        lpips = compute_lpips(pred_tensor, gt_tensor)
        
        # Validate finiteness
        assert np.isfinite(psnr) and np.isfinite(ssim) and np.isfinite(lpips)
        
        psnr_list.append(psnr)
        ssim_list.append(ssim)
        lpips_list.append(lpips)
        
        image_name = os.path.basename(pred_path)
        per_view[image_name] = {
            "PSNR": psnr,
            "SSIM": ssim,
            "LPIPS": lpips
        }
    
    n = len(psnr_list)
    global_metrics = {
        "PSNR": sum(psnr_list) / n if n > 0 else 0.0,
        "SSIM": sum(ssim_list) / n if n > 0 else 0.0,
        "LPIPS": sum(lpips_list) / n if n > 0 else 0.0
    }
    
    output = {
        f"ours_{iteration_id}": global_metrics,
        f"ours_{iteration_id}_per_view": per_view
    }
    
    return output
```

**Parameters:**
- `iteration_id`: String identifier for the current training step. Typical values: `"1000"`, `"50000"`.
- `normalize_range`: Always $[0, 1]$.

**Edge Cases:**
- Shape mismatch between predicted and GT images must be handled explicitly [G].
- Non-finite values (NaN, Inf) in metrics must be detected and reported [G].

## Subtopics

### [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md)
Assesses perceptual quality using deep feature representations. Preferred over pixel-space metrics for robustness.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of test images and $C$ is the cost of computing PSNR, SSIM, and LPIPS per image. LPIPS is the most expensive due to deep network inference.
- **Memory Complexity:** $O(N)$ to store per-view metrics. Global metrics are $O(1)$.
- **Strengths:** Standardized output; supports both global and per-view analysis.
- **Limitations:** LPIPS computation is slow and GPU-dependent.
- **When to use:** Final evaluation of NeRF models.
- **When not to use:** Real-time feedback during training (too slow).

## Background
Originated in the context of NeRF evaluation pipelines to standardize reporting of PSNR, SSIM, and LPIPS [S1, unofficial]. Developed as part of the Advanced3DGS project to facilitate consistent metric aggregation across iterations [S1, unofficial].

## Sources
[S1] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
