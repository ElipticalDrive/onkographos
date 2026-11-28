# Depth Supervision

Also called: Depth Prior Supervision, Monocular Depth Supervision

## Overview
Depth Supervision is a training technique that improves reconstructed geometry by providing depth priors using off-the-shelf depth estimation models, resulting in better geometry and reduced number of floaters [S1, unofficial]. It differs from [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md) by explicitly integrating external monocular depth priors rather than relying solely on photometric consistency, and builds on [Depth Priors and Initialization](15188-depth-priors-and-initialization.md) shared principles.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset containing images and corresponding depth maps stored in `<dataset-root>/depths/<FILENAME>.png`, where `FILENAME` matches the dataset image name [S1, unofficial]. A boolean parameter `use_depth_loss` in `OptimizationParams` enables the feature [S1, unofficial].
*   **Outputs:** Updated neural network parameters (or Gaussian attributes) that minimize the depth discrepancy.

**Data Structures:**
*   Depth maps are stored as PNG files in a subdirectory `depths/` relative to the dataset root [S1, unofficial].
*   The implementation must handle a boolean flag `use_depth_loss` to toggle the loss calculation [S1, unofficial].

**Steps:**
1.  **Load:** Retrieve depth maps from the dataset directory [S1, unofficial].
2.  **Refit:** Since depth estimates from off-the-shelf models may vary in scale between frames, all depth maps are refitted against a prior point cloud to a common scale [S1, unofficial].
3.  **Discard:** Depth maps that fail the refitting process against the prior point cloud are discarded and not used for supervision [S1, unofficial].
4.  **Render:** Compute the rendered depth $\hat{D}(\mathbf{r}) = \sum_{i=1}^N w_i t_i$ using the differentiable volume rendering framework [G].
5.  **Loss:** Enforce that the rendered depth matches the depth predicted by a pre-trained monocular depth network (e.g., MiDaS [37]) for the input image [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import statistics

def compute_depth_loss(rendered_depth, monocular_depth, prior_point_cloud_depths, use_depth_loss):
    """
    Computes the depth supervision loss.
    
    Args:
        rendered_depth: List of floats representing the rendered depth map.
        monocular_depth: List of floats representing the depth predicted by a monocular network.
        prior_point_cloud_depths: List of floats representing depths from the prior point cloud.
        use_depth_loss: Boolean flag to enable/disable depth supervision.
        
    Returns:
        loss: Scalar float representing the depth loss.
    """
    if not use_depth_loss:
        return 0.0

    # Filter valid depth values (positive and finite)
    valid_mono = [d for d in monocular_depth if d > 0 and math.isfinite(d)]
    valid_prior = [d for d in prior_point_cloud_depths if d > 0 and math.isfinite(d)]

    # Discard invalid maps that fail refitting preparation
    if not valid_mono or not valid_prior:
        return 0.0

    # Refit: estimate scale factor s using median ratio for robustness against outliers
    ratios = [p / m for p, m in zip(valid_prior, valid_mono)]
    scale_factor = statistics.median(ratios)

    # [G] Sources do not specify a quality threshold or exact refitting algorithm.
    # We proceed with the estimated scale factor.
    refitted_depth = [d * scale_factor for d in monocular_depth]

    # Compute L1 loss between rendered and refitted depth
    errors = [abs(r - rd) for r, rd in zip(rendered_depth, refitted_depth)]
    loss = sum(errors) / len(errors) if errors else 0.0
    return loss
```

**Parameters:**
*   `use_depth_loss`: A boolean parameter in `OptimizationParams` that enables depth supervision during training [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Refitting Failure:** Depth maps that fail the refitting process against the prior point cloud are discarded [S1, unofficial].
*   **Harmful Effects:** In some cases, monocular depth supervision was found to be harmful to reconstruction quality [S2].
*   **Noise:** Regularization terms involving depth can be noisy [S2].

## Subtopics

### [Refined Depth for Geometry Supervision](19001-refined-depth-for-geometry-supervision.md)
Refined Depth for Geometry Supervision is a lightweight depth-regularization objective used during per-scene training to refine reconstructed geometry, specifically improving local sharpness and depth consistency without sacrificing efficiency. It sets itself apart from standard photometric losses by explicitly leveraging rendered depths and input images to derive a geometry-supervision loss ($L_{refD}$), which is combined with the original training loss to yield more geometrically faithful surfaces with fast convergence.

## Operating characteristics
*   **Time Complexity:** Adds overhead for monocular depth inference and refitting per iteration [S1, unofficial][S2].
*   **Memory Complexity:** Requires storage for depth maps in the dataset directory [S1, unofficial].
*   **Measured Performance:** Sources report mixed results; one notes better geometry and reduced floaters [S1, unofficial], while another found it did not noticeably improve quality and was sometimes harmful [S2].
*   **Strengths:** Can reduce floaters and improve geometric fidelity when successful [S1, unofficial].
*   **Limitations:** Can be harmful to reconstruction quality in some cases [S2]. Results are not conclusive [S2].
*   **When to use:** When floaters are a significant issue and other methods are not available [S2].
*   **When not to use:** When reconstruction quality is already high and depth supervision introduces noise or harm [S2].

## Background
*   **Origin:** The technique is discussed in the context of Gaussian Splatting optimization [S1, unofficial] and RealFusion [S2].
*   **History:** It was explored as a way to improve geometry in single-view or multi-view reconstruction pipelines by leveraging pre-trained monocular depth networks like MiDaS [37] [S2].
*   **Usage:** Used in 3D reconstruction pipelines to constrain the location of surfaces [S1, unofficial].

## Sources
[S1] Better Gaussian Splatting in Julia, https://pxl-th.github.io/blog/better-gs-julia/, fetched 2026-10-26 (unofficial)
[S2] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
