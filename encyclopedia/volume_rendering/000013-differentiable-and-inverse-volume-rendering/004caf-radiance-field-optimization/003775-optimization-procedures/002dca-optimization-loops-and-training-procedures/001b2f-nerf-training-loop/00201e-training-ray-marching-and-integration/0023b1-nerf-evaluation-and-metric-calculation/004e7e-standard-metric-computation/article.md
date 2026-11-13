# Standard Metric Computation

Also called: NeRF Evaluation Metrics, Photorealistic Quality Metrics

## Overview
Standard Metric Computation quantifies the photorealistic and perceptual quality of rendered NeRF images by comparing them against ground-truth data using PSNR, SSIM, and LPIPS [S1, S2, unofficial][S3]. It differs from sibling methods by focusing on the post-processing stage: aggregating per-view scores into global statistics and structured reports, rather than performing the ray marching or the individual metric calculations themselves [S1, S2, unofficial]. It serves as the final quantitative step in the NeRF evaluation pipeline, ensuring that results from [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md) and [Masked Evaluation](7531-masked-evaluation.md) are standardized for comparison [S1, unofficial][S3].

## Shared principles
The family operates on paired lists of rendered and ground-truth (GT) images, typically in tensor format (CHW: Channel, Height, Width) normalized to $[0, 1]$ [S1, S2, unofficial]. The core mechanism involves computing pixel-wise errors and perceptual distances, then aggregating them into global means [S1, S2, unofficial].

**MSE and PSNR:**
The Mean Squared Error (MSE) is calculated as:
$$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (R_i - G_i)^2 $$
where $R_i$ is the rendered pixel value, $G_i$ is the GT pixel value, and $N$ is the total number of pixels [S2, unofficial]. The Peak Signal-to-Noise Ratio (PSNR) is derived as:
$$ \text{PSNR} = -10 \cdot \log_{10}(\text{MSE}) $$
[2, unofficial].

**SSIM and LPIPS:**
Structural Similarity Index (SSIM) and Learned Perceptual Image Patch Similarity (LPIPS) are computed if enabled. LPIPS typically uses an AlexNet or VGG backbone [S2, unofficial].

**Aggregation:**
Global metrics are the arithmetic mean of all per-view metrics [S1, S2, unofficial].

## Subtopics

### [Unified Metric Aggregation](3644-unified-metric-aggregation.md)
Unified Metric Aggregation processes rendered images against ground-truth data to compute, aggregate, and persist photorealistic and perceptual quality metrics. It differs from sibling methods by focusing on the post-processing stage: aggregating per-view scores into global statistics and structured reports, rather than performing the ray marching or the individual metric calculations themselves. It serves as the final quantitative step in the NeRF evaluation pipeline, ensuring that results from [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md) and [Masked Evaluation](7531-masked-evaluation.md) are standardized for comparison.

### [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md)
The PSNR Evaluation Pipeline computes the Peak Signal-to-Noise Ratio (PSNR) on held-out test or validation splits to quantify the rendering quality of a trained NeRF model. It differs from training metrics by using inference-optimized rendering paths and explicitly handling background compositing to ensure fair comparison against ground truth images.

### Official Metrics Evaluation
This variant manages setup, preflight, and PSNR/SSIM/LPIPS acceptance evidence across Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets [S1, unofficial]. It verifies that methods produce finite metrics before acceptance [S1, unofficial].
- Load render and ground truth images as tensors, handling batched or unbatched formats based on the style [S1, unofficial].
- Select metric functions based on style: `standard_3dgs` uses `loss_utils.ssim` and `batched=True`; `beta_splatting` uses `fused_ssim` and `batched=False` [S1, unofficial].
- Compute LPIPS using `lpips_module.lpips(render, gt, net_type="vgg").mean()` [S1, unofficial].
- Check that all metric values are finite [S1, unofficial].
- Compute mean metrics across all views and save to `results.json` and `per_view.json` [S1, unofficial].

## Background
The standard metrics (PSNR, SSIM, LPIPS) are widely used in computer vision for image quality assessment. In the context of NeRF, they were adopted to benchmark novel view synthesis quality against ground truth images. The evaluation pipeline typically involves rendering test views, compositing backgrounds, and computing these metrics to compare different NeRF architectures and training procedures. The "Official Metrics Evaluation" process described in [S1, unofficial] suggests a formalized acceptance criteria for method repositories, ensuring reproducibility and correctness of metric computation across different implementations.

## Sources
[S1] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
