# Near-Far Clipping

Also called: Fixed Near/Far Plane Collider, NearFar Collider

## Overview
Near-Far Clipping is a post-processing step in ray generation that constrains ray sampling intervals to a specified viewing volume defined by fixed global near and far distances [S1, S2, unofficial]. It ensures that samples are only generated within valid bounds, preventing rays from sampling empty space beyond the far plane or artifacts near the camera origin [S1, unofficial]. Unlike [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md) which adapts sampling density to scene content, or [Near-Far Linear/Disparity Ray Generation with Stratification](11841-near-far-linear-disparity-ray-generation-with-stratification.md) which handles stratified intervals, this method applies rigid, uniform bounds to all rays [S2, unofficial].

## How it works
**Inputs:**
- `ray_bundle`: A data structure containing ray origins, directions, and currently assigned near/far distances.
- `self.near_plane`: The minimum distance from the camera origin for valid samples [S1, unofficial].
- `self.far_plane`: The maximum distance from the camera origin for valid samples [S1, unofficial].
- `self.reset_near_plane`: A boolean flag (default `True`) [S2, unofficial].

**Outputs:**
- Updated `ray_bundle` with `nears` and `fars` tensors overwritten [S2, unofficial].

**Data Structures:**
- `ray_bundle.nears`: Tensor of shape `(batch_size, 1)` or matching `ray_bundle.origins[..., 0:1]`, storing the start distance of each ray [S2, unofficial].
- `ray_bundle.fars`: Tensor of shape `(batch_size, 1)` or matching `ray_bundle.origins[..., 0:1]`, storing the end distance of each ray [S2, unofficial].

**Steps:**
1. Create a tensor `ones` of ones with shape matching `ray_bundle.origins[..., 0:1]` [S2, unofficial].
2. Determine the effective `near_plane` value:
   - If `self.training` is `True` OR `self.reset_near_plane` is `False`, use `self.near_plane` [S2, unofficial].
   - Otherwise (inference mode with reset enabled), use `0.0` [S2, unofficial].
3. Set `ray_bundle.nears` to `ones * near_plane` [S2, unofficial].
4. Set `ray_bundle.fars` to `ones * self.far_plane` [S2, unofficial].
5. Return the updated `ray_bundle` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_near_far_clipping(ray_bundle, near_plane, far_plane, reset_near_plane, training):
    """
    Clips ray samples to specified near and far planes.
    
    Args:
        ray_bundle: Object containing ray origins and near/far tensors.
        near_plane: Float, the configured near distance.
        far_plane: Float, the configured far distance.
        reset_near_plane: Bool, whether to reset near to 0 during inference.
        training: Bool, whether currently in training mode.
        
    Returns:
        Updated ray_bundle.
    """
    # 1. Create ones tensor matching the shape of ray origins' first dimension
    # Assuming origins shape is [..., 3], we take slice 0:1 to get shape [..., 1]
    ones = torch.ones_like(ray_bundle.origins[..., 0:1])
    
    # 2. Determine effective near plane
    if training or not reset_near_plane:
        effective_near = near_plane
    else:
        effective_near = 0.0
        
    # 3. Set near distances
    ray_bundle.nears = ones * effective_near
    
    # 4. Set far distances
    ray_bundle.fars = ones * far_plane
    
    # 5. Return updated bundle
    return ray_bundle
```

**Parameters:**
- `near_plane`: Minimum distance from camera origin [S1, unofficial]. Typical values vary by scene; e.g., 1.00 for kitchen, 5.00 for playroom [S3].
- `far_plane`: Maximum distance from camera origin [S1, unofficial]. Not given in the sources.
- `reset_near_plane`: Boolean flag (default `True`) [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Clipping Artifacts:** Using a non-zero `near_plane` during inference can cause clipping artifacts if the evaluation or viewer camera moves closer to the object than the `near_plane` distance [S2, unofficial]. This is mitigated by `reset_near_plane=True` [S2, unofficial].
- **Precision:** Not given in the sources.
- **Boundaries:** Rays originating exactly at the near plane are included [S2, unofficial].

## Subtopics

### Near Clipping Plane Floater Masking
A technique to reduce the visual impact of floaters (noisy Gaussians near the camera) by using near clipping planes [S3]. For ray tracers, tracing starts further along the initial ray; for splatting, Gaussians with centroids closer than $d_{\text{near}}$ are culled [S3]. Choose this when dense initialization produces visible floaters near the camera [S3]. It is a variant of near clipping focused on artifact suppression rather than volume definition [S3].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the number of rays, due to tensor operations [S2, unofficial].
- Memory: $O(N)$ to store the updated near/far tensors [S2, unofficial].

**Measured Performance:**
- Metrics (PSNR, SSIM, LPIPS) are measured *without* clipping planes to ensure fair comparison, as clipping planes offer negligible improvements to quality metrics [S3].
- PSNR improvement for TreeHill at $d_{\text{near}}=1.0$ is negligible (26.91 → 26.93) [S3].
- PSNR improvement for Garden at $d_{\text{near}}=2.0$ is negligible (21.89 → 21.94) [S3].

**Strengths and Limitations:**
- **Strengths:** Simple to implement; effectively culls irrelevant geometry near the camera [S3].
- **Limitations:** Does not improve standard quality metrics; requires careful tuning of $d_{\text{near}}$ per scene [S3].

**When to Use:**
- Use during inference or rendering to hide floaters in dense Gaussian splatting or ray tracing [S3].
- Do not use for training evaluation if metric fairness is required [S3].

## Background
Originating from techniques to handle floaters in 3D Gaussian Splatting (3DGS) [S3], near clipping planes were adopted to mitigate visual artifacts caused by dense initialization [S3]. The method is implemented in toolkits like Gaussian-Splatting-Toolkit [S2, unofficial] and referenced in ray tracing optimizations like GRay [S3].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S2] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
[S3] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
