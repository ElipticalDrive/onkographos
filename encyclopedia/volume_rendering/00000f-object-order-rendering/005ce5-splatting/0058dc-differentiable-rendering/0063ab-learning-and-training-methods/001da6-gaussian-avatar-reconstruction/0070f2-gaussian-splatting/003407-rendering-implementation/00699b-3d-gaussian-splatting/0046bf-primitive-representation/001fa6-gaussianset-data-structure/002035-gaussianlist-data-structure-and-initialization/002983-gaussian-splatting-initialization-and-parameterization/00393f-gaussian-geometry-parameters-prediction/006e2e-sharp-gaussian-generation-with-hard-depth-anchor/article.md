# SHARP Gaussian Generation with Hard Depth Anchor

Also called: Hard-depth SHARP, SplatMend SHARP generation

## Overview
This method generates SHARP Gaussians for hole filling in 3D Gaussian Splatting by bypassing the learned Alignment UNet of the original SHARP pipeline [S1, unofficial]. It uses completed camera-Z depth as the primary anchor and SHARP's monocular relative gap as a secondary channel to maintain the five-channel decoder contract [S1, unofficial]. It differs from sibling methods like [GSGEN](8985-gsgen.md) by relying on explicit depth anchoring rather than learned generative priors for geometry initialization.

## How it works
**Inputs:** Input image, depth map, hole mask, camera JSON (intrinsics/extrinsics), SHARP model checkpoint, base Gaussian PLY.
**Outputs:** Generated Gaussian PLY with filled holes.

**Data Structures:**
*   `GaussianSet`: Fields include `mean_vectors` (B, C, H, W, D), `singular_values` (B, C, H, W, D), `quaternions` (B, C, H, W, D), `opacities` (B, C, H, W).
*   `CameraParams`: Intrinsics (focal length, principal point), Extrinsics (view matrix).

**Steps:**
1.  **Preparation:** Load SHARP predictor. Validate `gsplat` CUDA backend [S1, unofficial]. Verify camera is not orthographic and no per-Gaussian transforms are non-zero [S1, unofficial].
2.  **Flattening:** Permute `mean_vectors`, `singular_values`, `quaternions` from shape `(B, C, H, W, D)` to `(B, H*W, D)` and flatten `opacities` to `(B, H*W)` [S1, unofficial].
3.  **Depth Channel Construction:**
    *   Channel 1: Completed camera-Z depth.
    *   Channel 2: SHARP's monocular relative gap [S1, unofficial].
    *   Channels 3-5: Remaining SHARP input channels (not specified in sources) [S1, unofficial].
4.  **Quantile Computation:** Calculate tensor quantiles $p_{01}, p_{10}, \text{median}, p_{90}, p_{99}$ from flattened tensor values [S1, unofficial].
5.  **Tangent Estimation:**
    *   Compute forward/backward edges via finite-difference surface along axis [S1, unofficial].
    *   Compute norms and cosine similarity.
    *   Consistency check: Cosine similarity $> 0.25$ and norm ratio $< 3.0$ [S1, unofficial].
    *   Tangent vector: Average of forward/backward if consistent, else shorter vector [S1, unofficial].
    *   Clamp norm to $[0.5 \times \text{baseline}, \text{max\_stretch} \times \text{baseline}]$ [S1, unofficial].
6.  **Covariance Calculation:** Compute surface Jacobian covariance using surface points, layer size, focal length, image dimensions, tangent sigma, normal ratios, and max stretch [S1, unofficial].
7.  **Generation:** Pass inputs through SHARP decoder to generate Gaussian parameters [S1, unofficial].
8.  **Post-processing:** Crop and blend six-ring patch, append to base PLY, validate [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_sharp_gaussians(image, depth, hole_mask, camera_json, checkpoint):
    # 1. Validation
    if not gsplat_cuda_backend_loaded():
        raise RuntimeError("gsplat CUDA backend not loaded")
    if camera_json.is_orthographic:
        raise RuntimeError("Orthographic cameras not supported")
    if any(transform != 0 for transform in base_gaussians.transforms):
        raise RuntimeError("Per-Gaussian transforms not supported")

    # 2. Flatten Gaussians
    mean_flat = permute_and_flatten(base_gaussians.mean_vectors, (B, C, H, W, D), (B, H*W, D))
    sing_flat = permute_and_flatten(base_gaussians.singular_values, (B, C, H, W, D), (B, H*W, D))
    quat_flat = permute_and_flatten(base_gaussians.quaternions, (B, C, H, W, D), (B, H*W, D))
    opac_flat = flatten(base_gaussians.opacities, (B, C, H, W), (B, H*W))

    # 3. Depth Channels
    # Channel 1: Completed camera-Z depth
    depth_ch1 = complete_camera_z(depth, hole_mask)
    # Channel 2: SHARP monocular relative gap
    depth_ch2 = compute_sharp_relative_gap(image)
    # Channels 3-5: Placeholder for unspecified SHARP inputs
    depth_ch3 = get_sharp_channel_3(image) # Not given in the sources
    depth_ch4 = get_sharp_channel_4(image) # Not given in the sources
    depth_ch5 = get_sharp_channel_5(image) # Not given in the sources

    # 4. Quantiles
    p01 = quantile(mean_flat, 0.01)
    p10 = quantile(mean_flat, 0.10)
    median = quantile(mean_flat, 0.50)
    p90 = quantile(mean_flat, 0.90)
    p99 = quantile(mean_flat, 0.99)

    # 5. Tangent Estimation
    forward_edges = finite_difference_forward(depth_ch1)
    backward_edges = finite_difference_backward(depth_ch1)
    forward_norm = norm(forward_edges)
    backward_norm = norm(backward_edges)
    cosine_sim = cosine_similarity(forward_edges, backward_edges)
    norm_ratio = forward_norm / backward_norm

    consistent = (cosine_sim > 0.25) and (norm_ratio < 3.0)
    if consistent:
        tangent = (forward_edges + backward_edges) / 2.0
    else:
        tangent = select_shorter(forward_edges, backward_edges)

    baseline = compute_baseline(camera_json)
    max_stretch = 2.0 # Not given in the sources, typical value assumed for clamp
    tangent_norm = clamp(norm(tangent), 0.5 * baseline, max_stretch * baseline)
    tangent = normalize(tangent) * tangent_norm

    # 6. Covariance
    covariance = compute_surface_jacobian_covariance(
        surface_points=depth_ch1,
        layer_size=1.0, # Not given in the sources
        focal_length=camera_json.focal_length,
        image_dims=image.shape,
        tangent_sigma=tangent_norm,
        normal_ratios=compute_normal_ratios(tangent),
        max_stretch=max_stretch
    )

    # 7. SHARP Decoder
    # Inputs: [depth_ch1, depth_ch2, depth_ch3, depth_ch4, depth_ch5]
    gaussians = sharp_decoder.decode([depth_ch1, depth_ch2, depth_ch3, depth_ch4, depth_ch5])

    # 8. Post-processing
    patch = crop_six_ring_patch(gaussians, hole_mask)
    final_ply = append_and_blend(base_ply, patch)
    return final_ply
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Validated on Windows, RTX 5070 Ti (`sm_120`), CUDA Toolkit 12.8 [S1, unofficial].
*   **Strengths:** Bypasses learned Alignment UNet, reducing reliance on complex alignment networks [S1, unofficial].
*   **Limitations:** Requires specific CUDA backend (`gsplat`); does not support orthographic cameras or per-Gaussian transforms [S1, unofficial].
*   **When to use:** Hole filling in 3DGS where explicit depth anchoring is preferred over learned alignment [S1, unofficial].
*   **When not to use:** Orthographic camera setups or when per-Gaussian transforms are required [S1, unofficial].

## Background
Originated from the SplatMend project by zjllllllllll [S1, unofficial]. It adapts Apple's SHARP model for 3DGS hole filling by modifying the input pipeline to use hard depth anchors instead of learned alignment [S1, unofficial]. The method is part of a six-step depth-anchored pipeline for repairing 3DGS scenes [S1, unofficial].

## Sources
[S1] zjllllllllll/SplatMend, https://github.com/zjllllllllll/SplatMend, fetched 2026-10-27 (unofficial)
