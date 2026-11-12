# Unified Metric Aggregation

Also called: Evaluation & Metric Computation, Evaluation Reporting, Global and Per-view Metric Aggregation

## Overview
Unified Metric Aggregation processes the outputs of the NeRF evaluation pipeline to generate quantitative performance reports by aggregating per-view reconstruction metrics into global statistics [S1, S2, unofficial]. It differs from its siblings by handling the post-processing logic of collecting, validating, and averaging metrics (PSNR, SSIM, LPIPS) across all test views, whereas [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md) focuses on the specific calculation of PSNR and [Masked Evaluation](7531-masked-evaluation.md) focuses on spatially restricting these calculations to specific regions [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A list of paired file paths or tensors containing rendered RGB images and corresponding ground-truth (GT) images; optional depth maps; configuration flags for metric selection (PSNR, SSIM, LPIPS) and visualization saving [S1, S2, unofficial].
*   **Outputs:** A dictionary or JSON file containing global average metrics (PSNR, SSIM, LPIPS) and a nested dictionary of per-view metrics indexed by image name; optionally saved RGB/depth images and videos [S1, S2, unofficial][S3].

**Data Structures:**
*   **Per-view metrics list:** A list of dictionaries or tuples storing `{image_name: {PSNR: float, SSIM: float, LPIPS: float}}` for each view [S1, unofficial].
*   **Global metrics:** A single dictionary `{iteration: {PSNR: float, SSIM: float, LPIPS: float}}` representing the arithmetic mean of the per-view lists [S1, unofficial].
*   **Ray batching structure:** In some implementations, test rays are stored in a flat array `test_dataset.all_rays`, reshaped to `(H, W)` for rendering [S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize empty lists for per-view metrics and global metric accumulators [S1, unofficial].
2.  **Iteration Setup:** Determine the evaluation interval. If a maximum number of visualizations $N_{vis}$ is specified and $N_{vis} < 0$ (unlimited), evaluate all rays. Otherwise, calculate `img_eval_interval = max(test_dataset.all_rays.shape[0] // N_vis, 1)` and iterate over `test_dataset.all_rays[0::img_eval_interval]` [S2, unofficial].
3.  **Image Loading and Normalization:** For each pair, load the rendered RGB image and the GT image. Normalize pixel values to the range $[0, 1]$ and convert to CHW (Channel, Height, Width) tensor format [S1, unofficial].
4.  **Shape Validation:** Verify that the rendered image and GT image have identical dimensions (H, W, C). If not, skip or raise an error [S1, unofficial].
5.  **Metric Computation:**
    *   **PSNR:** Compute the Mean Squared Error (MSE) between the rendered RGB map ($R$) and GT ($G$):
        $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (R_i - G_i)^2 $$
        Then compute PSNR:
        $$ \text{PSNR} = -10 \cdot \frac{\log_{10}(\text{MSE})}{1} = -10 \cdot \log_{10}(\text{MSE}) $$
        [S2, unofficial].
    *   **SSIM:** If enabled, compute the Structural Similarity Index using a sliding window approach (typically AlexNet or VGG-based feature spaces for LPIPS, but SSIM is pixel/structural) [S1, S2, unofficial].
    *   **LPIPS:** If enabled, compute Learned Perceptual Image Patch Similarity using a pre-trained network (AlexNet or VGG) to compare perceptual features [S1, S2, unofficial].
6.  **Validation:** Check that all computed metric values are finite (not NaN or Inf). Discard or flag invalid results [S1, unofficial].
7.  **Storage:** Store the per-view metrics in the per-view dictionary [S1, unofficial].
8.  **Aggregation:** After iterating through all views, compute the arithmetic mean of the PSNR, SSIM, and LPIPS lists to obtain global metrics [S1, unofficial].
9.  **Reporting:** Write the global metrics and the per-view dictionary to a JSON file. The structure is:
    ```json
    {
      "ours_{iteration}": {
        "PSNR": float,
        "SSIM": float,
        "LPIPS": float
      }
    }
    ```
    And per-view:
    ```json
    {
      "ours_{iteration}": {
        "image_name": {
          "PSNR": float,
          "SSIM": float,
          "LPIPS": float
        }
      }
    }
    ```
    [S1, unofficial]. Optionally save rendered RGB/depth images and generate videos [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import json
import os

def compute_psnr(rendered_rgb: torch.Tensor, gt_rgb: torch.Tensor) -> float:
    """Compute PSNR from MSE between two RGB tensors normalized to [0,1]."""
    mse = torch.mean((rendered_rgb - gt_rgb) ** 2)
    if mse == 0:
        return float('inf')
    psnr = -10.0 * torch.log10(mse)
    return psnr.item()

def compute_ssim(rendered_rgb: torch.Tensor, gt_rgb: torch.Tensor) -> float:
    """Compute SSIM. Implementation details depend on library (e.g., skimage)."""
    # Placeholder for SSIM calculation logic
    pass

def compute_lpips(rendered_rgb: torch.Tensor, gt_rgb: torch.Tensor, net_type: str = 'alex') -> float:
    """Compute LPIPS using pre-trained network."""
    # Placeholder for LPIPS calculation logic
    pass

def unified_metric_aggregation(rendered_paths, gt_paths, save_dir, compute_extra_metrics=True):
    per_view_metrics = {}
    global_psnrs = []
    global_ssims = []
    global_lpips = []

    for rendered_path, gt_path in zip(rendered_paths, gt_paths):
        # 1. Load and normalize
        rendered_img = load_image(rendered_path) # Returns CHW tensor, [0,1]
        gt_img = load_image(gt_path)             # Returns CHW tensor, [0,1]

        # 2. Validate shape
        if rendered_img.shape != gt_img.shape:
            continue

        # 3. Compute Metrics
        psnr_val = compute_psnr(rendered_img, gt_img)
        
        # Validate finiteness
        if not np.isfinite(psnr_val):
            continue

        ssim_val = 0.0
        lpips_val = 0.0
        if compute_extra_metrics:
            ssim_val = compute_ssim(rendered_img, gt_img)
            lpips_val = compute_lpips(rendered_img, gt_img)

        # 4. Store per-view
        image_name = os.path.basename(gt_path)
        per_view_metrics[image_name] = {
            "PSNR": psnr_val,
            "SSIM": ssim_val,
            "LPIPS": lpips_val
        }

        global_psnrs.append(psnr_val)
        global_ssims.append(ssim_val)
        global_lpips.append(lpips_val)

    # 5. Aggregate
    if len(global_psnrs) > 0:
        global_metrics = {
            "PSNR": np.mean(global_psnrs),
            "SSIM": np.mean(global_ssims),
            "LPIPS": np.mean(global_lpips)
        }
    else:
        global_metrics = {"PSNR": 0, "SSIM": 0, "LPIPS": 0}

    # 6. Write JSON
    output_json = {
        "ours_iteration": global_metrics,
        "per_view": per_view_metrics
    }
    
    os.makedirs(save_dir, exist_ok=True)
    with open(os.path.join(save_dir, "metrics.json"), "w") as f:
        json.dump(output_json, f)
        
    return global_metrics, per_view_metrics
```

**Parameters:**
*   `compute_extra_metrics`: Boolean flag to enable SSIM and LPIPS computation. Default: `True` [S2, unofficial].
*   `N_vis`: Maximum number of views to visualize/evaluate. If $<0$, evaluate all. Default: Not specified [S2, unofficial].
*   `img_eval_interval`: Calculated step size for ray iteration. Default: `max(total_rays // N_vis, 1)` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **MSE = 0:** If the rendered image is identical to the GT, MSE is 0, leading to $\log(0)$ which is $-\infty$. PSNR should be handled as infinity or a large constant [S2, unofficial].
*   **Shape Mismatch:** Rendered and GT images must have identical dimensions. Mismatches should be skipped or handled explicitly [S1, unofficial].
*   **Non-finite Values:** NaN or Inf values in metrics must be validated and excluded from averages [S1, unofficial].

## Subtopics

### [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md)
The LPIPS Evaluation Protocol computes the Learned Perceptual Image Patch Similarity distance between reference and rendered images to assess perceptual quality. It is used in radiance field evaluation protocols, including those for 3D Gaussian Splatting variants. It differs from PSNR/SSIM by using deep feature spaces (AlexNet/VGG) to correlate better with human perception. Choose it when perceptual fidelity is more important than pixel-wise accuracy. Link: [LPIPS Evaluation Protocol](4376-lpips-evaluation-protocol.md).

### [Metric Computation (PSNR, SSIM, LPIPS)](9928-metric-computation-psnr-ssim-lpips.md)
This method computes standard photorealistic evaluation metrics—Peak Signal-to-Noise Ratio (PSNR), Structural Similarity Index (SSIM), and Learned Perceptual Image Patch Similarity (LPIPS)—to quantify the fidelity of synthesized novel views against ground-truth images. It differs from depth-specific metrics by focusing exclusively on RGB pixel-space and perceptual feature-space comparisons, serving as the final quantitative step in the NeRF evaluation pipeline after ray marching and integration. Link: [Metric Computation (PSNR, SSIM, LPIPS)](9928-metric-computation-psnr-ssim-lpips.md).

### Evaluation Reporting
This variant extends metric aggregation to include system-level performance indicators. It reports PSNR, SSIM, and LPIPS alongside the size of the resulting model (in MB) and rendering speed on specific hardware platforms (standalone PC, PC-VR, mobile VR) [S3]. It is chosen when evaluating real-time applicability and memory efficiency in addition to visual quality [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of test views $V$. Metric computation (PSNR/SSIM/LPIPS) is $O(H \cdot W)$ per view, where $H, W$ are image dimensions [S1, S2, unofficial].
*   **Memory Complexity:** $O(V \cdot H \cdot W)$ if storing all images in memory, or $O(H \cdot W)$ if processing views sequentially [S1, S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Provides standardized, comparable quantitative scores; separates visual quality from system performance metrics [S1, unofficial][S3].
*   **Limitations:** LPIPS computation is computationally expensive compared to PSNR; does not account for depth accuracy or geometric fidelity directly [S1, unofficial][S3].
*   **When to use:** Standard evaluation of novel view synthesis quality.
*   **When not to use:** When geometric accuracy (depth error) is the primary concern, or when real-time frame rate is the only metric of interest [S3].

## Background
Unified Metric Aggregation is a standard post-processing step in radiance field and NeRF evaluation pipelines. It originated from the need to standardize reporting across different neural rendering methods. The specific implementation details of aggregating PSNR, SSIM, and LPIPS are widely adopted in recent works such as 3D Gaussian Splatting evaluations [S1, unofficial] and Tensorial Radiance Fields [S2, unofficial]. The inclusion of system-level metrics (memory, speed) in variants like Evaluation Reporting reflects the shift towards real-time rendering applications in VR and mobile platforms [S3].

## Sources
[S1] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S2] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S3] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
