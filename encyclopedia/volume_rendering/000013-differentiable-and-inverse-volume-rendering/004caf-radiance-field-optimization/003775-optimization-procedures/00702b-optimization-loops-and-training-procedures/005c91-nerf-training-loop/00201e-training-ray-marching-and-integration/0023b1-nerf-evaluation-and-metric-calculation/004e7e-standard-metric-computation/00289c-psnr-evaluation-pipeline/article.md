# PSNR Evaluation Pipeline

Also called: NeRF validation loop, PSNR calculation, Quantitative Evaluation, Test Set PSNR, Validation and PSNR Evaluation Pipeline

## Overview
The PSNR Evaluation Pipeline computes the Peak Signal-to-Noise Ratio (PSNR) on held-out test or validation splits to quantitatively assess the rendering quality of a trained NeRF model [S1, unofficial]. It builds on [Standard Metric Computation](13741-standard-metric-computation.md) and differs from training metrics by using inference-optimized rendering paths and strictly handling ground truth transparency via white-background compositing to ensure fair comparison with the rendered RGB output [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Trained model state (coarse and/or fine MLPs), dataset split identifier (`"test"` or `"val"`), and optional resolution flag (`half_res`) [S1, unofficial].
*   **Outputs**: Per-frame PSNR values, mean PSNR, standard deviation, and optionally side-by-side visual comparisons of ground truth, rendered RGB, and depth [S1, S2, unofficial].

**Data Structures**
*   **Evaluation State**: A model configuration object where the standard training raymarch and render functions are replaced with inference variants (e.g., `state_eval = state.replace(raymarch=args.raymarch_eval, render=args.render_eval)`) [S2, unofficial].
*   **Image Arrays**: Ground truth images are stored as RGBA (uint8 or float32) and rendered images as RGB (float32) [S1, S2, unofficial].

**Steps**
1.  **Load Data**: Load the camera poses and images for the specified split. If `half_res` is true, images are loaded at half resolution (e.g., $200 \times 200$ for Blender datasets) [S1, unofficial].
2.  **Prepare State**: Instantiate the evaluation state using inference-specific rendering functions [S2, unofficial].
3.  **Composite Ground Truth**: For each ground truth image in the split, composite the RGBA image against a white background [S1, S2, unofficial].
4.  **Render**: Iterate over each validation view. For each view, extract the camera transform and render the image using the inference state. Chunked rendering is used to manage memory [S1, S2, unofficial].
5.  **Compute MSE**: Calculate the Mean Squared Error between the composite ground truth RGB and the rendered RGB [S1, S2, unofficial].
    *   Formula: $\text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (R_i - G_i)^2$, where $R_i$ is the rendered pixel and $G_i$ is the composite ground truth pixel [S2, unofficial].
6.  **Compute PSNR**: Convert MSE to PSNR [S1, S2, unofficial].
    *   Formula: $\text{PSNR} = -10 \cdot \log_{10}(\text{MSE})$ [S2, unofficial].
    *   *Edge Case*: If $\text{MSE} = 0$, PSNR is infinite [S1, unofficial].
7.  **Aggregate**: Accumulate PSNR values across all frames in the split. Compute the mean and standard deviation [S1, unofficial].
8.  **Visualize (Optional)**: Concatenate ground truth, rendered RGB, and depth maps side-by-side for qualitative inspection [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def load_data(dataset, split, half_res):
    """Standard routine returning images (N, H, W, 4) and poses (N, 4, 4)."""
    pass

def render_image(key, transform, state):
    """Standard routine returning a dict with 'rgb' (H, W, 3) and 'depth' (H, W, 1)."""
    pass

def evaluate_psnr(model_state, dataset, split="test", half_res=False):
    images, poses = load_data(dataset, split, half_res)
    eval_state = model_state.replace(raymarch=model_state.raymarch_eval, render=model_state.render_eval)
    psnr_values = []
    for i, (image_rgba, transform) in enumerate(zip(images, poses)):
        alpha = image_rgba[..., 3:4] / 255.0
        rgb_gt = image_rgba[..., :3] / 255.0
        white_bg = np.ones_like(rgb_gt)
        gt_rgb_composite = rgb_gt * alpha + white_bg * (1 - alpha)
        rendered = render_image(i, transform, eval_state)
        rendered_rgb = rendered['rgb']
        mse = np.mean((rendered_rgb - gt_rgb_composite) ** 2)
        if mse == 0:
            psnr = float('inf')
        else:
            psnr = -10.0 * math.log10(mse)
        psnr_values.append(psnr)
    mean_psnr = np.mean(psnr_values)
    std_psnr = np.std(psnr_values)
    return mean_psnr, std_psnr, psnr_values
```

**Parameters**
*   `split`: Dataset split to evaluate (`"test"` or `"val"`) [S1, unofficial].
*   `half_res`: Boolean flag to load images at half resolution for faster evaluation [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Infinite PSNR**: Occurs when MSE is exactly zero [S1, unofficial].
*   **Alpha Handling**: Failing to composite ground truth RGBA with a white background leads to incorrect MSE calculations against opaque RGB renders [S1, S2, unofficial].
*   **Resolution Mismatch**: Ensure rendered resolution matches ground truth resolution before computing MSE [S1, unofficial].

## Subtopics

### Quantitative Comparison
A variant of the evaluation pipeline that focuses on comparing PSNR values across different methods (e.g., NeRF vs. SDF-based methods) on standard datasets like NeRFsynthetic [S3]. It highlights that NeRF often achieves higher PSNR than geometric regularization-based methods because it prioritizes color reproduction over strict geometric fidelity [S3].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Standard, interpretable metric for image fidelity; easy to implement and compare across papers [S1, unofficial][S3].
*   **Limitations**: Does not measure geometric accuracy [S3].
*   **When to Use**: Standard validation for novel view synthesis tasks where photometric accuracy is the primary goal [S1, unofficial].

## Background
The PSNR evaluation pipeline is used to benchmark rendering quality on datasets like NeRFsynthetic (Blender) and DTU [S3]. The specific handling of alpha compositing and inference state replacement is detailed in community implementations and forks [S1, S2, unofficial].

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S2] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S3] PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces, http://arxiv.org/abs/2305.05594v1, fetched 2026-10-27
